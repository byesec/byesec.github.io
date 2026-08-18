---
title: Multipass安装与使用
author: 干 脆 面
tags:
  - Multipass
categories:
  - 软件
top: 9
abbrlink: 99b06d32
date: 2022-01-24 10:59:35
---

## 一、Multipass介绍

**Multipass** 是您工作站上的迷你云，使用所有受支持平台（Windows、macOS 和 Linux）的本机管理程序，只需单击（“打开 shell”）或简单`multipass shell`命令，它就会为您提供 Ubuntu 命令行，甚至键盘快捷键。

**Multipass** 是一个轻量虚拟机管理器，是由 **Ubuntu** 运营公司 **Canonical** 所推出的开源项目。运行环境支持 **Linux**、**Windows**、**macOS**。在不同的操作系统上，使用的是不同的虚拟化技术。在 **Linux** 上使用的是 **KVM**、**Window** 上使用 **Hyper-V**、**macOS** 中使用 **HyperKit** 以最小开销运行VM，支持在笔记本模拟小型云。

同时，**Multipass** 提供了一个命令行界面来启动和管理 **Linux** 实例。下载一个全新的镜像需要几秒钟的时间，并且在几分钟内就可以启动并运行 **VM**。

![image-20220124102540386](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220124102540386.png)
<!-- more -->
## 二、安装Multipass

要在 **macOS** 上安装 Multipass，您有两个选择：安装程序包或[`brew`](https://brew.sh/)

### 安装程序包

下载程序https://github.com/CanonicalLtd/multipass/releases/

卸载脚本：

```bash
$ sudo sh "/Library/Application Support/com.canonical.multipass/uninstall.sh"
```

### 通过brew安装

```bash
$ brew install --cask multipass
```

卸载：

```bash
$ brew uninstall multipass
# or
$ brew uninstall --zap multipass # to destroy all data, too
```

## 三、使用方式

### 帮助指南

```bash
$ multipass help
```

```
Usage: multipass [options] <command>
Create, control and connect to Ubuntu instances.

This is a command line utility for multipass, a
service that manages Ubuntu instances.

Options:
  -h, --help     Displays help on commandline options.
  --help-all     Displays help including Qt specific options.
  -v, --verbose  Increase logging verbosity. Repeat the 'v' in the short option
                 for more detail. Maximum verbosity is obtained with 4 (or more)
                 v's, i.e. -vvvv.

Available commands:
  alias     Create an alias
  aliases   List available aliases
  delete    Delete instances
  exec      Run a command on an instance
  find      Display available images to create instances from
  get       Get a configuration setting
  help      Display help about a command
  info      Display information about instances
  launch    Create and start an Ubuntu instance
  list      List all available instances
  mount     Mount a local directory in the instance
  networks  List available network interfaces
  purge     Purge all deleted instances permanently
  recover   Recover deleted instances
  restart   Restart instances
  set       Set a configuration setting
  shell     Open a shell on a running instance
  start     Start instances
  stop      Stop running instances
  suspend   Suspend running instances
  transfer  Transfer files between the host and instances
  umount    Unmount a directory from an instance
  unalias   Remove an alias
  version   Show version details
```

### 查看正在运行的**Multipass**版本

```bash
$ multipass version
```

```
multipass   1.8.1+mac
multipassd  1.8.1+mac
```

### 快速创建运行

打开**Multipass**--打开**状态栏图标**--**Open Shell**

![image-20220124102731263](https://byesec-blog-img.oss-cn-beijing.aliyuncs.com/uPic/image-20220124102731263.png)

### 查看可下载安装的Ubuntu镜像

```bash
$ multipass find
```

```
Image                       Aliases           Version          Description
18.04                       bionic            20220104         Ubuntu 18.04 LTS
20.04                       focal,lts         20220118         Ubuntu 20.04 LTS
21.10                       impish            20220118         Ubuntu 21.10
anbox-cloud-appliance                         latest           Anbox Cloud Appliance
charm-dev                                     latest           A development and testing environment for charmers
minikube                                      latest           minikube is local Kubernetes
```

### 下载镜像配置并运行

```bash
$ multipass launch -n vm01 -c 1 -m 1G -d 10G
```

```
-n, --name: 名称
-c, --cpus: cpu核心数, 默认: 1
-m, --mem: 内存大小, 默认: 1G
-d, --disk: 硬盘大小, 默认: 5G
```

### 查看虚拟机列表

```bash
$ multipass list
```

```
Name                    State             IPv4             Image
primary                 Running           192.168.64.2     Ubuntu 20.04 LTS
vm01                    Running           192.168.64.3     Ubuntu 20.04 LTS
```

### 查看虚拟机信息

```bash
$ multipass info vm01
```

```
Name:           vm01
State:          Running
IPv4:           192.168.64.3
Release:        Ubuntu 20.04.3 LTS
Image hash:     6fbeb622aace (Ubuntu 20.04 LTS)
Load:           0.41 0.14 0.05
Disk usage:     1.2G out of 9.5G
Memory usage:   135.0M out of 970.0M
Mounts:         --
```

### 在虚拟机外部执行命令

```bash
$ multipass exec vm01 pwd
```

```
/home/ubuntu
```

### 进入虚拟机

```bash
$ multipass shell vm01
```

成功进入终端

```
ubuntu@vm01:~$
```

### 宿主机挂载虚拟机

```bash
$ multipass mount /mnt vm01:/mnt
```

### 宿主机卸载虚拟机挂载

```bash
$ multipass unmount vm01
```

### 重启虚拟机

```bash
$ multipass restart vm01
```

### 删除虚拟机

```bash
$multipass delete vm01
$multipass delete --purge vm01 彻底删除
```

### 恢复删除虚拟机

```bash
$ multipass recover vm01 
```

### 暂停虚拟机

```bash
$ multipass stop ubuntu-lts
```

### 启动虚拟机

```bash
$ multipass start vm01
```

### 挂起虚拟机

```bash
$ multipass suspend vm01 
```

