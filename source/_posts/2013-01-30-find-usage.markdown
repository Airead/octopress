---
layout: post
title: "find usage"
date: 2013-01-30 16:43
comments: true
published: false
categories: linux-tools
---

## How to set chmod for a folder and all of its subfolders except files?
find . -type d -exec chmod 755 {} \;
