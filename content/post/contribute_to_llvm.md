---
title: "如何给LLVM贡献代码"
author: "Jun"
date: 2021-12-11T20:50:50+08:00
---

## LLVM 简介

LLVM的代码托管在[Github](https://github.com/llvm/llvm-project)，截止到这篇博客
发出时，其Bug tracker 已经从原来的[Bugzilla](https://bugs.llvm.org/) 成功迁移到了[Github issue](https://github.com/llvm/llvm-project/issues)上，
具体可以看这封[邮件](https://lists.llvm.org/pipermail/cfe-dev/2021-December/069580.html)。但是LLVM暂时不接受，或者说主项目不接受Github PR, 而是使用[Phabricator](https://reviews.llvm.org/)。



## 编译安装
```bash
git clone https://github.com/llvm/llvm-project.git
```



如果不希望保留以前的Git历史的话，可以使用：

```bash
git clone --depth=1 https://github.com/llvm/llvm-project.git
```
这会极大减少下载时间和硬盘占用。



### 构建工具

因为LLVM是使用C++14标准写的（实际上使用了很多C++17特性），所以我们要保证我们的编译器支持其标准，以及有其它配套的构建工具：
- ninja
- cmake
- gcc/clang/MSVC
- ccache
- gold


```bash
cd llvm-project
mkdir build && cd build
```

```bash
cmake -DCMAKE_BUILD_TYPE=Release  \
      -DLLVM_TARGETS_TO_BUILD=X86 \
	  -DLLVM_USE_LINKER=gold      \
      -DBUILD_SHARED_LIBS=ON      \
      -DLLVM_CCACHE_BUILD=ON      \
	  -DLLVM_APPEND_VC_REV=OFF     \
	  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" \
      -G"Ninja" ../llvm
```

可以看到在cmake配置中我们使用了大量额外选项，这主要是为了减少编译时间和硬盘占用：
- `DLLVM_USE_LINKER=gold` 使用gold而不是默认的ld，这可以减少链接时的内存消耗
- `BUILD_SHARED_LIBS=ON` 默认LLVM会使用静态链接，这会导致非常夸张的硬盘消耗，个人建议
  使用shared library。
- `DLLVM_CCACHE_BUILD=ON` 使用ccache， 这样当我们修改代码重新编译后只会编译修改的部分，
  便免消耗时间重新编译未更改的代码。
- `Ninja` 可以比`make`更快地编译。
- `LLVM_APPEND_VC_REV=OFF` 避免git pull后无意义的重新链接
- `LLVM_ENABLE_PROJECTS`可以选择启用的子项目，我们这里只编译`clang`和`clang-tools-extra`。


```bash
ninja
```

## 提交代码

前面说到LLVM使用的是Phabricator，而不是Github PR。如果你需要在[Phabricator](https://reviews.llvm.org/auth/start/?next=%2F)注册一个账号，可以直接将Github账号绑定在一起。

在命令行中提交代码个人认为比较简单，只要使用官方提供的工具arcanist即可。当然使用Web界面也是可以的，但是要麻烦的多，具体可以看LLVM官方的[文档](https://llvm.org/docs/Phabricator.html#requesting-a-review-via-the-web-interface)。

下载安装：
```bash
git clone https://github.com/phacility/arcanist.git
cd arcanist
git fetch https://github.com/rashkov/arcanist update_cacerts
```

将其添加到环境变量中：
```
export PATH="$PATH:/somewhere/arcanist/bin/"
```

授权登陆：
```
arc install-certificate
```

在写代码的时候可以使用`git commit --amend`将多个提交压缩成一个，从而以patch的形式发出去。
举个例子：
```bash
git checkout -b patch
# make some changes
git add .
git commit
# make more changes
git commit --amend -a
```

当我们的patch在本地通过测试后，便可以准备发出去了,但在此之前一定要注意用`clang-format`保证我们的
patch符合LLVM Coding Style。

建议使用LLVM官方提供的`git-clang-format`工具，效果最好，避免意外修改到其他的代码。
可以直接将[git-clang-format](https://github.com/llvm/llvm-project/blob/main/clang/tools/clang-format/git-clang-format)复制一份到PATH所在的路径中就可以了，比如`/usr/local/bin`。


```bash
git clang-format HEAD~1
```

注意format后并不会自动提交，所以你需要：
```bash
git commit --amend -a
```

### 寻找Reviewer

arcanist提供了一个选项帮助我们寻找潜在的Reviewer:
```bash
arc cover
```

这会在终端中显示同样修改过最近一个commit中的文件的所有人。注意显示的只是修改者的人名，
而我们需要的是他们在Phabricator中的账号名。此后：

```bash
arc diff --reviewers=account1,account2
```
接着你便需要写一个title和summary，这个之后会作为commit message。成功后arcanist会给我们
一个链接，这便是我们的Revision啦。


### 更新patch

再Reviewer看过代码后，可能会给我们一些修改的建议，我们可以继续使用arcanist更新我们的patch:
```bash
# make some changes
git commit --amend -a
```

其中`DXXXXX`是我们之前的Revision，因为arcanist无法知道上下文，所以我们必须手动指定。
```
arc diff `git merge-base HEAD origin` --update DXXXXX
```

## 社区支持

LLVM有一个非常活跃，非常热情的社区。如果遇到了任何问题，完全可以寻找社区的帮助，那里有大量的志愿者非常乐意回答你的问题。当然，和所有成熟的开源项目一样，LLVM有着非常丰富的文档，善于从文档中寻找解决答案也是一项重要的技能。

- [邮件列表](https://lists.llvm.org/mailman/listinfo)
- [IRC](irc://irc.oftc.net/llvm)
- [Discourse](https://llvm.discourse.group/)
- [Discord](https://discord.gg/xS7Z362)

Happy Hacking :^)
