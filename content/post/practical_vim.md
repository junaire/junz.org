---
title: "Vim实用技巧"
author: "Jun"
date: 2021-03-27T11:04:30+08:00
categories: ["Vim"]
tags: ["Vim"]

---

## 前言

Vim 是一款开源的老牌编辑器，有编辑器之神的美称。但是它复杂的快捷键劝退了很多人，尤其是现在市面上涌现了一大批操作简单，功能强大的IDE。但在深入接触过Vim，已经形成惯性后，大多数情况下我还是选择使用Vim Key Binding的方式来编辑文件。可以说，我现在已经无法适应无Vim风格的编辑器了。下面我简单记录下自己对Vim的理解以及常用的tricks。

## Vim中的基本概念

### 模式

#### `Normal mode`
与大多数编辑器不同，当你打开Vim时并不能直接输入文字，而是进入了普通模式。在这个模式下，所有的按键都是快捷键，或者功能键。比如在此模式下你可以使用`h j k l`进行移动光标。

#### `Insert mode`
在此模式你就可以像现代编辑器一样进行正常的输入了，你可以在左下角看到提示。

你可以在普通模式输入以下键进入：

- `i`表示在光标前插入
- `a`表示在光标后插入
- `A`表示在行末插入
- `I`表示在行头插入
- `o`表示向下新建一行，并进入插入模式
- `O`表示向上新建一行，并进入插入模式

#### `Visual mode`
在可视化模式下，你可以选择任何数量的字符，相当于你用鼠标去选择一段文字。然后你可以用其他命令操作这段文字，比如复制，删除等。

你可以在普通模式下输入以下键进入：

- `v`
- `V`
- `<CTRL>v`

### 寄存器
Vim的寄存器是一组用于保存文本的简单容器。我们可以把一些命令储存在其中，然后重放它，从而减少重复操作的麻烦。我们也可以把一些文本储存在其中，当作剪切板一样使用。在Vim中，也就是说我们使用的复制粘贴等操作并不是对剪切板生效，而是对寄存器进行操作。这也解释了我们为什么要用`"*p`或者是`"+p`来粘贴系统剪切板中的内容。

### 缓冲区
当Vim打开一个文件时，实际上该文件的内容是被读入一个具有相同名字的缓冲区中，我们只有保存时，缓冲区中的变动才会被写入硬盘。每一个文件都对应一个缓冲区。

在Vim中，窗口是Vim的显示区域，我们可以让多个窗口同时显示一个缓冲区，也可以打开多个窗口，让他们分别显示不同的缓冲区。


## 操作多个文件

### 切分屏幕
- 竖直分割
```
<CTRL>w s
```
- 水平分割
```
<CTRL>w v
```
- 切换窗口
`<CTRL>w` 加上`h j k l`

### 标签页

Vim的标签页与大多数编辑器的标签页不同，我们可以理解为标签页是窗口的容器。也就是说，我们可以新建一个标签页，在其中继续拆分窗口。

#### 新建标签页
```
:tabe {filename}
```

#### 切换标签页
```
gt
```

#### 关闭当前标签页及其中窗口
```
:tabc
```

#### 只保留当前标签页
```
:tabo
```

###
Vim与大多数IDE不同，并没有直接显示一个目录树，但我们可以用以下命令打开：
```
:E
```

### 快速移动

我们可以使用`gf`在文件中快速跳转。比如在C语言中：
```
#include <stdio.h>
```
我们可以将光标移到`stdio.h`上，并输入`gf`跳转到其中。

如果显示不在Path中：
```
:set path+=./**
```
这样就会将当前目录及其下面所有的子目录包含到Path中，对Vim可见。

## Vim中的复用操作

### 宏
前面我们提到了寄存器，我们已经知道我们可以把一段修改序列录制下来，储存到寄存器中，用于之后的回放。这个过程，我们就称之为录制一个宏。

#### 开始录制
```
q{register}
```

#### 结束录制
```
q
```

#### 回放宏
```
@@ #回放最近寄存器中的宏

@{register} #回放指定寄存器中的宏
```

#### 回放指定次数的宏

我们可以使用次数加`@`来回放指定次数的宏。

可以使用`22@@`来快速回放宏，当发生错误时，Vim会自动停止。（用22是因为它与@刚好在一起，方便起见）

## Vim中的一些Tricks

### 替换某一个特定的单词

```
:%s/\<old\>/new/g
```

### 统计字数
```
g<C>g
```
### 删除行后多余的空格
```
:%s/\s\+$//
```
### 将终端执行的结果写入Vim中

假设你需要复制当前的目录路径
```
:r!pwd
```

### 保存无权限的文件

假设你正使用一个具有sudo权限的用户，并想要编辑一个属于Root的文件，比如`/etc/hosts`，你可能会输入：
```
vim /etc/hosts
```

可由于你并没有权限，也没有加`sudo`，所以你在保存的时候Vim会告诉你操作失败，权限错误。一般情况下我们可能会选择退出重新编辑，这里介绍一个小技巧：
```
:w !sudo tee %> /dev/null
```
此后终端会提示你输入密码，即可直接保存。

### 保存目录不存在的文件

假设你想新建一个src目录，并在其中新建一个文件，名为`main.c`，你可能会输入：
```
vim src/main.c
```
可由于src目录你并没有实际创建，所以直接保存时Vim会报错。一般情况下我们可能会选择退出重新编辑，这里介绍一个小技巧：
```
:!mkdir -p %:h
```
此后Vim会智能地创建`src`目录，然后我们正常保存就行了。

### 与系统剪切板交互

假设你正从StackOverFlow上找到了一段代码，并复制进了系统剪切板里，打算在Vim中使用`p`命令粘贴进去。这时候你可能惊奇地发现Vim粘贴的内容与你想要的不符。没错，Vim对于复制粘贴有自己的一套法则，而不是简单地使用系统剪切板。

我们可以使用以下命令粘贴系统剪切板中内容：
```
"*p
```
### 给制定内容加引号

使用此功能你需要安装[Surround.vim](https://github.com/tpope/vim-surround)插件。

假设你想要在C语言中打印`Hello, Vim!`,你可能会输入：
```
printf(Hello, Vim!);
```
这时候你突然发现你忘记加引号了，你只需要选中内容，并输入：
```
S"
```
此后Vim将会帮你在两边加上双引号。


## Vim插件

Vim作为一款强大的生产力，有大量的第三方插件，但我在这里只简单说一下如何安装其中一个插件--`YouCompleteMe`。

### 什么是`YouCompleteMe`
`YCM`时一款优秀的代码补全插件，支持语义补全。相比之下，Vim本身只支持简单的匹配，也就是说只要你之前没有输入过就无法补全。而且`YCM`还可以跳转函数定义和声明，极大地补充了Vim的不足。

### 如何安装`YouCompleteMe`

现在网上的教程大多数都是自己编译安装或者使用Vim插件管理器，不仅操作麻烦，而且很容易失败。
这里我推荐一个十分简单的安装办法:
```
sudo apt install -y vim-gtk3 vim-addon-manager vim-youcompleteme vim-python-jedi

vam install youcompleteme python-jedi
```

### 如何简单配置`YouCompleteMe`

按这种办法安装ycm的配置文件在`/usr/lib/ycmd/ycm_extra_conf.py`

我们可以修改其中的flags变量，Vim会在这些目录中搜索匹配。这主要在C/C++中寻找头文件时有效。

具体文件如下：
```
flags = [
'-Wall',
#'-Wextra',
'-Werror',
'-Wno-long-long',
'-Wno-variadic-macros',
'-fexceptions',
'-DYCM_EXPORT=',
'-x',
'c++', #表示默认为C++文件
'-isystem',
'/usr/include/c++/9', #在这些目录下寻找头文件
'/usr/local/include',
]

```

修改vimrc，使ycm支持语义匹配和变量跳转。
```
let g:ycm_global_ycm_extra_conf = '/usr/lib/ycmd/ycm_extra_conf.py'
"打开vim时不再询问是否加载ycm_extra_conf.py配置
let g:ycm_confirm_extra_conf = 1
set completeopt=longest,menu
"自动开启语义补全
let g:ycm_seed_identifiers_with_syntax = 1
"在注释中也开启补全
let g:ycm_complete_in_comments = 1
let g:ycm_collect_identifiers_from_comments_and_strings = 0
"字符串中也开启补全
let g:ycm_complete_in_strings = 1
let g:ycm_collect_identifiers_from_tags_files = 1
"开始补全的字符数
let g:ycm_min_num_of_chars_for_completion = 2
"补全后自动关闭预览窗口
let g:ycm_autoclose_preview_window_after_completion = 1
"禁止缓存匹配项,每次都重新生成匹配项
let g:ycm_cache_omnifunc=0
"离开插入模式后自动关闭预览窗口
autocmd InsertLeave * if pumvisible() == 0|pclose|endif
"语法关键字补全
let g:ycm_seed_identifiers_with_syntax = 1

"<leader>键在Vim中默认是反斜杠\
nnoremap <leader>gl :YcmCompleter GoToDeclaration<CR>
nnoremap <leader>gf :YcmCompleter GoToDefinition<CR> "跳转到函数定义，十分有用
nnoremap <leader>gg :YcmCompleter GoToDefinitionElseDeclaration<CR>
```
### 其它插件

其它的插件可以使用[vim-plug](https://github.com/junegunn/vim-plug)快捷安装

具体插件可按自己需求安装

## Vim的配置文件
```
set nu "显示行号
set relativenumber "显示相对行号
set ruler
set autoindent
set smartindent
set hlsearch "开启高亮搜索
syntax on "开启语法高亮
set textwidth=80

filetype indent on


"根据文件类型设置缩进
autocmd FileType php,python,c,java,perl,shell,bash,vim,ruby set ai
autocmd FileType php,python,c,java,perl,shell,bash,vim,ruby set sw=4
autocmd FileType php,python,c,java,perl,shell,bash,vim,ruby set ts=4
autocmd FileType php,python,c,java,perl,shell,bash,vim,ruby set sts=4
autocmd FileType javascript,cpp,html,css,xml set ai
autocmd FileType javascript,cpp,html,css,xml set sw=2
autocmd FileType javascript,cpp,html,css,xml set ts=2
autocmd FileType javascript,cpp,html,css,xml set sts=2
```
