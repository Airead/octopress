---
layout: post
title: "Use Egypt To Create Call Graphs"
date: 2012-12-04 15:36
comments: true
categories: linux-tools graphviz
---

Call graphs could give a big picture of some project which makes you better and faster understanding of a project.
[Egypt](http://www.gson.org/egypt/) is a simple tool for creating call graphs of C programs. Egypt neither analyzes source code nor lays out graphs. Instead, it leaves the source code analysis to GCC and the graph layout to Graphviz, both of which are better at their respective jobs than egypt itself could ever hope to be. Egypt is simply a very small Perl script that glues these existing tools together.  
Run egypt, You will also need gcc, Perl, and [Graphviz](http://www.graphviz.org/).

<!--more-->