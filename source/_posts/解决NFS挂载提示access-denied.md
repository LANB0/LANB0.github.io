---
title: 解决NFS挂载提示access denied
date: 2020-07-30 10:50:53
categories: 技术
tags: [NFS,access denied]
---

## 前言
最近在倒腾工作站分配的虚拟机时，为了可以在WSL访问虚拟机的资源，将虚拟机的data目录以NFS的方式挂载到WSL以便vscode的远程访问。（主要是为了偷懒，不用重新配置远程环境）
<!--more-->
## NFS server配置步骤

### 安装server程序
```
sudo apt update
sudo apt install -y nfs-kernel-server
```

### 配置NFS目录
```
sudo vim /etc/exports
```
在文件末尾添加：
```
/data *(rw,sync,no_root_squash,no_subtree_check)
```

### 重启服务
```
sudo /etc/init.d/rpcbind restart
sudo /etc/init.d/nfs-kernel-server restart
```

## 客户端挂载
### 安装客户端程序
在客户端执行以下指令
```
sudo apt update
sudo apt install -y nfs-common
```

### 执行挂载指令
```
sudo mount -t nfs -o nolock 192.168.1.187:/data ~/work/xzl
```

## 问题
正常情况下就会报以下错误：
```
mount.nfs: access denied by server while mounting 192.168.1.187:/data
```

## 分析及解决方法

### 原因

如果端口号大于1024，则需要将 insecure 选项加入到配置文件（/etc/exports）相关选项中mount客户端才能正常工作

### 解决方法
```
sudo vim /etc/exports
```
将安装时的添加行改为:
```
/data *(insecure,rw,sync,no_root_squash,no_subtree_check) 
```
这里不能使用IP地址，一定要用*，加insecure

修改后参考安装步骤中指令重启服务即可成功挂载