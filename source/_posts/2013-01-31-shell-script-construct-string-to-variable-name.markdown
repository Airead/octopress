---
layout: post
title: "shell script construct string to variable name"
date: 2013-01-31 11:30
comments: true
categories: shell
---

This post shows how to access shell variable via a name, which created by a string. Ok, let's see the code straight:

{% codeblock lang:sh %}
#!/bin/bash

a1=10
a2=20
a3=30

for i in `seq 3`
do
        name=a$i
        echo ${!name}
done
{% endcodeblock %}

The script will print:

<code>
10  
20  
30  
</code>

Note the `echo ${!vname}`. if it change to `echo ${name}` and will just print:

<code>
a1  
a2  
a3  
</code>

That's the difference. The **`!'** construct string to variable name.
