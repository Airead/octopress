---
layout: post
title: "buildout 入门"
date: 2013-08-29 23:39
comments: true
categories: 
---

吐槽先，[zc.buildout](http://www.buildout.org/en/latest/)的官方文档对于 buildout 的初学者来说还真是不容易看。相对来说，pypi中的[文档](https://pypi.python.org/pypi/zc.buildout/2.2.0)还算可以。但是它全程都是使用**doctest-based Documentation**的方式来说明，悲剧的是，`>>>` 后面的的命令还是不能运行的，这让初学者情何以堪呐。入正题。

这篇文章会简单介绍一下 zc.buildout 的安装，然后说明如果使用 buildout 简单的进行包的安装，生成基于 buildout 环境(sys.path) 的 python 解释器。
<!-- more -->

- 操作系统：Ubuntu 12.04.2
- zc.buildout: version 2.2.0
- 整个过程需要网络支持


## 安装 zc.buildout
在 Ubuntu 上安装 buildout 非常简单：

`sudo pip install zc.buildout`

好了，你现在应该成功安装了。

## 初始块 buildout 根路径（这个词用的不好)
因为我们是测试，所以选择临时路径来玩 buildout.

```
mkdir /tmp/test
cd /tmp/test
```

在 `/tmp/test/` 中运行：

`buildout init`

如果系统自带 setuptools 的版本小于 0.7 的话，会报错：

`VersionConflict: (setuptools 0.6c11 (/usr/local/lib/python2.7/dist-packages), Requirement.parse('setuptools>=0.7'))`

只要把 setuptools 升级一下就可以了：

`sudo pip install --upgrade setuptools`

然后使用 `rm` 把 `/tmp/test/`清空，再运行

`buildout init`

这根路径中会产生如下内容：

```
/tmp/test
├── bin
│   └── buildout
├── buildout.cfg
├── develop-eggs
│   ├── setuptools.egg-link
│   └── zc.buildout.egg-link
├── eggs
└── parts
```

先只介绍 builout.cfg 这个文件。buildout.cfg 是 生成 buildout 发布环境的配置文件，我们使用 `buildout init` 生成的这个配置文件是最简化的合法配置。

```
[buildout]
parts =
```

## 使用 buildout 安装包
我们将要安装 `ab` 这个软件包，这个包是我临时从 pypi 上找的，希望你还没有装过这个包，原因以后说。那么，编辑 buildout.cfg:

{% codeblock lang:cfg %}
[buildout]
parts = ab_part

[ab_part]
recipe = zc.recipe.egg
eggs = ab
{% endcodeblock %}

然后在 buildout 所在路径下运行 `bin/buildout`，终端会显示安装的过程：

<pre>
airead@AIREAD:/tmp/test$ bin/buildout 
Getting distribution for 'zc.recipe.egg>=2.0.0a3'.
Got zc.recipe.egg 2.0.0.
Installing ab_part.
Getting distribution for 'ab'.
zip_safe flag not set; analyzing archive contents...
Got ab 0.3.
Generated script '/tmp/test/bin/ab.py'.
Generated script '/tmp/test/bin/ab-gtk.py'.
</pre>

这时，`/tmp/test/` 下的目录结构如下：

```
/tmp/test
├── bin
│   ├── ab-gtk.py
│   ├── ab.py
│   └── buildout
├── buildout.cfg
├── develop-eggs
│   ├── setuptools.egg-link
│   └── zc.buildout.egg-link
├── eggs
│   ├── ab-0.3-py2.7.egg
│   └── zc.recipe.egg-2.0.0-py2.7.egg
└── parts
```

## References
    - [pypi_buildout](https://pypi.python.org/pypi/zc.buildout/2.2.0)
