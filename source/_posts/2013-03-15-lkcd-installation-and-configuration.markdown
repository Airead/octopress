---
layout: post
title: "LKCD 的安装和配置"
date: 2013-03-15 13:35
comments: true
published: false
categories: kernel
---

**目的：** 介绍如何安装和配置 Linux Kernel Crash Dump(LKCD) 工具
**不涉及：** 不涉及如何使用 LKCD 工具去分析一个 linux crash dump。它只告诉你如何去获取一个可分析 crash dump。
**预置知识：** 你需要知道如何构建、安装和启动 linux kernel。

## LKCD 简介
### 简介
通常，当 Linux 系统出现故障时，保存系统内存的镜像是有必要的。它允许对故障做延后分析。一旦需要保存的镜像(crash dump)被存入磁盘，系统会重启到生产环境。

### 什么是 LKCD
LKCD 项目创建了一系列工具链和 kernel patches, 它们可以捕获一个 crash dump。其它还包括了让 kernel 开发人员分析这些 crash dump 的工具。

### 我应该什么时候安装 LKCD
LKCD 必须在故障前被安装！系统管理员在最初配置系统的时候应该被鼓励安装 LKCD。如果等到发生故障后再安装 LKCD，那么就会需要再出现一次故障，因此会延后解决问题的时间。

### 什么时候能拿到 crash dump
一旦 LKCD 安装后，crash dump 会自动生成，当：
  - Kernel Oops 时
  - Kernel panic 时
  - 系统管理员在终端键入 Alt-SysRq-c 来生成一个 crash dump 时
  
### LKCD 支持哪些平台
下面的平台在 LKCD 第四个版本中被支持：
  - i386
  - ia64
  - alpha
  
## Crash Dump 过程
### 简介
使用 LKCD 来创建 crash dump 的过程会在本节讨论。

### 过程
捕获 crash dump 的过程分两步。**第一**,系统内存的内容被复制到一个临时的磁盘位置，称其为 dump device. **第二**，Linux 被启动，把保存的内存镜像由 dump device 移动到永久储存的路径 `/var/log/dump`。dump device 和储存路径都是可配置的。

### 保存了什么
在 crash dump 过程完成后，下表中的文件将会生成。这些文件被保存在 `/var/log/dump`的子目录中。子目录的名字为0,1,2等等。每一次 dump 都会对应一个新的数字目录。
  - dump.n: crash dump 文件。n 随着第一次的 crash dump 文件增涨。`/var/log/dump/bounds` 保存下一次 crash dump 的 n 值
  - kerntypes.n: 这个文件包含了 linux kernel 使用的数据结构的信息。 Kerntypes 最初在 kernel 编译的时候生成。在 kernel 安装时被复制到 `/boot` 中。它随着系统的 crash 再次复制到 `/var/log/dump`。
  - map.n: 这个文件包含 linux kernel 的符号表。它是 `/boot/System.map` 的副本。
  - analysis.n: 这个文件随着系统 crash 自动生成。它包含 crash 文本分析，可能对分析 crash 有用。
  - lcrash.n: crash 时，系统 `/sbin/lcrash 工具的副本。
  - bounds: 包含创建下一次 dump 的子目录名。

### 非中断的 dumps
LKCD 提供选项生成非中断的 dumps。这意味着在生成 dump 之后，系统将会继续运行（如果可能）。当配置非中断dumps后，dump过程不会进行第一步的重启。非中断的 dump 只可能在 kernel Oops 或当键入 Alt-SysRq-c 时（版本4.01或更新）发生。panic 总是会中断操作，因此即使配置了非中断 dumps,系统也会在 panic 后重启。

**注意**：
  1. `lkcd save`命令必需在非中断 dump 发生且已经创建 dump 文件之后运行。
  2. 当配置非中断 dumps（参考选择 Dump Device）时，一个专用的 dump device 必须被使用。
  
## 命令，脚本，文件
### 简介
本节介绍 LKCD 包中包含或使用的命令，脚本，工具和其它重要的文件。

### 命令和脚本
运行和配置 LKCD 包的命令和脚本有：
  - `/sbin/lkcd`: 这个脚本用来配置和保存 crash dump。它接受2个参数: config 或 save. `config` 被用来把系统 dump 参数设置到正在运行的 kernel。`save` 用来保存 crash dump 到磁盘上（经过两步）。
  - `/etc/sysconfig/dump`: 包含 LKCD 的配置变量。这些变量被 `lkcd config` 命令使用。编辑这个文件来配置 dump.
  - `/sbin/lkcd_config`: 这个二进制文件被 `/sbin/lkcd/` 调用，用来设置系统 dump 参数。
  - `/sbin/lcrash`: linux crash dump 分析器
  
### 其它文件和路径
几个重要的文件和路径:
  - `/proc/sys/dump/*`: 这个路径的文件是用来查看 kernel 当前 crash dump 的配置的。每一个 `/etc/sysconfig/dump` 中的参数都会在这个路径中生成一个文件。使用 `cat` 命令读取它们。
  
## 选择 Dump Device

## 配置选项

## 安装 LKCD
### 简介
本节介绍如何安装和配置 LKCD

### 前期需求
安装 LKCD 需要编译新的 Linux kernel。在安装 LKCD 之前确保 kernel 源与你目前系统正在运行的kernel是相匹配的。一个好的主意是，先验证新的 kernel 可以被编译，安装且能启动后，再安装 LKCD 的补丁。

### 获取 LKCD 包
最新的 LKCD 可以从 [http://lkcd.sourceforge.net/](http://lkcd.sourceforge.net/) 下载。
