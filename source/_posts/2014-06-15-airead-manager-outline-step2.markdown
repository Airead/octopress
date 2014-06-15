---
layout: post
title: "airead-manager-outline-step2"
date: 2014-06-15 15:30
comments: true
categories: airead-manager
---

添加一个前端接口
<!-- more -->

## 创建一个 controller
在 `static/coffee/controllers/` 创建文件 `world.coffee`.
{% include_code lang:py airead-manager/world.coffee %}
刷新页面，会发现 `学习` 没有出现。这是因为 `static/coffee/controllers/main.coffee` 中

    1. 该文件没有导入 `world.coffee`;
    2. 菜单组(`navHeaderList`)中没有中定义 `学习` 组;

## 添加一个菜单组
### 1. 导入 world.coffee
将 `./world` 添加到 `./test_perm` 的下面。

```
define [
  './base'
  './test'
  './home'
  './userManager'
  './groupManager'
  './test_perm'
  './world'  # 添加这一行，导入 world 菜单
], (indexCtlModule) ->
  moduleName = 'main'
```

### 2. 添加 学习 组
```
$scope.navHeaderList = ['系统管理', '权限管理', '学习', 'hide']  # 学习
```

这时刷新页面，应该能看到 `item Hello` 菜单了。但是点击 `item Hello` 却什么也没显示，这是因为

    没有创建模块文件

## 添加模版文件
在 `static/templates` 中创建文件 `world.html`
{% include_code airead-manager/world.html %}


再刷新页面，点击 `item Hello` 菜单，应该能看到，`world.html` 里的内容了。

`git checkout step2` 查看源代码