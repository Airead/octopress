---
layout: post
title: "r lang: get data from data.frame by one level"
date: 2013-03-09 10:51
comments: true
categories: R
---

## Prepare Data
<!--more-->
`$ x <- 1:20`  
`$ y <- c("A", "B", "C", "D")`  
`$ fr <- data.frame(x, y)`  

{% codeblock %}
result:
    x y  
1   1 A  
2   2 B  
3   3 C  
4   4 D  
5   5 A  
6   6 B  
7   7 C  
8   8 D  
9   9 A  
10 10 B  
11 11 C  
12 12 D  
13 13 A  
14 14 B  
15 15 C  
16 16 D  
17 17 A  
18 18 B  
19 19 C  
20 20 D
{% endcodeblock %}

## get one level data
`$ fr[fr$y == "A", ]`
{% codeblock %}
result:  
    x y  
1   1 A  
5   5 A  
9   9 A  
13 13 A  
17 17 A  
{% endcodeblock %}

`$ fr[fr$y == "B", ]`
{% codeblock %}
result:
    x y  
2   2 B  
6   6 B  
10 10 B  
14 14 B  
18 18 B  
{% endcodeblock %}
