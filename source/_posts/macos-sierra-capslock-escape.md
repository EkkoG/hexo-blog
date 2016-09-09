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
>![](https://ws3.sinaimg.cn/large/74681984gw1f7nl62c671j210m0tewkm.jpg)
>Credit to this github comment: https://github.com/tekezo/Seil/issues/68#issuecomment-230131664

先在系统偏好设置-键盘-修饰键里，修改 CapsLock 键为 Control 键。

![](https://ws3.sinaimg.cn/large/74681984gw1f7nkuubdytj20if0b3401.jpg)

再使用 Keyboard Maestro 设置如下：

![](https://ws3.sinaimg.cn/large/74681984gw1f7nkxb7nh7j20ef0brabh.jpg)‘

注意这里 CapsLock 后面用了 tapped，而不是 Stackoverflow 中的 pressed，使用 pressed  的话，只要按了 CapsLock 键就会响应 Escape 事件，但以我使用 Karabiner 的习惯来讲，我只希望在只按 CapsLock 键并松手的时候才响应 Escape 键，这样的效果只有使用 tapped 可以达到，如果按住 CapsLock 并组合其他键使用，并不会响应 CapsLock 的 tapped 事件，这样就不会响应 Escape，然后此时的作用就相当于 Control 键了，可以使用 CapsLock + F 前移光标，CapsLock + B 后退光标等等，和使用 Karabiner 时效果一样。

不过使用的这个软件是付费的，在 Karabiner 适配 macOS Sierra 前就先用这个替代了。

完。

## 参考资料

1. [Using Caps Lock as Esc in Mac OS X](https://stackoverflow.com/questions/127591/using-CapsLock-as-esc-in-mac-os-x)
2. [macOS (10.12) compatibility](https://github.com/tekezo/Karabiner/issues/660)

--EOF--

