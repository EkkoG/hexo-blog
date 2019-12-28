---
title: 升级 macOS Sierra 后 Vim 和 Tmux 异常的解决办法
tags:
  - macOS
  - Vim
  - Tmux
  - 共享剪切板
categories:
  - 杂顶
date: 2016-09-11 01:13:48
---

尝鲜的代价大概就是要先踩坑，苹果在 9 月 7 日的发布会后，发布了包括 macOS，iOS 和 tvOS 在 GM 版和候选发布版，这种版本在没有大 bug 的情况下，一般就是正式版了，也就是足够稳定了，就算不稳定也不会和正式版差很多，然后我就升级了，升级后一些软件不能正常工作，这里记录一下。

首先主要是键盘映射类的，因为 Sierra 上对键盘和鼠标的驱动发动较大，导致了与其相关的软件基本都不能正常工作，这个在之前的文章中有提到，见 [在 macOS Sierra 上替换 CapsLock 键为 Escape 键](https://imciel.com/2016/09/09/macos-sierra-capslock-escape/) 好在找到了解决办法。

<!-- more -->

然后发现，之前设置的 Vim 和系统共享剪切板不能工作，并在 V2EX 上发现了同样情况的网友：
[macOS Sierra 下是不是 vim 不能再和系统共享剪贴板了？](https://www.v2ex.com/t/305371)

之后发现在退出 Tmux 后，在终端使用 Vim，复制后，内容是和系统剪切板共享的，问题可能在 Tmux 身上，然后查到了另个一外贴子，[os x 下 vim 无法复制到系统剪切板的问题](https://www.v2ex.com/t/96300) 中提到：

>Mac + tmux + Vim 用户的解决方案在这里： 
>https://coderwall.com/p/j9wnfw

根据该贴子中的提示，使用 brew 安装了另个一个工具：

```
brew install reattach-to-user-namespace
```

后在 `.tmux.conf` 中添加一行配置：

```
set-option -g default-command "reattach-to-user-namespace -l bash"
```

设置这些后，在 Tmux 中开启 Vim，复制内容也可以与系统剪切板共享了。

其中 Tmux 的那一行配置在我之前的配置里也有，在 OS X EL Capitan 中没有异常，比较奇怪，可能之前系统中的工作方式不同吧。

解决了 Tmux 中 Vim 的剪切板问题，该解决 Tmux 本身的剪切板问题了。

在 Tmux 中不能直接用鼠标选中复制，我使用了 `y` 来复制内容到系统剪切板：

```
bind -t vi-copy y copy-pipe 'pbcopy'
```

但是这个也不工作了。在 Stackoverflow 找到了一个解决方案：
[On MacOS Sierra beta 5 using iterm 2 and tmux, I have lost the ability to copy/paste in tmux.](https://superuser.com/questions/1114694/on-macos-sierra-beta-5-using-iterm-2-and-tmux-i-have-lost-the-ability-to-copy-p/1114729) ，在 Iterm 2 中，需要打开一个选项，使应用可以访问剪切板：

![](https://i.imgur.com/ujbAjO4.jpg)

勾上这个选项后，在 Tmux 中可以正常使用 `y` 来访问剪切板，复制内容到系统的剪切板中。

最后一个问题，在暂时退出 Tmux 会话时，会报一个警告：

![](https://i.imgur.com/a1V8Xp4.jpg)

查到 Tmux 中一个组件（姑且这么叫吧）kqueue 在 macOS Sierra 上不工作了，在 Tmux 最新的更改中已经默认关掉了这个功能，可以使用 brew 安装最新的版本解决：

```
brew uninstall --force tmux
brew install --HEAD tmux
```

或者使用环境变量暂时关闭这个功能：

```
export EVENT_NOKQUEUE=1
```

到目前为止异常情况都可以解决。有新的话再总结。

参考资料：

1. [在 macOS Sierra 上替换 CapsLock 键为 Escape 键](https://imciel.com/2016/09/09/macos-sierra-capslock-escape/) 
2. [macOS Sierra 下是不是 vim 不能再和系统共享剪贴板了？](https://www.v2ex.com/t/305371)
3. [os x 下 vim 无法复制到系统剪切板的问题](https://www.v2ex.com/t/96300)
4. [Vim & Tmux & System Clipboard](https://coderwall.com/p/j9wnfw/vim-tmux-system-clipboard)
5. [macos sierra: [warn]: kq_init: detected broken kqueue; not using.: File exists ](https://github.com/tmux/tmux/issues/475)




