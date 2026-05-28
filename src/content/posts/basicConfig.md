---
title: linux的基本配置
published: 2026-05-28
description: 一些基本的开荒配置
image: " "
tags: [linux]
category: 学习笔记
draft: false
---

下面是我自己整理的一些基本的配置命令


不过那些过于基础的基础的常用命令我就不整理了

*类似于：cd,ls,systemctl什么的...*


**(这里我用的是Debian的系统哦，红帽系列的也可当作参考)**

*基本上每一台新linux系统都会经历的...*

```

用户：

useradd <>
passwd <>
usermod -aG sudo <>



网络配置：

nmtui
systemctl restart NetworkManager



环境变量：

/etc/profile
export PATH=$PATH:/usr/sbin

/etc/bash.bashrc
alias ll='ls -lh'
source /etc/bash.bashrc

**/etc/profile	全局登陆**
**/etc/bash.bashrc	全局bash**
**~/.bashrc	用户bash**
**source  <刚刚编辑的配置文件>**

换源：

备份：
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
vi /etc/apt/sources.list

sudo apt update




ufw防火墙：

sudo apt intstall -y ufw
sudo systemctl start ufw
sudo systemctl enableufw
sudo ufw status
sudo ufw enable
sudo ufw allow <>/<>

```