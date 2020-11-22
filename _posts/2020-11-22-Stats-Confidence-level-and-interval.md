---
layout: post
title: Understanding Confidence level and Confidence interval
description: "Understanding Confidence level and Confidence interval while using netperf"
categories: [statistic]
tags: [statisti, math]
Comments: true

---







# Understanding Confidence level and Confidence interval

`netperf` utility allows us to do linux network tcp connection benchmark. It allows you to measure state of connectivity using statistical method of **confidence internval and level**. Here are notes which first to understand those statistical terms refering to this [blog entry](https://sanjayjsw05.medium.com/confidence-interval-and-confidence-level-layman-explanation-572a20eaaf2d). 

## Table of Contents

* Kramdown table of contents
{:toc .toc}




### Defination of Confidence level

Confidence level refers to the percentage of all possible samples that can be expected to include the true population parameter (like mean, median etc) .



### Statistical Example

In above block entry, author wanted to find median score for population of 3K student. Doing this with each student is time consuming so we decided to sample the population. This is how sampling works

- Take a sample of 15 scores and take their mean, median and standard deviation.
- Take mutiple samples of the same size. Sample size 15 is represented by `n`
- And count of samples represented by `K`. Over here `K` is 20.
- Now decide on range of mean which could be guessed from available samples between `x` and `y`  This range is the **confidence interval**.
- And see how many samples has a mean which falls into range from `x` to `y`. 
- Number of samples with a mean in that range vs total sample count should provide you probability number. That number is the **confidence level**. 



### Netperf

 `netperf` allows you to define confidence level and confidence interval with `-I` parameter. For example following netperf command 

```
netperf -l -1000 -H <IP> -t TCP_RR -i 30,10 -I 95,30 -- -D -r 1,1 -s 0 -S 0 -O min_latency,mean_latency,max_latency,stddev_latency,p90_latency,p99_latency,transaction_rate 
```

 Where 

- `-I 95,30` means our 95% test samples mean falls into band of +/-15% of the average sampled mean.
- `-i 30,10` allows you to take maximum 30 samples to minimum 10 samples to meet above criteria.    



