---
title : "Linux常用命令总结"
author : "Jun"
date : 2021-02-28T15:38:25+08:00
categories : ["Linux"]
tags : ["Linux"]
---

### 本文使用Cent OS 7

#### 编译安装

```bash
./configure
make
make install
```



#### rpm使用

```bash
rpm -i xxx.rpm //安装
rpm -e xxx.rpm //卸载
rpm -qa | grep "xxx" //查询xxx
rpm -v //显示执行信息
rpm -U xxx.rpm //升级xxx
```

#### 压缩与解压缩

- tar

```bash
tar -cf result.tar *.jpg //-c 是表示产生新的包，-f 指定包的文件名。
tar -rf result.tar *.jpg //-r 是表示增加文件的意思
tar -xf result.tar //-x指解压

tar -czf result.tar.gz *.jpg//-z指压缩成.gz
tar -cjf result.tar.bz2 *.jpg//-j指压缩成bz2
```

- zip

```bash
zip -r result.zip /src //-r指递归压缩
zip -v result.zip *.jpg //-v指显示详细信息

unzip xxx.zip -d xxx
```



#### 查看相关系统信息

- 硬件

```
uname -a //查看内核信息
cat /proc/cpuinfo //查看cpu信息
```

- 资源

```
free -m //查看内存信息
df -h //查看各分区
uptime //查看系统运行时间
cat /proc/loadavg //查看系统负荷

```

- 网络

```
netstat -lntp //查看所有监听端口
lsof -i tcp:xxx //查看xxx端口使用情况
firewall-cmd --zone=public --add-port=xxx/tcp --permanent //开放xxx端口
firewall-cmd --reload //重启防火墙
firewall-cmd --query-port=xxx/tcp //查询xxx端口是否开放
```

- 进程

```
ps -ef //查看所有进程
top //查看实时进程信息
```

- 用户

```
whoami //查看当前用户
last //查看上一次登录信息
```

#### 修改时区

```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

#### 编译依赖

```
yum groupinstall "development tools"
yum install -y openssl-devel, libelf-dev, libelf-devel, ncurses-devel
```

