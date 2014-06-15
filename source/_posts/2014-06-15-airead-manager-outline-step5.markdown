---
layout: post
title: "airead-manager-outline-step5"
date: 2014-06-15 18:01
comments: true
categories: airead-manager
---

添加增删改查 restful 接口
<!-- more -->

## 添加 rest 接口
在 `blueprints/rest` 中创建文件 `persons.py`, 注意这里有`s`
{% include_code airead-manager/persons.py %}

## 测试 rest 接口
为了验证 persons rest 接口的可用性，下面使用 nose 结何 flask-testing 进行测试

在 `test` 中创建文件 `test_persons.py`
{% include_code airead-manager/test_persons.py %}

运行 `nosetests test/test_persons.py -v`, 应该全部测试OK

添加增删改查 restful 接口完成

`git checkout step5` 查看源码