---
title: Xcode 合并冲突后 Discard All Changes 导致代码丢失问题
tags:
  - iOS
  - Xcode
  - Git
categories:
  - 编程
date: 2017-10-13 17:02:57
---

使用 Git 合并时常会产生冲突，某些情况下我们想丢弃此次合并，通常会使用 `git reset --hard HEAD` 命令来重置，其实 `git reset --hard HEAD` 命令在此时不光重置了暂存区，还结束了丢弃了这次合并，和 Xcode 的 Discard All changes 行为和 `git reset --hard HEAD` 并不完全一致，只是重置暂存区而没有结束 merge，接下来的 commit 将自动完成这次 merge，因为同时重置了暂存区，最终造成代码丢失。下面用一个 demo 来重现和解释这个现象。

<!-- more -->

demo 如下：


```
$ ls
CPYMergeDemo/  CPYMergeDemo.xcodeproj/  CPYMergeDemoUITests/
```

在 master 分支上，修改 ViewController.m 一行代码，并进行一次 commit

```
$ gc -a -m "test 0"
[master 9a38a47] test 0
 1 file changed, 1 insertion(+)
```

切换到 test 分支，修改同一行代码，这样合并的时候就会产生冲突，进行一次提交，如下：

```
$ gc -a -m "test 1"
[test 0c21bb9] test 1
 1 file changed, 1 insertion(+)
 
```

在另一个方法中添加一行，再进行一次提交，如下：

```
$ gc -a -m "test 2"
[test 54d3bd2] test 2
 1 file changed, 1 insertion(+), 1 deletion(-)
```

把 test 分支合并到 master 分支，产生冲突，如下：

```
$ git merge test
Auto-merging CPYMergeDemo/ViewController.m
CONFLICT (content): Merge conflict in CPYMergeDemo/ViewController.m
Automatic merge failed; fix conflicts and then commit the result.
```

ViewController.m 文件如下：

```
//
//  ViewController.m
//  CPYMergeDemo
//
//  Created by ciel on 2017/10/14.
//  Copyright © 2017年 CPY. All rights reserved.
//

#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
<<<<<<< HEAD
    NSLog(@"0");
=======
    NSLog(@"1");
>>>>>>> test
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
    NSLog(@"2");
}


@end
```

合并产生冲突后，会在 `.git` 目录产生几个记录此次 merge 相关的标记文件，如下：

```
$ ls .git
COMMIT_EDITMSG  MERGE_HEAD  MERGE_MSG  config       hooks/  info/  objects/
HEAD            MERGE_MODE  ORIG_HEAD  description  index   logs/  refs/
```

暂存区的情况变成如下：

```
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)

	both modified:   CPYMergeDemo/ViewController.m

no changes added to commit (use "git add" and/or "git commit -a")
```

若想丢弃此次合并，可以使用命令重置，回到当前分支的 HEAD，命令如下：


```
git reset --hard HEAD
```

此时 git 状态回到合并前，如下：

```
$ ls .git
COMMIT_EDITMSG  HEAD  config  description  hooks/  index  info/  logs/  objects/  refs/
```

暂存区里没有更改：

```
$ git status
On branch master
nothing to commit, working tree clean
```

但是使用 Xcode 的 Source Control 中的 Discard All Changes... 操作重置后，暂存区变成如下所示，显示出有一个 merge 正在进行。

```
$ git status
On branch test_1
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)
```

git 的状态还保留了合并未完成的状态，如下：

```
$ ls .git
COMMIT_EDITMSG  MERGE_HEAD  MERGE_MSG  config       hooks/  info/  objects/
HEAD            MERGE_MODE  ORIG_HEAD  description  index   logs/  refs/
```

MERGE_HEAD 中记录了要合并的 commit 信息，如下：

```
$ cat .git/MERGE_HEAD
6b9d187115f0551691cd06d77d3c8f8ed996c278
```

这是一个很隐蔽的状态，不特别注意几乎不可能注意到这个状态，此时若我们进行修改，暂存区变成这样：

```
$ git status
On branch master
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   CPYMergeDemo/ViewController.m
```

进行提交：

```
$ gc -a -m "test 3"
[master 84e8302] test 3
```

由于 merge 标识没有删除，此次提交，相当于完成了此次合并，同时由于暂存区被重置，所有代码回到当前分支的最新的一次修改，而由于 MERGE_HEAD 标记的存在，提交的同时也会完成这次 merge，把被合并分支中有而当前分支没有 commit 记录合并，最终 git log 会成这样：

```
*   84e8302 - (39 seconds ago) test 3 — cielpy (HEAD -> master)
|\
| * 6b9d187 - (21 minutes ago) test 2 — cielpy (test)
| * 20dfa6d - (21 minutes ago) test 1 — cielpy
* | 9a38a47 - (24 minutes ago) test 0 — cielpy
|/
* 14bb742 - (25 minutes ago) Initial Commit — cielpy
```

commit 6b9d187 中在 `didReceiveMemoryWarning` 中方法中添加了一行代码，这条 commit 出现在了历史记录中，而现在 ViewController.m 文件的内容是这样的：

```
//
//  ViewController.m
//  CPYMergeDemo
//
//  Created by ciel on 2017/10/14.
//  Copyright © 2017年 CPY. All rights reserved.
//

#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    NSLog(@"0");
    NSLog(@"1");
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}


@end
```

6b9d187 的修改「丢失」了。

在一般情况下，Xcode 的 Discard All Changes... 操作重置暂存区，重置后再修改，再提交，都是没有问题的，但是在有冲突的情况下使用 Discard All Changes...，就埋下了一个很隐蔽的坑，不知不觉中，就出现了明明 commit 已经被合并过来，但是相关代码却没有出现的奇怪问题。

总结：

慎用 Discard All Changes... 操作！


Demo 地址：https://github.com/cielpy/CPYMergeDemo

