---
layout: post
title: "How programs get run"
description: "Notes on pair of LWN article with similar title"
categories: [linux]
tags: [linux, kernel]
redirect_from:
  - /2020/11/01/
---


# EXECVE() system call

Links to original articles, [first](https://lwn.net/Articles/630727/) and [second](https://lwn.net/Articles/631631/)


### Setup

Prototype of the `execve()` looks like following

```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```

This passes both `argv` and `envp` to the new binary which is about to get executed. This function can execute a binary or interprated script. To demostrate the the internals we have following programs setup

**do_execve.c** is a wrapper which takes any executable as an argument and pass it to `execve()` with sample `args` and `envp`

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    char *args[] = {"zero", "one", "two", NULL};
    char *envp[] = {"ENVVAR1=1", "ENVVAR2=2", NULL};
    execve(argv[1], args, envp);
    /* won't reach here if argv[1] can be executed */
    fprintf(stderr, "Failed to execute '%s', %s\n", argv[1], strerror(errno));
    return 1;
}
```



**show_info.c** is a sample executable binary called by above wrapper to display `args` and `envp` which it got from the wrapper executable itself. 

```c
#include <stdio.h>

extern char **environ;

int main(int argc, char *argv[])
{
    int ii;
    char **p = environ;
    for (ii = 0; ii < argc; ii++)
        printf("argv[%d] = '%s'\n", ii, argv[ii]);
    while (*p)
        printf("%s\n", *p++);
    return 0;
}
```



**show_info.sh** is sample shell script which does the same as above binary executable.

```c
#!/bin/sh
echo "\$0 = '$0'"
ii=1
for arg in "$@"; do
    echo "\$$ii = '$arg'"
    ii=`expr $ii + 1`
done
env
```



### Execution

Lets cover execution of the binary and the script with the wrapper  __do_execve__

- Execution of the binary

  ```shell
  % ./do_execve ./show_info
  argv[0] = 'zero'
  argv[1] = 'one'
  argv[2] = 'two'
  ENVVAR1=1
  ENVVAR2=2
  ```

- Execution of the script

  ```shell
  % ./do_execve ./show_info.sh
  $0 = './show_info.sh'
  $1 = 'one'
  $2 = 'two'
  ENVVAR1=1
  ENVVAR2=2
  PWD=/home/drysdale/src/lwn/exec
  ```

  There are two differences in script execution compared to binary execution

  - argument 0 got `zero` got replaced with `./show_info.sh` which is a name of a script.
  - `additional PWD` environment variable got added.

- Executing wrapper with another wrapper

  - Simple Wrapper 
   ```c
   #create a wrapper with following script
   
   % cat ./wrapper
   #!./show_info
   
   #Execute above wrapper
   
   % ./do_execve ./wrapper
   argv[0] = './show_info'
   argv[1] = './wrapper'
   argv[2] = 'one'
   argv[3] = 'two'
   ENVVAR1=1
   ENVVAR2=2
   ```
  - Wrapper with the arguments
   ```shell
   % cat ./wrapper_args
   #!./show_info -a -b -c
   
   % ./do_execve ./wrapper_args
   argv[0] = './show_info'
   argv[1] = '-a -b -c'
   argv[2] = './wrapper_args'
   argv[3] = 'one'
   argv[4] = 'two'
   ENVVAR1=1
   ENVVAR2=2
   ```
  - Nesting of wrappers
   ```c
   argv[0]:  'zero'=>'./wrapper4'=>'./wrapper3'=>'./wrapper2'=>'./wrapper' =>'./show_info'
   argv[1]:  'one'   './wrapper5'  './wrapper4'  './wrapper3'  './wrapper2'  './wrapper'
   argv[2]:  'two'   'one'         './wrapper5'  './wrapper4'  './wrapper3'  './wrapper2'
   argv[3]:          'two'         'one'         './wrapper5'  './wrapper4'  './wrapper3'
   argv[4]:                        'two'         'one'         './wrapper5'  './wrapper4'
   argv[5]:                                      'two'         'one'         './wrapper5'
   argv[6]:                                                    'two'         'one'
   argv[7]:                                                                  'two'
   
  #after 6th level of wrapper it fails with ELOOP error
   
   
   % ./do_execve ./wrapper6
   Failed to execute './wrapper6', Too many levels of symbolic links
   
   
   ```
  

One key observation is that `arg[0]` still getting replaced just once and newer wrappers are getting pushed in the `args` stack between `arg[0]` and first script argument till 6th wrapper.





### Under the hood

- `execve()` system call reaches to the ` do_execve_common()` defined in `fs/exec.c` which builds  `struct linux_binprm`. This structure has various fields which helps invocation operation of the executable.

- The `bprm_mm_init()` function allocates and sets up the associated `struct mm_struct` and `struct vm_area_struct` data structures in preparation for managing the virtual memory of the new program. In particular, the new program's virtual memory ends at the highest possible address for the architecture; its stack will grow downward from there.

- The `p` field is set to point at the end of memory space for the new program, but leaves space for a `NULL` pointer as an end marker for the stack, which then get appended by environment variables followed by command line arguments. Memory layout for binary execution looks like following 

  ```text
      ---------Memory limit - begining of the stack ---------
      NULL pointer
      program_filename string
      envp[envc-1] string
      ...
      envp[1] string
      envp[0] string
      argv[argc-1] string
      ...
      argv[1] string
      argv[0] string
  ```

   Which changes slighly for interpreted file execution where `arg[0]` is replaced by three entries.

  ```text
      ---------Memory limit---------
      NULL pointer
      program_filename string
      envp[envc-1] string
      ...
      envp[1] string
      envp[0] string
      argv[argc-1] string
      ...
      argv[1] string
      program_filename string
      ( interpreter_args )
      interpreter_filename string
  ```

- The `buf` scratch space is filled with the first chunk (128 butes) of data from the program file which allows us to detect binary format and call related handlers accordingly. These handlers can be `binfmt_elf.c` ,  `binfmt_script.c`  and many more. 



This above arrangement of variables later get addressed by a program



```
    ------------------------------------------------------------- 0x7fff6c845000
     0x7fff6c844ff8: 0x0000000000000000
            _  4fec: './stackdump\0'                      <------+
      env  /   4fe2: 'ENVVAR2=2\0'                               |    <----+
           \_  4fd8: 'ENVVAR1=1\0'                               |   <---+ |
           /   4fd4: 'two\0'                                     |       | |     <----+
     args |    4fd0: 'one\0'                                     |       | |    <---+ |
           \_  4fcb: 'zero\0'                                    |       | |   <--+ | |
               3020: random gap padded to 16B boundary           |       | |      | | |
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -|       | |      | | |
               3019: 'x86_64\0'                        <-+       |       | |      | | |
     auxv      3009: random data: ed99b6...2adcc7        | <-+   |       | |      | | |
     data      3000: zero padding to align stack         |   |   |       | |      | | |
    . . . . . . . . . . . . . . . . . . . . . . . . . . .|. .|. .|       | |      | | |
               2ff0: AT_NULL(0)=0                        |   |   |       | |      | | |
               2fe0: AT_PLATFORM(15)=0x7fff6c843019    --+   |   |       | |      | | |
               2fd0: AT_EXECFN(31)=0x7fff6c844fec      ------|---+       | |      | | |
               2fc0: AT_RANDOM(25)=0x7fff6c843009      ------+           | |      | | |
      ELF      2fb0: AT_SECURE(23)=0                                     | |      | | |
    auxiliary  2fa0: AT_EGID(14)=1000                                    | |      | | |
     vector:   2f90: AT_GID(13)=1000                                     | |      | | |
    (id,val)   2f80: AT_EUID(12)=1000                                    | |      | | |
      pairs    2f70: AT_UID(11)=1000                                     | |      | | |
               2f60: AT_ENTRY(9)=0x4010c0                                | |      | | |
               2f50: AT_FLAGS(8)=0                                       | |      | | |
               2f40: AT_BASE(7)=0x7ff6c1122000                           | |      | | |
               2f30: AT_PHNUM(5)=9                                       | |      | | |
               2f20: AT_PHENT(4)=56                                      | |      | | |
               2f10: AT_PHDR(3)=0x400040                                 | |      | | |
               2f00: AT_CLKTCK(17)=100                                   | |      | | |
               2ef0: AT_PAGESZ(6)=4096                                   | |      | | |
               2ee0: AT_HWCAP(16)=0xbfebfbff                             | |      | | |
               2ed0: AT_SYSINFO_EHDR(33)=0x7fff6c86b000                  | |      | | |
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .        | |      | | |
               2ec8: environ[2]=(nil)                                    | |      | | |
               2ec0: environ[1]=0x7fff6c844fe2         ------------------|-+      | | |
               2eb8: environ[0]=0x7fff6c844fd8         ------------------+        | | |
               2eb0: argv[3]=(nil)                                                | | |
               2ea8: argv[2]=0x7fff6c844fd4            ---------------------------|-|-+
               2ea0: argv[1]=0x7fff6c844fd0            ---------------------------|-+
               2e98: argv[0]=0x7fff6c844fcb            ---------------------------+
     0x7fff6c842e90: argc=3
```

