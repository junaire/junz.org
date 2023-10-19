---
title : "Git学习笔记"
author : "Jun"
date : "2021-02-28T15:38:15+08:00"
---

本文历经几次修改，主要是我对Git的一些学习和理解。其内容主要参考[Pro Git](https://git-scm.com/book/en/v2)。
在经过几次重写后，我决定以问答和解释相结合的形式来总结与Git相关的知识点。

## git的基本工作模型
```
+--------------+      +--------------+      +----------+
|              |      |              |      |          |
| Working area | ===> | Staging area | ===> | Git repo |
|              |      |              |      |          |
+--------------+      +--------------+      +----------+
```

- **Working area** 所有被Git track过的文件都认为被放到这个区域。
- **Staging area** 这里负责存放准备提交的文件。
- **Git repo** 这里就是提交后的文件了。

## git分支
```
+-----+    +-----+    +-----+ <---- master   HEAD
|123a |<---|1e69 |<---|5adc | ^               |
+-----+    +-----+    +-----+ |_______________+
```

在Git中，分支可以认为是指向一个commit的指针，其中`HEAD`永远指向当前所在的commit。

## 怎么将文件加入staging area?
如果是提交单个文件, 可以使用`git add <filename>`，如果是提交多个文件，可以使用`git add .`或
`git add *.c`等。如果想将一个文件中的多个修改分开提交，可以使用`git add -p`。

## 怎么将加入staging area的文件退回工作区？
如果错误使用`git add .`将一些不想要的文件加入了staging area, 可以使用`git restore --staged <filename>`
将文件退回工作区，并保留了更改。

## 怎么撤销当前的提交并保留修改？
假如我们刚写好一个patch并想要写下一个，但是忘记切回主分支而是直接切了一个新分支，这就导致了新分支的base branch错了。我们可以先撤销掉当前的提交：
```
git reset HEAD^
```
接着我们再切一个新的分支提交就好了：
```
git checkout -b new-branch
git commit -a
```

## 怎么修改之前的commit？
假如你对上一个commit不满意，比如少提交了文件，写错了commit message等，你可以：
```
git commit --amend
```
### 那怎么修改非上一个commit呢？
如果你想修改非上一个commit的话，就需要使用rebase了。

假设我们有如下的git历史：
```
+----------+
| commit 3 |
+----------+
     ||
	 \/
+----------+
| commit 2 |
+----------+
     ||
	 \/
+----------+
| commit 1 |
+----------+
```
我们想修改`commit 2`的提交信息的话，可以：
```
git rebase -i HEAD~3
```
接着找到我们想修改的提交，将其`pick`改成`edit`，并保存退出。

这时候我们就可以使用`git commit --amend`来修改了，最后`git rebase --continue`就可以了。

## 怎么将放弃工作区的修改？
对于处于工作区的修改，可以使用`git restore <filename>` 丢弃修改。

**注意此操作不能撤销，修改的文件将直接丢失！**

## 怎么查看git历史？
```
git log #默认的查看历史
git log --pretty=oneline #以单行的形式查看历史
git log --author="Jun Zhang" #搜索指定作者的commits
```

## 如何将多个commits合并成一个？
假设我们有git历史如下：
```
commit 4ab5de2355dc7f910ed39d1b71052fc1d30a0da3 (HEAD -> master)
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:27:02 2021 +0800

    finally fix up

commit 2d187de6e426e754084663a1710de9fd7a87c871
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:26:25 2021 +0800

    another fix up

commit ca0fe9c9c1f116cf83905aaacc0736359af3fb7e
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:26:03 2021 +0800

    fix up

commit 418809d837801cc8b2816567b373747599730736
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:25:41 2021 +0800

    Initial commit
```

我们想把前三个commit合并成一个，因为他们是相关联的。我们可以这么做：
```
git rebase -i HEAD~3
```

git会展示如下内容：
```
pick ca0fe9c fix up
pick 2d187de another fix up
pick 4ab5de2 finally fix up
# Rebase 418809d..4ab5de2 onto 418809d (3 commands)
```
可以看到这里的历史与git log展示的历史顺序刚好相反。我们要把下面两个后来的commits
合并到之前的那一个上，我们可以：

```
pick ca0fe9c fix up
squash 2d187de another fix up
squash 4ab5de2 finally fix up
```
保存后git便会要求我们重写git commit messge，此后使用git log可以看到3个commits被合并成了一个。
```
commit fedae1c1c4265c314166729c16a2855dfddc1f3c (HEAD -> master)
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:26:03 2021 +0800

    squash 3 commits into one.

commit 418809d837801cc8b2816567b373747599730736
Author: Jun Zhang <jun@junz.org>
Date:   Fri Dec 24 22:25:41 2021 +0800

    Initial commit
```

## 怎么在切换分支的时候暂时保存工作进度？
有时候在一个分支还没有工作完，我们并不想commit，但是因为一些原因我们又需要切一个新的分支，
这时候便可以使用`git stash`这个命令。

比如我们现在在分支`dev`上，我们有一些在staging area的文件，而我们并不想立刻commit。我们可以
使用`git stash`暂存更改。`git stash`会把更改放到一个栈型的结构中，后面我们可以继续使用`git stash`
暂存更改。

使用`git stash list`来查看所有暂存的更改，据个例子：
```
stash@{0}: WIP on master: fedae1c squash 3 commits into one.
stash@{1}: WIP on dev: fedae1c squash 3 commits into one.
```
我们分别在dev分支和master分支上使用了`git stash`，可以看到我们有两个暂存的更改。

当我们准备好时，可以使用`git stash apply stash@{n}`来释放之前暂存的修改。如果直接使用`git stash apply`，
会应用栈顶的那个更改。但是注意应用后在栈中存放的更改并不会丢失，而是继续存放在那除非手动使用`git stash drop stash@{n}`。
也可以使用`git stash pop`应用栈顶的那个更改并自动将其`drop`掉。

## 怎么合并分支？
合并分支有两种策略，一种是默认的`git merge`，另一种是`git rebase`。
```
                      +-----+
              +-------|9e7d |(dev)
              |       +-----+
             \/
+-----+    +-----+    +-----+
|123a |<---|1e69 |<---|5adc |(master)
+-----+    +-----+    +-----+
```

对于merge来说，其工作原理是由其公共commit后diff创建一个新的commit。即它会合并`9e7d`和`5adc`，并
加到`5adc`上。

而对于rebase来说，其工作原理是将新的commit移动到其他分支上，像是改变了其parent。
```
git checkout -b dev
# make some changes
git rebase master
git checkout master
git merge dev
```
如果说在开发过程中远程master分支有了新的commit，如上图的`5adc`，我们应该先：
```
git checkout master
git pull
```
在此之后再使用`rebase`。
