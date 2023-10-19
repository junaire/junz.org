---
title : "Linux硬盘管理"
author : "Jun"
date : 2021-02-28T15:38:05+08:00
---



## 新增硬盘基本步骤

1. 创建磁盘分区
2. 格式化分区，创建文件系统
3. 挂载到某一目录下

## 磁盘分区

在Linux中一切皆文件，包括硬盘和分区，他们都位于`/dev`目录下。

### 各种接口的磁盘在Linux中的名称

- /dev/sd[a-p]1-15为SCSI, SATA, U盘, Flash闪盘等接口的磁盘文件名
- /dev/hd[a-d]1-30为 IDE 接口的磁盘文件名

### 分区的目的

- 提升数据的安全性(一个分区的数据损坏不会影响其他分区的数据)
- 支持安装多个操作系统

### 分区种类

可以用 `sudo fdisk -l` 查看

- `MBR`
  - 由主分区、扩展分区和逻辑分区组成。
  - 主分区的最大个数为4
  - 扩展分区也是一个主分区，并且最多只能有一个扩展分区。
  - 对于逻辑分区，Linux 规定它们必须建立在扩展分区上。
- `GPT`
  - `GPT`头
  - 分区表（在最开头，仍然存储了一份传统的`MBR`，以向后兼容）
  - `GPT`分区
  - 备份区

### 注意点
- 如果使用GRUB legacy作为bootloader，必须使用`MBR`。
- 如果使用传统的BIOS，并且双启动中包含 Windows ，必须使用`MBR`。
- 如果使用 UEFI 而不是BIOS，并且双启动中包含 Windows 64位版，必须使用`GPT`。
- 没有限制的情况下，优先选择 `GPT`。 



## 文件系统

磁盘分区后应该进行格式化，然后操作系统才能使用此分区

### Linux VFS (Virtual Filesystem Switch)

磁盘的各个分区可以有各自的文件系统，Linux的内核则通过`VFS`去读取文件系统


### 文件系统种类

#### ext文件系统
  - `inode` 负责存放文件权限(`rwx`)与文件属性(拥有者、群组、时间参数等）
  - `data block` 负责存放实际数据
  - `superblock` 记录整体信息，包括`inode/block`的总量、使用量、剩余量，以及文件系统的格式与相关信息等

#### xfs文件系统

  - data section
  - log section
  - realtime section

  #### vfat文件系统

  #### windows与Linux都可以识别，可以用来共享数据




## 挂载

在一个区被格式化为一个文件系统之后，这个时候Linux操作系统还找不到它，我们还需要把这个文件系统注册进Linux操作系统的文件体系里，这个过程称为挂载。

**挂载点必须是目录**

注意
- 根目录必须挂载，且最先挂载
- 所有分区在同一时间只能挂载一次
- 卸载文件系统时，必须位于其挂载目录

```
mount 分区名 目录挂载点
mount -a //挂载所有文件系统
mount -t //指定文件系统种类
mount -n //挂载但不将其写入/etc/fstab中
```

```
unmount -f 分区名或挂载点//强制卸载
unmount -n //不将改变写入/etc/mtab
```



### 开机挂载设置

Linux在开机时进行自动挂载文件系统，相关配置文件在`/etc/fstab`中．

配置文件文件结构

-　设备/UUID
-　挂载点
-　文件系统
-　文件系统的参数
　　-　 `auto/noauto`当执行`mount -a`时，此文件系统会被自动挂载
　　- 　`rw/ro`使文件系统以可读写或者只读的方式挂载上来
- 是否用`fsck`检查分区（现在直接填０即可）


## 基本命令

### 分区管理 

- `fdisk`(主要支持`MBR`)

- `parted`(既支持`MBR`也支持`GPT`)

  ```
  parted 硬盘名 命令
  
  常用命令：
  新建分区 mkpart [primary|logical|extend] [ext4|vfat|xfs] start end
  显示分区 print
  删除分区 rm [partion]
  ```

  

### 格式化文件系统

```
mkfs -t 文件系统种类 分区名
```



### 磁盘检查

**不要随意使用此命令！**

```
fsck [-t 文件系统] [-ACay] 分区名称

-t  ：指定文件系统
-a  ：自动修复检查到的有问题的分区
-C  ：显示目前的进度
```



### 创建swap分区

- 对硬盘进行分区
  - 新建分区
  - 将ID改为82（swap的识别码）
  - 写入更改后让内核升级 partition table 执行`partprobe`
- 创建swap的文件系统 `mkswap 分区名`


## LVM

### 基本概念

 #### 物理卷——PV（Physical Volume）

physical volume对应底层的物理硬盘或者物理分区 

需要将实际分区或者硬盘的系统识别码改为`8e`才能创建physical volume

相关命令

- `pvcreate`创建physical volume
- `pvscan`扫描系统中所有的physical volume
- `pvdisplay`显示目前系统上面的physical volume 状态
- `pvremove`移除physical volume的属性

  

#### 卷组——VG（Volume Group）

相关命令

- `vgcreate`创建一个volume group `vgcreate -s 100G VG_NAME PV_NAME`
- `vgextend`在volume group内增加physical volume
- `vgreduce`在volume group内删除physical volume
- `vgremove`删除volume group

#### 逻辑卷——LV（Logical Volume）

logical volume不同于volume group，必须使用全名，如`/dev/vg_test/lv_test`

相当于一个文件系统，可以被格式化。

相关命令

- `lvcreate`创建logical volume `lvcreate -L 10G -n LV_NAME VG_NAME`
- `lvextend`在logical volume内增加容量
- `lvreduce`在logical volume内删除容量
- `lvremove`删除一个logical volume
- `lvresize`调整logical volume的大小 `lvresize -L +10G /dev/vg_test/lv_test`

### 底层原理

### PE（Physical Extent）

创建 physical volume时，在逻辑上把物理存储介质划分为了 N 个大小相同的块。可以使用`pvdisplay`查看详细信息。

### LE（Logical Extent）

底层也是划分为若干个大小相同的块，一般情况下，LE 与 PE 一一对应。

### LVM 快照

假设需要备份的原始LV为`/dev/vg_raw/lv_raw`

备份快照名称为`/dev/vg_raw/vg_snapshot`

```
/lvcreate -s -l 5G -n vg_snapshot /dev/vg_raw/lv_raw
```

**因为在XFS文件系统中不允许相同UUID的文件系统的挂载，所以要加上`nouuid`这个参数**

```
xfsdump -l 0 -L lvm1 -M lvm1 -f /home/lvm.dump /srv/snapshot
```
