---
title: 在 macOS Sierra 上替换 CapsLock 键为 Escape 键
tags:
  - 键盘映射
categories:
  - 杂项
date: 2016-09-09 19:40:19
---

根据 Karabiner 作者的描述，macOS Sierra 对键盘和鼠标的驱动的修改和 Karabiner 有冲突，所以现在 karabiner 在 macOS Sierra 上不能工作了。

>macOS 10.12 changes of the generic keyboard and mouse drivers made a great impact on Karabiner and Seil.
We should make a large changes in Karabiner and Seil architecture.
There is not a workaround for this issue.

>Please wait an update of Karabiner and Seil for macOS 10.12.
(It may take a long time.)

引用自：https://github.com/tekezo/Karabiner/issues/660#issuecomment-226942420

作为一人每天使用这个软件的我简直是晴天霹雳啊，现在对键盘的使用习惯是把 CapsLock 键映射成 Control 键，并在只按左边的 Control 键时，等同于 Escape 键，这样用起来比较顺手，对于使用 Vim 较多的人来说，使用 CapsLock 替换掉 Escape 可以省去左手在键盘上跑来跑去。作者说要适配 macOS 10.12 要做的修改很大，已经在进行中了，新开了一项目：https://github.com/tekezo/Karabiner-Elements ，作者建议现在不要用在自己的电脑上，所以，先找替代方案吧。

经过一番搜索，在 Stackoverflow 上找到一个方案：

>Seil isn't yet available on macOS Sierra (10.12 beta). As such, I've been using Keyboard Maestro with these settings: 
>![](http://i.stack.imgur.com/cW1RK.png)
>Credit to this github comment: https://github.com/tekezo/Seil/issues/68#issuecomment-230131664

不过使用的这个软件是付费的，在 Karabiner 适配 macOS Sierra 前就先用这个替代了。

完。

## 参考资料

1. [Using Caps Lock as Esc in Mac OS X](https://stackoverflow.com/questions/127591/using-CapsLock-as-esc-in-mac-os-x)
2. [macOS (10.12) compatibility](https://github.com/tekezo/Karabiner/issues/660)

--EOF--

