---
layout: post
title: "airead-manager-outline-step3"
date: 2014-06-15 16:39
comments: true
categories: airead-manager
---

前端显示后端内容
<!-- more -->

## 修改 controller
修改 `static/coffee/controllers/world.coffee` 为
{% include_code lang:py airead-manager/step3/world.coffee %}

## 修改模版
修改 `static/templates/world.html` 为
{% include_code airead-manager/step3/world.html %}

刷新页面，应该会看到 `hello example!` 内容，这是服务器返回的

`git checkout step3` 查看源码