---
title: Git 父提交
tags:
  - Git
  - 版本控件
categories:
  - 编程
date: 2016-09-09 23:19:25
---

使用 Git 的时候，如果需要重置本地目录到最新一次提交，一般会使用以下命令：

```
git reset --hard HEAD
```

一直以来我都以为 `HEAD` 是最新一次提交，那么 `HEAD^` 就是倒数第二次提交，和 `HEAD^` 类似的 `HEAD~1` 也是最近一次提交，`HEAD~2` 是倒数第二次提交，这样理解看起来似乎没有问题，因为一次次的使用证明 `HEAD^` 就是倒数第二次提交，直到在一个比较复杂的项目里想打印最近 10 条提交的 log，使用以下命令：

<!-- more -->

```
git log --pretty=format:'%s' HEAD~10..HEAD
```

结果出现了不止 10 条，才开始怀疑是不是理解错了。

可能因为之前操作的项目不复杂，大多没有很多合并操作，如果有，可能早就能发现 `HEAD~2` 不是倒数第二次提交，也就没有上面想用 `git log --pretty=format:'%s' HEAD~10..HEAD` 打印最近 10 条 log 的天真想法。

发现不对后，查了下资料，HEAD 并不是那么简单的指向最后一次提交那么简单，在 Git 的版本管理中，每一次提交 Git 会保存一个提交对象（commit object），而每一次提交产生的提交对象都有一个父对象，即本次提交的上次提交。这是 Git 的数据存储方式。简单了解到这里，详细的请移步至 [1.1 起步 - 关于版本控制](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%85%B3%E4%BA%8E%E7%89%88%E6%9C%AC%E6%8E%A7%E5%88%B6#_getting_started)

Git 中有一个特殊的指针 `HEAD`，它指向当前所在的本地分支（也可以认为是本地当前分支的别名），如果所有的提交都在同一个分支（假设是 master）产生，HEAD 应该指向 master 分支，如图所示：

![](https://i.imgur.com/vbGwPoI.jpg)
图1

`HEAD^` 指向其父提交，`HEAD^^` 指向其父提交的父提交即祖父提交，`^` 后面可以跟数字，`HEAD^1` 意为第一父提交，`HEAD^2` 为第二父提交，这个语法只有在合并分支时产生的提交中有效，合并分支时产生的提交有两个父提交，第一父提交为合并前当前分支的最后一次提交，第二父提交为被合并的分支的最后一次提交。`HEAD~` 形式中 `~` 后面也可以跟数字，`HEAD~1` 为父提交，`HEAD~2` 为父提交的父提交即祖父提交。参见：[祖先引用](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%80%89%E6%8B%A9%E4%BF%AE%E8%AE%A2%E7%89%88%E6%9C%AC#祖先引用)

如果用以下命令看最新一次提交的更改内容，是可以达到想要的效果的：

```
git show HEAD
```

如果在 HEAD 后面加个 `^`，命令如下：

```
git show HEAD^
```

同样可以看到倒数第二条的修改内容。也可以加多个 `^`，加几个就是倒数第几条，都没有问题。

同样的，使用这样的命令查看倒数第 N 条的 更改内容，也是没有问题的：

```
git show HEAD~1
git show HEAD~2
git show HEAD~3
...
```

一切看起来都很正常。在这样的一个仓库中，任何一次提交都是没有第二父提交的，因为没有合并产生。

如果有分支合并情况，假设目前的分支状态如下：

![](https://i.imgur.com/Hix9aQs.jpg)
图 2

在 qwq3 的时候有分支 test 产生，于 db56 提交后，合并入 master 分支并产生提交 ad0b。

此时 `git show HEAD` 会显示最近一次提交及合并产生的那一次提交所修改的内容，这个没什么疑问，如果使用 `git show HEAD^` 呢，是会显示 20af 还是 db56 的修改内容？

根据 Git Pro 中的说明，HEAD 目前指向 master 分支，其第一父提交应为合并前的 db56，第二父提交为 20af。而 `HEAD~1` 形式的只会拿第一父提交往前追溯，那么基于这样的机制，使用 `git show HEAD~10` 是不一定出现倒数第 10 次的修改内容的，因为这样的追溯是会拐弯的。


如图 2 中的分支形式，`HEAD~3` 是指向 fc40 的，而如果按照文章开头的方法打印最近 3 条的日志的话，在 `HEAD~3` 到 `HEAD` 之间还有一个 20af 提交，所以并不止 3 条日志出现。

如果想打印最近 10 次提交的日志，正确的方法应该用如下命令：

```
git log -10
```

这样会按照提交时间顺序打印最近的 10 条提交的日志。

## 参考资料

1. [7.1 Git 工具 - 选择修订版本](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%80%89%E6%8B%A9%E4%BF%AE%E8%AE%A2%E7%89%88%E6%9C%AC#祖先引用)
2. [3.1 Git 分支 - 分支简介](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%AE%80%E4%BB%8B)
3. [What's the difference between HEAD^ and HEAD~ in Git?](https://stackoverflow.com/questions/2221658/whats-the-difference-between-head-and-head-in-git)

