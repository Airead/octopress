---
layout: post
title: "Use Egypt To Create Call Graphs"
date: 2012-12-04 15:36
comments: true
categories: linux-tools graphviz
---

Call graphs could give a big picture which makes you better and faster understanding of one project.  
[Egypt](http://www.gson.org/egypt/) is a simple tool for creating call graphs of C programs. Egypt neither analyzes source code nor lays out graphs. Instead, it leaves the source code analysis to GCC and the graph layout to Graphviz, both of which are better at their respective jobs than egypt itself could ever hope to be. Egypt is simply a very small Perl script that glues these existing tools together.  

<!--more-->
## Install Egypt
The most recent release of egypt is version 1.10, which can be downloaded [here](http://www.gson.org/egypt/download/egypt-1.10.tar.gz). Online documentation is [here](http://www.gson.org/egypt/egypt.html). You will also need gcc, Perl, and [Graphviz](http://www.graphviz.org/).

To install, extract the compressed tar file, cd to the egypt directory, and type  
<code>
perl Makefile.PL  
make  
make install  
</code>  

## Generate Call Graph
First of all, one project is necessary. you can [DOWNLOAD](/downloads/code/graphviz/summary-1.0.1.tar.gz) this project or select other one you prefered.  
cd to summary-1.0.1, and type  
<code>
./configure  
make CFLAGS=-fdump-rtl-expand  
</code>  
Each .c in project will generate a corresponding .expand. Egypt uses these .expand files to generate the .dot file, and graphviz use .dot file to generate call graph. For example,  
<code>
./src/person/main.c.144r.expand  
./src/person/do.c.144r.expand  
./src/lib/lib.c.144r.expand  
</code>  
will be generated in summary project. 

Now type
{% codeblock %}
egypt ./src/person/main.c.144r.expand ./src/person/do.c.144r.expand ./src/lib/lib.c.144r.expand | dot -Grankdir=LR -Tsvg -o summary.svg
{% endcodeblock  %}

then open summary.svg which is shown below.  
{% img center /images/graphviz/summary.svg %}  
Note that, the dash line represents function pointer (C language) calling, the picture makes you better and faster understanding of this project. The main() directly call analyze_person(), and call do\_\*() by calling function pointer. The do\_\*() directly call lib\_print().  

You will happy to use egypt and graphviz to study a unfamiliar project. Enjoy it!

## References
 - [egypt index](http://www.gson.org/egypt/)
 - [Go to Egypt for Call Graph](http://openbrd.com/?p=35)
