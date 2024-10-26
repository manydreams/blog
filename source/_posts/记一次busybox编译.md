---
title: 记一次busybox编译
date: 2024-10-26 21:16:22
categories:
 - Linux
tags:
 - Linux
 - Utils
 - Busybox
---

## 版权声明：本文依据[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/legalcode)许可证进行授权，转载请附上出处链接及本声明。

## 什么是 **busybox** ？

“我使用的 `route 200` 路由器中安装了busybox 1.37.0。我可以访问路由器的
`telnet`界面，并且可以访问他的shell。我想更改一些配置，然后添加一些工具。我可以
启用`ftp`吗？`scp`呢？”

像以上这些问题，表明提问者可能没有完全理解什么是 **busybox** 。**busybox**并不是一个
嵌入式设备的完整解决方案。**busybox**仅仅只是GNU/coreutils工具集的子集。尽管它足够
支撑起linux的运行，但却支撑不起一个嵌入式设备的需求，就像建筑工人的工具箱和材料不能自己
建造房子一样。

为了基于**busybox**的系统有用，系统开发人员们需要添加一些自己的部件。至少，
有一些`shell`脚本来控制要运行那些工具，何时以及如何运行。此外，开发人员通常
会在`kernel`和**busybox**之外添加一些其他工具。

到了这里相信你已经对最开始的问题有了答案。

## 为什么要使用 **busybox** ？

**busybox** 提供（集成）了许多常用linux工具，但相比于完整的GNU/coreutils的170多MB
，它仅有几MB大小，如果你愿意舍弃一些工具，甚至可以压缩到1MB以内，这对于嵌入式设备
来说是非常有利的。

## 获取 **busybox**

**busybox** 项目的主页：[https://busybox.net/](https://busybox.net/)

### 获取二进制文件

你可以在[这里](https://busybox.net/downloads/binaries/latest/)下载预构建的
二进制版本，当然从源码编译才是这篇文章的重点。

### 获取源码

你可以在[https://busybox.net/downloads](https://busybox.net/downloads)下载
源码

## 从源码构建 **busybox**

如果你想从源码构建 **busybox**，你需要安装一些依赖：

```bash
##  gcc, make, bzip2  ##
#ubuntu
sudo apt install gcc make bzip2

#archlinux
sudo pacman -S gcc make bzip2

##  curses  ##
#ubuntu
sudo apt install libncurses5-dev

#archlinux
sudo pacman -S ncurses

```

然后，解压并切换到源码目录：

```bash
tar -xvf busybox-x.y.z.tar.bz2
cd busybox-x.y.z
```

配置并编译：

>tips: 如果你的kernel版本在6.8以上，你需要在`make menuconfig`时，关闭tc，这是
一个已知的[Bug 15934](https://lists.busybox.net/pipermail/busybox-cvs/2024-January/041752.html)

```bash
make menuconfig
make    #你也可以使用多线程编译：make -j8
```

编译完成后，运行：

```bash
make install
```

这将把 **busybox** 安装到系统目录中，如果不希望将其安装到系统目录，二进制文件就在当前
目录下

运行以下命令以确定编译成功：

```bash
./busybox --help
```

到此为止，你已经成功编译（并安装）了 **busybox** 。

撒花(^&^)/

## 后记

这篇文章是我在kernel 6.11.5中编译busybox是踩到的坑，至于更详细的内容，先鸽着吧。

## 参考

1. [Busybox FAQ](https://busybox.net/FAQ.html)