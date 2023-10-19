---
title : "Linux开机流程"
author : "Jun"
date : 2021-02-28T15:38:20+08:00
tags : ["Linux"]
categories : ["Linux"]
---

# Linux开机流程

## 综述

１.载入BIOS或者UFEI，进行硬件检测

2.载入硬盘第一个扇区(MBR)，读取其中的Boot Loader，载入内核，或者转交给其他Boot Loader进行系统启动工作．

3.内核开始调用systemd，并以default.target准备开机

4.用户登录，获得shell

## BIOS与Boot Loader

计算机首先载入BIOS，其间BIOS会检测硬件设备是否正常．接着，BIOS会通过硬件的INT 13中断功能读取硬盘的第一个扇区(MBR)，而其中储存着boot loader，它负责识别操作系统的文件格式并载入内核．

目前Linux的Boot Loader主要是grub2

grub2最主要的配置文件在`/boot/grub/grib.cfg`中，一般不建议直接修改，我们应通过`/etc/default/grub`进行控制


### 多系统的问题

每个操作系统在安装的时候都会自动安装一份Boot Loader到自己的文件系统中，对于Linux，我们还可以选择安装一份Boot Loader到MBR中，当然也可以选择不安装．但对于Windows来说，他会自动将自己的Boot Loader覆盖到MBR上，导致无法多系统引导．所以应该先安装Windows，再安装Linux．在开机时，我们可以在MBR中选择将开机管理功能转交到其他Boot Loader上，达到多系统引导的目的．

```
+---+----------+------+--------------+
|MBR|          | boot |              |
|---+ partion1 |loader|    partion2  |
|              |------+              |
+------------------------------------+
```



## 载入内核

Boot Loader开始读取内核后，Linux就会将内核解压缩到内存中，接管BIOS的工作，再次检测硬件设备的状态．

最关键的内核部分一般位于`/boot/vmlinuz`中，其他次要的功能内核部分被编译成模块，位于`/lib/modules/$(uname -r)/kernel`中．

### 虚拟文件系统

开机时我们需要挂载根目录，而要挂载硬盘的驱动位于`/lib/modules`中，即根目录中，从而陷入了死循环．对此，我们通过虚拟文件系统(Initial RAM Disk)，位于`/boot/initrd`或者`/boot/initramfs`中，它会在内存中虚拟一个根目录，并提供一个可执行程序，载入一些与硬盘相关的内核模块．

## 开机后的第一个程序`systemd`
