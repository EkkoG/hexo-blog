---
title: 几个 Xcode 调试技巧
tags:
  - Xcode
  - 调试
categories:
  - 编程
date: 2016-06-10 12:54:06
---

### 给断点加上声音

有些断点不是经常触发，可以加上声音，这触发的时候会发出声音提醒一下。

![](https://www.bignerdranch.com/img/blog/2013/11/edit-breakpoint-menu.png)

![](https://www.bignerdranch.com/img/blog/2013/11/edit-breakpoint-window.png)

Action 改为 sound

![](https://www.bignerdranch.com/img/blog/2013/11/breakpoint-actions.png)

"Automatically continue after evaluation" 的意思是触发断点的时候继续执行。

### 断点的时候执行命令

这个玩法就多了，可以执行 AppleScript, Shell 等，还可以跟参数。例如调用 `say` 命令说一句话。

![](https://ww2.sinaimg.cn/large/74681984gw1f2ubn7j61kj20ci06y3z8.jpg)

### LLDB 命令

也可以在断点的时候自动执行调试命令，如 `po` 等，`bt` 命令是 `backtrace` 意思是回溯栈？
配合 "Automatically Continue" 可以在触发某个断点的时候打印当时的栈的情况，并且不影响程序继续执行。`bt` 可以加参数如 `bt 10` 就是打印最近10条记录。

** 以上三条是每个断点都可以添加的特性，并且可以配合使用 **

### 符号断点

当想 debug 一些三方库或者系统库的时候，看不到代码没办法直接加断点，可以通过符号断点来断点。

"Add Symbolic Breakpoint..."

![](https://www.bignerdranch.com/img/blog/2013/11/add-breakpoint-menu.png)


![](https://www.bignerdranch.com/img/blog/2013/11/UIViewController-viewDidLoad-symbolic-breakpoint.png)

这样每个 UIViewController 对象的 `viewDidLoad` 方法执行的时候都会自动断点。类似的，可以给别的看不到代码的方法加上断点。

### Watchpoints（数据断点）

Watchpoints 可以监控变量的值，或者内存地址的变化。

如果一些奇怪的问题，某个变量在有些时候变成奇怪的值而这个变量牵扯的逻辑很复杂，就可以用 `Watchpoints` 在这个变量的值改变的时候触发断点进行调试。也可以加个声音啊什么的。

在打断点，触发断点后，找到想要监控的变量。右键可以看到 `Watch xxx`

![](https://www.bignerdranch.com/img/blog/2013/11/set-watchpoint-menu.png)

点击就可以看到在断点列表里多了一个断点。当然也可以加个声音什么的提醒一下。

每次运行完成后，数据断点不会被保存。

