---
layout: post
title: Euler's number 
description: "Understanding exponetial decay function and its usage"
categories: [statistic]
tags: [statisti, math]
Comments: true


---



Euler's number is a magical entitiy in the mathematics.

```
2.7182818284590452353602874713527 (and more ...)
```



### Interest calculation example

Lets start with very hypothetical example, I have a credit card debt of $100 and they charge me 100% interest rate annually. I am being naive will think that 100% interest rate will only add up \$100 more dollars at the end of the year. So I have to pay just \$200 in total to credit card company. This is according to following simple interest calculation  
$$
Final Amount = Principal Amount * (1 + r)^y \\
 \  \\
\begin{align*} \\
r = annual\ interest\ rate\ in\ decimals \\
y = number\ of\ years  \\
\end{align*}
$$


Where interest rate put in the decimal points.
$$ 100 * (1 + 100/100)^1 = 100 * (1+1)^1 = 100 * 2 = 200  $$





But I missed the fine print saying that it will get compounded daily. What the hell is that? That means it is compunding interest rate. In our example, 

- 100% interest will get extrapolated for a day  
- Then interest for each day will get calculated
- That interest for the day will get added into principal amount 
- New principal will be used for next days calculation

$$
100\%/365 ~= 0.27\% \\
Day 1 \\ 
$100 * (1 + 0.0027) = $100.27 \\
Day 2 \\
$100.27 *  (1 + 0.0027) = $100.54 \\
Day 365 \\
$270.71 * (1 + 0.0027) = $271.45 \\
$$

Now it is way more than \$200 which I calculated initially with the simple interest.

Formula for compounding interest which we can use 
$$
Final Amount = Principal Amount * \left[1 + \frac{r/100}{n}\right]^n \\
Effective Annual Rate = \left[1+\frac{r/100}{n}\right]^n - 1 \\
 \  \\
\begin{align*} \\
r = annual\ interest\ rate\ in\ percentage \\
n = compounding\ internval \\
\end{align*}
$$
In our example effective annual rate is `171%` if compounded daily. 
$$
(1 + 1/365)^{365} - 1 = 2.714567482 - 1 = 1.714567482 ~= 171.4567482\%
$$
But look at that number `2.714567` which is closer to Euler's number above. In nutshell, it can be represented as following
$$
\lim_{n\to\infty} \left[1+\frac{r/100}n\right]^n = e^{r/100}
$$
So effective annual rate calcualtion can be done 
$$
Effective\ Annual\ Rate\ (\%) = ( e^{r/100} - 1 ) * 100  \\
\ \\
r = annual\ rate\ in\ percentage
$$







### Poisson Process 

The Poisson process is one of the most widely-used counting processes. It is usually used in scenarios where we are counting the occurrences of certain events that appear to happen at a certain rate, but completely at random (without a certain structure).  

Above example of compounding is exponential growth function while Poission process has an exponential decay function like following which reaches to 1 along with increase in value of `t`.



![Exponential_decay](images/2020-11-26-Eulers-number/Exponential_decay.jpg)



This is similar to above function converts into 
$$
-\frac1{rate} log(rand(0..1))
$$
 



```ruby
#!/usr/bin/env ruby
#


rate = 1000000
e = 2.7182

sum = 0

rate.times do |a|

 interval = Math.log(rand(),e)/rate * -1

 # over here we can add a multiplication factor 
 sum += interval

end

# this sum is always close to 1
puts sum
```



#### Referece

- [Basic Concepts of the Poisson Process](https://www.probabilitycourse.com/chapter11/11_1_2_basic_concepts_of_the_poisson_process.php)


















