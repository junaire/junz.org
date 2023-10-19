---
title : "Linux用户管理"
author : "Jun"
date : 2021-02-28T15:38:30+08:00
categories : ["Linux"]
tags : ["Linux"]
---



## `UID`与`GID`
Linux 并不认识用户名，当我们登陆Linux系统时，系统会依据`/etc/passwd`和`/etc/group`中的内容找到用户的`UID`和`GID`，从而判断用户的账号与群组。
如果我们修改某一账号的`UID`为0，则该账号将实际获得`root`的权限。

## `/etc/passwd`
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
```
各个部分含义
-  账号名称
- 密码，现在已经废弃，密码实际存放在`/etc/shadow`中
- `GID`
- 使用者信息
- 用户家目录
- 用户的shell，如果为`/sbin/nologin`， 则无法登录
## `/etc/group`

```
root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:syslog,user
```

- 群组名称
- 群组密码（已弃用，现在只存在`x`）
- `GID`与`/etc/passwd`对应
- 群组内的用户成员，可以有多个

### 初始群组

`/etc/passwd`中用户的`GID`对应的群组，为首要群组

### 有效群组

`/etc/group`中用户所存在的群组，为次要群组

## 创建用户

```
useradd -u[UUID] -g [指定初始化群组]　-G [指定次要群组]　-d [指定用户家目录] -s［shell］
```

### 修改用户密码

```
echo "PASSWD" | passwd --stdin user_name
```


## 查询用户信息

### 查询已登录用户
```
w
who
```
### 查询最近登录的用户
```
last
```
### 查询最近登录失败的用户
```
lastb
```

## `sudo`
`sudo`使用户可以暂时得到`root`权限去执行一些命令，相关配置文件为`/etc/sudoers`

### 添加某一用户为`sudo`用户
- 修改配置文件
```
user_name ALL=(ALL) ALL
```
- 添加用户至相关群组

```
usermod -aG wheel user_name // for centos 
usermod -aG sudo user_name // for ubuntu
```
