---
layout: post
title: Linux Kernel tracing and profiling notes
description: "Notes gathered from various blog posts on Linux kernel tracing and profiling"
categories: [linux]
tags: [linux, tracing, profiling]
Comments: true
---


## Table of Contents

* Kramdown table of contents
{:toc .toc}
## Objective

This place is the collective notes which I gathered while studying through various blog posts reagrding linux kernel analysis. Any issue with the linux system can go through following phases of troubleshooting

-  Who is causing the issue? It could be some Program or the User which can be easily find out through bunch of unix commands like `top`.
- What is issue? This is the effect of the undesired activity on various subsystems like CPU,Memory, Network  etc.  Preliminary infromation can be gathred through various commands `sar` and `vmstat`. For more advance usage, you can use `perf stat -a -d`.
- Why is this issue? For this stage you require profiling and tracing where you can explore effect of parts of the program on each subsystem. 
- How this issue got manifested over the period of time? This requires time trend analysis.

We will see how profiling and tracing helps to execute above phases. To make any of those steps to work, we have to remove kernel symbols restriction with following command 

```shell
$ echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
```



## Profiling and Tracing

What is profiling? It is a sampling of the events. It literly interrupts the program execution in the steps of very short intervals. The purpose of each interruption is to take a sample, which is done by visiting each running thread, and then examining the stack to discover which functions are running. 



The profiler records each sample and all of the samples are aggregated into a report or turned into a graph like `flame graphs`.



On the other hand, the trace is a log of events within to the program. This log is detailed enough to report functions calls and returns with various timing information. This requires lot of instrumentation in the kernel code. As trace is a sequence of events logged which can be seen as profiling of those events. We can generate profiling out of traced data, in other words, trace is a fine grained version of profiling.



There are two types of tracing, one is Static tracing which gets the log of events emmited from predefined locations in the kernel with the `TRACE_EVENT` macro. Dynamic tracing is the mechanism you can insert a inspection code into kernel address space which can allow you to analyze specific function and its parameters. 

Kprobes are implmented using `do_init3` calling `kprobe_handler` which is similar to how gdb implementes step by step execution. Kernel probe adds around `0.86us` of overhead.





## Perf profiling and tracing
In linux `perf` utility can be used to do sampling with following commands for the entire system. You can do sample for particualr event with `-e` and/or particualr process with `-p`. Here `-F 99` represents sampling frequency which is 99 time per second aka 99 Hz.

```shell
# counting hardware events
## entire system
$ sudo perf stat -C 10   -- sleep 5

## specific process
$ sudo perf stat -ddd ./noploop


# profiling entire system 
$ sudo perf record -F 99 -a -g -- sleep 1

# output generated from perf.data
$ sudo perf annotate --stdio
```

Above commnds generate `perf.data` file which later can be used to generate flamegraph. Here is typical workflow for `perf` based profiling 

![perf_action_workflow](https://github.com/valmikroy/learnings.github.io/blob/master/_posts/images/2020-10-30-Kernel-Tracing/perf_action_workflow.jpg?raw=true)

`perf` is implemented with `perf_event` system call which collects sample on the userspace ring buffer for above userspace processing. It is just frontend through which you can access various tracepoints and counters for hardware and software events. 

Here is a diagram 

![linux-tracing-perf_event.png](https://github.com/valmikroy/learnings.github.io/blob/master/_posts/images/2020-10-30-Kernel-Tracing/linux-tracing-perf_event.png?raw=true) 



`perf` can be used for static tracing where you can count occurance of hitting defined `TRACE_EVENTS` points in the kernel.

```shell
# list all tracepoints 
$ sudo perf list 2>&1 | grep Tracepoint

# list processes which are hitting trace point
$ sudo perf trace -e   kmem:kmalloc

# number of counts 
$ sudo perf stat -a -e   kmem:kmalloc -I 1000

```

`perf probe` can let you insert dynamic probes into the kernel.

```shell

# probe Add
$ sudo perf probe --add clear_huge_page

# probe Use
$ sudo perf stat -a -e probe:clear_huge_page -I 10000

# probe Delete
$ sudo perf probe --del=probe:clear_huge_page


$ sudo perf probe -V tcp_sendmsg
Available variables at tcp_sendmsg
        @<tcp_sendmsg+0>
                int     ret
                size_t  size
                struct msghdr*  msg
                struct sock*    sk

# probe add for a size param value above  
$ sudo perf probe --add 'tcp_sendmsg size’

# show you the process and the call to tcp_sendmsg
$ sudo perf trace  -e probe:tcp_sendmsg -a  

# probe delete
$ sudo perf probe --del probe:tcp_sendmsg


#You can also look at the place this symbol has exported 
$ sudo perf probe -L tcp_sendmsg
<tcp_sendmsg@/build/linux-aws-9iwObr/linux-aws-5.4.0/net/ipv4/tcp.c:0>
      0  int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
      1  {
      2         int ret;


      4         lock_sock(sk);
      5         ret = tcp_sendmsg_locked(sk, msg, size);
      6         release_sock(sk);


      8         return ret;
         }
         EXPORT_SYMBOL(tcp_sendmsg);


# probing systemcalls

## open()
$ sudo perf probe --add 'do_sys_open filename:string'
$ sudo perf record --no-buffering -e probe:do_sys_open -o - -a | PAGER=cat perf script -i -
$ sudo perf probe --del do_sys_open

## exec()
$ sudo perf probe --add 'do_execve +0(+0(%si)):string +0(+8(%si)):string +0(+16(%si)):string +0(+24(%si)):string'
$ sudo perf record --no-buffering -e probe:do_execve -a -o - | PAGER="cat -v" stdbuf -oL perf script -i -
$ sudo perf probe --del do_execve

```



perf UserSpace probing examples 

```shell
# Add user probe
$ sudo perf probe --exec=/lib/x86_64-linux-gnu/libc.so.6 --add malloc

# print count of malloc calls
$ sudo perf stat -a -e probe_libc:malloc -I 10000

# trace system wide malloc 
$ sudo perf record -e probe_libc:malloc -a
$ sudo perf report -n

# Delete user probe
$ sudo perf probe --exec=/lib/x86_64-linux-gnu/libc.so.6  --del malloc

```



## Ftrace

Ftrace is another framework which gives you access to the various `TRACE_EVENT`s. It also allows you inject probing on the userspace functions and allows you to create flow of function execution with `function_graph` tracer functionality. `trace-cmd` is the frontend for the ftrace.

In general `ftrace` works similar to about `perf_event` implementation with some differences like

- It has ring buffer in kernel space per CPU

- It can do call graphs and stack tracing 

- It has other tracers which can be used for MMIO or Block IO tracing 

- It allows you to install user probes

  ```shell
  ~/sys/kernel/debug/tracing#: cat available_tracers
  blk mmiotrace function_graph wakeup_rt wakeup function nop
  ```

  

![linux-tracing-ftrace.png](https://github.com/valmikroy/learnings.github.io/blob/master/_posts/images/2020-10-30-Kernel-Tracing/linux-tracing-ftrace.png?raw=true) 



Here are some useful commands for `trace-cmd`

```shell
# get a list of tracers which fetches list from  /sys/kernel/debug/tracing/available_events 
$ sudo trace-cmd list  | grep Tracepoint

# run
$ sudo trace-cmd record -p function -e sched_switch ls > /dev/null

# what interrupt has highest latency 
$ sudo trace-cmd record -p function_graph -e irq_handler_entry -
l do_IRQ sleep 10; trace-cmd report

# kmalloc calls greater than 1000 bytes
$ sudo trace-cmd report -F ’kmalloc: bytes_req > 1000’

# get a function graph of ls command
$ sudo trace-cmd record -p function_graph ls /bin
```

You can also do user space probing with [uprobe_tracer](https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt) with following narrative 

```shell
Following example shows how to dump the instruction pointer and %ax register
at the probed text address. Probe zfree function in /bin/zsh:

    # cd /sys/kernel/debug/tracing/
    # cat /proc/`pgrep zsh`/maps | grep /bin/zsh | grep r-xp
    00400000-0048a000 r-xp 00000000 08:03 130904 /bin/zsh
    # objdump -T /bin/zsh | grep -w zfree
    0000000000446420 g    DF .text  0000000000000012  Base        zfree

  0x46420 is the offset of zfree in object /bin/zsh that is loaded at
  0x00400000. Hence the command to uprobe would be:

    # echo 'p:zfree_entry /bin/zsh:0x46420 %ip %ax' > uprobe_events

  And the same for the uretprobe would be:

    # echo 'r:zfree_exit /bin/zsh:0x46420 %ip %ax' >> uprobe_events
```





## eBPF



More programatic extention to access all the trace and probing hooks

![linux-tracing-bpf-for-tracing.png](https://github.com/valmikroy/learnings.github.io/blob/master/_posts/images/2020-10-30-Kernel-Tracing/linux-tracing-bpf-for-tracing.png?raw=true)



Here’s how it works

- write an eBPF program
- attach that probe to a kprobe/uprobe/tracepoint/USDT-probe inside kernel
- program writes out data to an eBPF map / ftrace / perf buffer to get collected for analysis



## USDT and uprobe
This section will take a look at `uprobe` with `bpftrace` which allows us to probe through existing binaries on the fly. Later we will see Userspace Statically Defined Tracepoint (USDT) which allows us to add custom probes in the application code which can then probed by `bpftrace` or `systemtap`.


### uprobe
This is a dynamic probing of any binary available. There is a details document [here](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#3-uprobeuretprobe-dynamic-tracing-user-level)  but brief description with following example

- Every program can provide you location of each function in its object dump. For example `readline` function used by `bash` is located at certain place.

  ```shell
  $ objdump -tT /bin/bash | grep " readline"
  00000000000b7cd0 g    DF .text  0000000000000252  Base        readline_internal_char
  00000000000b71a0 g    DF .text  000000000000015f  Base        readline_internal_setup
  00000000000b8530 g    DF .text  000000000000009a  Base        readline
  00000000000b7300 g    DF .text  0000000000000120  Base        readline_internal_teardown
  ```

- The availble addresses can be used to attach a uprobe to this function with following commands

  ```shell
  bpftrace -e 'uprobe:/bin/bash:readline { printf("read a line\n"); }'
  
  bpftrace -e 'uretprobe:/bin/bash:readline { printf("read a line\n"); }'
  ```

- You can also fetch the arguments of the function, though this part is CPU architecture dependent.

  ```shell
  bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc-2.23.so:fopen { printf("fopen: %s\n", str(arg0)); }'
  Attaching 1 probe...
  fopen: /proc/filesystems
  fopen: /usr/share/locale/locale.alias
  fopen: /proc/self/mountinfo
  ^C
  
  bpftrace -e 'uretprobe:/bin/bash:readline { printf("readline: \"%s\"\n", str(retval)); }'
  Attaching 1 probe...
  readline: "echo hi"
  readline: "ls -l"
  readline: "date"
  readline: "uname -r"
  ^C
  ```

- You can also go by offset to pinpoint the opcodes. 

Syntax

```
uprobe:library_name:function_name[+offset]
uprobe:library_name:address
uretprobe:library_name:function_name
```



### USDT 

USDT allows you to create custom probe points for your userspace application.

Here is an example C program 

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/sdt.h>

void rdom(int seed, int *a, int *b);

int main(){
    int a, b, r;


    while (1) {
        r = rand();
        rdom(r,&a,&b);
        printf("tick\n");
        DTRACE_PROBE2(tick, probe1,a,b );
        sleep(5);
    }

        return 0;
}


void rdom(int seed, int *a, int *b) {
    //printf("seed - %d\n",seed);
    DTRACE_PROBE1(tick, probe2,seed);
    *a = seed % 8;
    *b = seed % 5;
}
```

 Above program requires systemtap development libraries and you can compile it with `gcc tick.c  -o tick`

Once compiled, you can see probe related note in the `elf` binary structure 

```
$ readelf -a   tick

<snip>

Displaying notes found in: .note.stapsdt
  Owner                Data size        Description
  stapsdt              0x00000034       NT_STAPSDT (SystemTap probe descriptors)
    Provider: tick
    Name: probe1
    Location: 0x00000000000011d3, Base: 0x0000000000002009, Semaphore: 0x0000000000000000
    Arguments: -4@%eax -4@%edx
  stapsdt              0x00000030       NT_STAPSDT (SystemTap probe descriptors)
    Provider: tick
    Name: probe2
    Location: 0x00000000000011f3, Base: 0x0000000000002009, Semaphore: 0x0000000000000000
    Arguments: -4@-4(%rbp)
```

You can run a tick binary and then access those probles 

```
# list 
$  sudo bpftrace -l 'usdt:./tick'
usdt:./tick:tick:probe1
usdt:./tick:tick:probe2

# peek
$ sudo bpftrace -e 'usdt:./tick:tick:probe2 { printf("here  - %d \n", arg0 );  }'
Attaching 1 probe...
here  - 1681692777
here  - 1714636915
^C

$ sudo bpftrace -e 'usdt:./tick:tick:probe1 { printf("here  - %d - %d\n", arg0, arg1 );  }'
Attaching 1 probe...
here  - 3 - 4
here  - 3 - 3
^C
```

These probes can be protected by semaphore, that means the structures related to probing will only get initialized if semaphore is set. For that you have to use little more elaborated mechanism to generate header file and compile the program. Here is quick example from  [leezhenghui's](https://github.com/leezhenghui) [USDT blog](https://leezhenghui.github.io/linux/2019/03/05/exploring-usdt-on-linux.html)

- dtrace file  tick-dtrace.d

```
provider tick {
        probe loop1(int);
        probe loop2(int);
}
```

- Tick program 

  ```c
  #include <stdio.h>
  #include <unistd.h>
  #include "tick-dtrace.h"
  
  int
  main(int argc, char *argv[])
  {
          int i;
  
          while(1) {
                  i++;
                  DTRACE_PROBE1(tick, loop1, i);
                  if (TICK_LOOP2_ENABLED()) {
                          DTRACE_PROBE1(tick, loop2, i);
                  }
                  printf("tick: %d\n", i);
                  sleep(5);
          }
  
          return (0);
  }
  ```

- generate header file `tick-dtrace.h` and compile 
```shell
dtrace -G -s tick-dtrace.d -o tick-dtrace.o
dtrace -h -s tick-dtrace.d -o tick-dtrace.h
gcc -c tick-main.c
gcc -o tick tick-main.o tick-dtrace.o

readelf -n ./tick
```












## Credits

- Thanks to [leezhenghui](https://github.com/leezhenghui) for his blog on [USDT](https://leezhenghui.github.io/linux/2019/03/05/exploring-usdt-on-linux.html), and I also copied formatting of his github pages.
- Thanks to Brendan Gregg for his [SCALE13x talks](http://www.brendangregg.com/blog/2015-02-27/linux-profiling-at-netflix.html), I picked up few details from there.
- [Profiling versus tracing](https://www.jwhitham.org/2016/02/profiling-versus-tracing.html) by Jack Whitham.
- How [Kprobe works](https://lwn.net/Articles/132196/)? 
- [Various examples](http://www.jauu.net/talks/data/runtime-analyse-of-linux-network-stack.pdf) from all probing tools
- [eBPF customization examples](https://www.collabora.com/news-and-blog/blog/2019/04/15/an-ebpf-overview-part-2-machine-and-bytecode/)




