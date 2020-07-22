---
title: 为WSL2编译最新版本的Linux内核
date: 2020-07-22 16:58:15
tags: [WSL2,Linux,kernel]
---

## 前言
WSL 2相比WSL 1，最大的改变就是使用了真正的Linux内核，而不仅仅是一个适配层。我们可以通过命令uname -r查看这个内核的版本，通过正式推送获得的WSL 2的内核版本应当是4.19.104。最近闲下来，折腾着装了archLinux来玩儿，滚动一时爽，一直滚动一直爽，爽完了发现内核版本似乎不大对，内核一直没更新掉啊，作为arch怎么可以不滚最新的内核。查资料发现，kernel居然在windows文件系统下，于是怒编之，替换之！

本次我们编译[https://www.kernel.org/](https://www.kernel.org/)上面最新的5.7.9版本的内核。

## 前置工作
首先，我们需要一个可用的Linux环境，我使用的是日常开发使用的Ubuntu18.04，作为内核编译的平台。

我们需要安装编译内核所必要的依赖：

sudo apt install g++ make flex bison libssl-dev libelf-dev bc
接着我们需要将内核源码下载下来，直接去[https://www.kernel.org/](https://www.kernel.org/)上下载即可，我下载的是最新的5.7.9版本，地址如下：
```
https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.7.9.tar.xz
```
接着我们使用如下的命令解压该压缩文件：
```
tar xvf linux-5.7.9.tar.xz
cd linux-5.7.9
```
接着，我们需要配置文件，否则在编译是就需要填写大量的选项，麻烦又无聊。我们可以使用微软官方的配置文件，但是官方配置比较老（4.19.84）。好在，有叶欣仁为我们整理了适合Linux 5.7.9的WSL 2的配置文件，文件在下方：
```
https://github.com/xieyubo/WSL2-Linux-Kernel/blob/wsl-xyb-port-5.7.y/Microsoft/config-wsl
```
下载该文件后，在linux-5.7.9文件夹下新建一个叫Microsoft的文件夹，将配置文件改名为config-wsl放入其中即可。

当然，由于是微软的配置文件，我们需要做自己加点料。
```
CONFIG_LOCALVERSION
```
你可以在这个字段下写一些个性化的东西，例如自己的名字，这样在新系统中使用uname -r查看内核版本时就会出现自己的名字。

## 开始编译
编译内核是在linux-5.7文件夹下进行的，在该文件夹下执行命令：
```
make KCONFIG_CONFIG=Microsoft/config-wsl
```
就开始了内核的编译工作，如果想要发挥多核心处理器的优势，可以命令后面加上-j n的参数，即可使用n个核心进行编译。

我使用8核心进行编译大概需要6分钟。

编译完成后的内核镜像存放为arch/x86/boot/bzImage文件，将其重命名为kernel，并拷贝到win 10中。

## 替换WSL内核
在替换内核之前，首先需要关闭正在运行的WSL实例，在CMD中使用如下命令即可：
```
wsl --shutdown
```
接着进入Win 10中的C:\Windows\System32\lxss\tools文件夹下，将kernel文件替换为刚刚编译的那个即可（当然最好把旧的那个备份下）。重启Windows 10。

再次进入WSL 2后，使用uname -r命令，就可以看到内核已经变成5.7.9版本的内核了，如果编译时加上了个性化信息的话，也会出现在内核版本的后面。