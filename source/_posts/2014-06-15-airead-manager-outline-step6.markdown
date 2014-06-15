---
layout: post
title: "airead-manager-outline-step6"
date: 2014-06-15 18:49
comments: true
categories: airead-manager
---

前端与后端交互
<!-- more -->

## 修改 world.coffee
为 `world.coffee` 增加增删改查函数，如下
{% include_code lang:py airead-manager/step6/world.coffee %}

## 修改模版
让模版使用 `world.coffee` 中新加的函数，修改如下:
{% include_code airead-manager/step6/world.html %}

## 操作验证
- 点击查询，可以查找数据库中已经存在的记录，在添加记录之前是没有记录的
- 输入人名点击添加，可以添加一个人的记录
- 在表中输入人名，就可以修改该人的名字

`git checkout step6` 查看源代码
