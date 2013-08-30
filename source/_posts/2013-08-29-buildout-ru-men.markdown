---
layout: post
title: "buildout 入门"
date: 2013-08-29 23:39
comments: true
categories: python buildout
---
这篇文章会简单介绍一下 zc.buildout 的安装，然后说明如果使用 buildout 简单的进行包的安装，生成基于 buildout 环境(sys.path) 的 python 解释器。

{% blockquote worldhello, buildout 使用小例 %}
使用setuptools能够用来打包应用程序，生成egg包，但对于有些应用程序来说，可能需要依赖很多egg包，在此基础上进行开发。这时setuptools就无能为力了。幸运的是，我们还有buildout。buildout不但能够像setuptools一样自动更新或下载安装依赖包，而且还能够像virtualenv一样，构建一个封闭的开发环境。
{% endblockquote %}

吐槽先，[zc.buildout](http://www.buildout.org/en/latest/)的官方文档对于 buildout 的初学者来说还真是不容易看。相对来说，pypi中的[文档](https://pypi.python.org/pypi/zc.buildout/2.2.0)还算可以。但是它全程都是使用 **doctest-based** 的方式来说明，悲剧的是，`>>>` 后面的的命令还是不能运行的，这让初学者情何以堪呐。入正题。

<!-- more -->

- 操作系统：Ubuntu 12.04.2
- zc.buildout: version 2.2.0
- 整个过程需要网络支持


## 安装 zc.buildout
在 Ubuntu 上安装 buildout 非常简单：

`sudo pip install zc.buildout`

好了，你现在应该成功安装了。

## 初始化 buildout 根路径（这个词用的不好)
因为我们是测试，所以选择临时路径来耍耍 buildout.

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

然后在 buildout 所在路径下运行 `bin/buildout`，终端会显示安装的过程，有必要说一下直接运行 buildout 时真正运行的是 `/usr/local/bin/buildout`, 而现在我们运行的是 `/tmp/test/bin/buildout`：

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

这时，bin 下多了 ab-gtk.py 和 ab.py, 这两个脚本都是软件包 ab 要求安装的运行脚本。也就是说，软件要求安装运行脚本会被装在 bin 下。eggs 下也出现了两个 .egg, 它们分别是我们想要装的 ab，和 buidout 自动安装的 zc.recipe.egg, 因为在 buildout.cfg 中配置了 `recipe = zc.recipe.egg`。也就是说，buildout 会自动下载需要的菜谱(recipe)包。

简单介绍一下 buildout.cfg。每个 buildout.cfg 必须包含一个 [buildout] 段，可以想像它为程序的入口(main)函数，buildout 就是从这里开始进行构建。[buildout] 里必须包含 parts 项，parts 项允许为空。可以想像 parts 的每个值是被 [buildout] 调用的子函数。

那么好了，整个 buildout 的流程是：先调用 [buildout]，发现里 parts 里有 ab\_part 这个子函数，然后调用 ab\_part，ab\_part 也会调一个子函数，就是 ab\_part 中的 recipe 项。除了 recipe 项，ab\_part 中的其它项都可以认为是 recipe 函数的参数。当 zc.recipe.egg 被调用时，buildout 发现，它还没有被安装，于是就自动安装它。接着，zc.recipe.egg 会安装所以 eggs 指定的软件包。结果就是 eggs 里出现了 `ab-0.3-py2.7.egg` 和 `zc.recipe.egg-2.0.0-py2.7.egg` 两个 egg。

这时我们就可以通过　`bin/ab.py` 运行 ab 程序了。

既然文章开头说过，buildout 可以构建一个封闭的开发环境，那么想不想看一下 buildout 封闭环境的效果？

## 安装 python 解释器
其实安装 python 解释器很简单啦，只需要将参数 `interpreter = py` 传递给 zc.recipe.egg 就行啦。说干就干，编辑 buildout.cfg

{% codeblock lang:cfg %}
[buildout]
parts = ab_part

[ab_part]
recipe = zc.recipe.egg
eggs = ab
interpreter = python
{% endcodeblock %}

在根路径(/tmp/test/)下运行 builout 后，bin 目录下会多出一个可执行的 python 文件。打开 `/tmp/test/bin/python` 这个文件后，它开头有这样一段

{% codeblock lang:py %}
#!/usr/bin/python

import sys

sys.path[0:0] = [
  '/tmp/test/eggs/ab-0.3-py2.7.egg',
  ]
{% endcodeblock %}

现在发现了 buildout 创建封闭环境的神奇之处了吧，它直接把环境变量改成了我们刚安装的 ab egg 的路径，buildout 就是用这种方法建造独立的封闭环境。

buildout 会将所有依赖的软件包安装在 eggs 路径下，然后在生成可执行脚本时会直接改变 sys.path 创建封闭环境的 path。

## 验证封闭环境是否生效
<pre>
airead@AIREAD:/tmp/test$ bin/python
>>> import sys, pprint
>>> pprint.pprint(sys.path)
['/tmp/test/eggs/ab-0.3-py2.7.egg',
 '/tmp/test/bin',
 '/usr/lib/python2.7',
 '/usr/lib/python2.7/plat-linux2',
 '/usr/lib/python2.7/lib-tk',
 '/usr/lib/python2.7/lib-old',
 '/usr/lib/python2.7/lib-dynload',
 '/usr/local/lib/python2.7/dist-packages',
 '/usr/lib/python2.7/dist-packages',
 '/usr/lib/python2.7/dist-packages/PIL',
 '/usr/lib/python2.7/dist-packages/gst-0.10',
 '/usr/lib/python2.7/dist-packages/gtk-2.0',
 '/usr/lib/pymodules/python2.7',
 '/usr/lib/python2.7/dist-packages/ubuntu-sso-client',
 '/usr/lib/python2.7/dist-packages/ubuntuone-client',
 '/usr/lib/python2.7/dist-packages/ubuntuone-control-panel',
 '/usr/lib/python2.7/dist-packages/ubuntuone-couch',
 '/usr/lib/python2.7/dist-packages/ubuntuone-installer',
 '/usr/lib/python2.7/dist-packages/ubuntuone-storage-protocol']
>>> 
</pre>

结果令人大跌眼镜，eggs 里 ab-0.3-py2.7.egg 确实被添加进了 sys.path, 但是 sys.path 里为什么会多出来这么多 `/usr/*`。按照上面我们看到的脚本内容理解，sys.path 应该完全被替换成 `/tmp/test/eggs/ab-0.3-py2.7.egg`，但现在显然不是。具体原因先不解释，因为我还不清楚，但这已经证明了我们创建封闭式开发环境的尝试失败了。

目前我所知道的创建真正封闭开发环境的方法是借助 [virtualenv](http://www.virtualenv.org/en/latest/) 先创建一个干净的虚拟环境，然后用虚拟环境中的 buildout 来构建封闭环境。

至此，你已经大概了解了 buildout 执行流程，也能够使用 buildout 简单的安装软件包了。

**悄悄告诉你**，使用 buildout 的 `download-cache` 选项可以把下载的软件包保存起来，利用这个特性你可以把自己常用的软件包保存起来以备不时之需，它可以自动解决依赖哟。

## References
- [https://pypi.python.org/pypi/zc.buildout/2.2.0](https://pypi.python.org/pypi/zc.buildout/2.2.0)
- [http://www.buildout.org/en/latest/index.html](http://www.buildout.org/en/latest/index.html)
- [http://www.worldhello.net/2010/12/10/2207.html](http://www.worldhello.net/2010/12/10/2207.html)
