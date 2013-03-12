---
layout: post
title: "add license in your octopress blog"
date: 2013-03-12 10:11
comments: true
categories: octopress
---

It's very easy to add license in your octopress blog. just two step:

<!--more-->
## Step One: Get File
download the following file [license](/download/code/octopress/license.html) to **octopress/source/_includes/custom/asides/**.
{% include_code octopress/license.html %}

## Step Two: Set _config.yml
add `custom/asides/license.html` in `default_asides`, as follows:  
<code>
default\_asides: [custom/asides/about.html, custom/asides/weibo.html, asides/recent\_posts.html, custom/asides/category\_list.html, **custom/asides/license.html**, ]
</code>

Now you have put the license in your blog.
