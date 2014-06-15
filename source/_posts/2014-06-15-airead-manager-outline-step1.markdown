---
layout: post
title: "airead-manager-outline-step1"
date: 2014-06-15 12:31
comments: true
categories: airead-manager
---

添加一个新的后端接口
<!-- more -->

## 框架结构
```
aireadManager
├── blueprints  # 放置 flask 蓝图文件夹
│   └── rest  # restful 蓝图文件夹
├── config  # 配置文件夹
├── model  # 数据库模型文件夹
├── static  # 前端静态文件夹
│   ├── coffee  # coffee script 源码
│   │   └── controllers  # 子页面 controller
│   ├── css  # css 文件夹
│   │   ├── fonts  # 字体文件夹
│   │   └── lib  # css 库文件
│   ├── js  # 由 coffee 编译生成
│   │   └── controllers  # 由 coffee 编译生成
│   ├── lib  # js 库文件
│   └── templates  # 子页面模版文件
├── test  # 后端测试脚本
└── utils  # 后端工具类文件夹
```
<!-- more -->

## 创建新的 url
在 `blueprint` 里创建一个文件 `hello.py`.
{% include_code airead-manager/hello.py %}
保存后，接口 `/hello/` 已经创建完毕

## 运行服务测试
`$ python aireadManager/main.py`

然后打开 [http://localhost:5000/hello/](http://localhost:5000/hello/), 可以看到浏览器显示

    hello example！
    
**url 创建成功**

`git checkout step1` 查看源代码