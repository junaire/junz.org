---
title: "GDB基本使用笔记"
date: 2021-10-05T14:36:52+08:00
author: "Jun"
---

## GDB简介

gdb全称GNU Debugger，是一个支持多语言的调试工具。


## 使用

### 编译时启用debug symbol

```
gcc demo.c -g
```

### 开启GDB调试

```
gdb ./a.out
```

### 调试时加入参数

假设原程序以`./a.out 4 2`的方式运行，我们可以在开启gdb后以`r 4 2`的方式调试。



### 更友好的文本界面

- ctrl-x-a 打开文本界面
- ctrl-l  重新渲染界面
- ctrl-x-2 查看反汇编和寄存器
- ctrl-x-1 返回原界面
- ctrl-p 上一条指令
- ctrl-n 下一条指令


### 断点

- `break function_name` 在某一函数开始时断下来
- `break *0x1234` 在某一地址断下来
- `break 42` 在某行断下来

##### 条件断点

```
break [] if EXPRESSION
```

比如：

```
break func if a == 1
```

#### 删除断点

- `delete 1` 删除某一断点
- `delete` 删除所有断点

### 执行


- `next`
- `step`
- `finish`
- `continue`

### 查看数据

- `print`

  - `print $rax`
  - `print variable`
  - `print *(int*)0x1234`
- `info`

  - `info registers`
  - `info frame`
  - `info break`
- `display` 始终打印出某一变量或寄存器
  - `undisplay` 取消打印

- `whatis` 检查类型



### 回放调试

- `record` 在程序开始运行后执行，开始录制
- `reverse-stepi`
- `reverse-contiune`



## 外部脚本

### Python

```
(gdb) python
```

### Shell

```
(gdb) shell COMMANDS
```

## GDB配置文件

gdb的配置文件位于`~/.gdbinit`中

示例配置：

```
set print pretty on
set print object on
set print static-members on
set print vtbl on
set demangle-style gnu-v3kip -gfi /usr/include/c++/*
```
