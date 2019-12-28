---
title: 使用 Karabiner 映射快捷键
tags:
  - Mac
  - Karabiner
categories:
  - 杂项
date: 2016-04-04 14:31:29
---

用快捷键做某些操作是很方便的，很多软件也提供了快捷键，但是某些软件只提供了固定组合的快捷键，这些组合不一定合口味，如果不习惯，就可以用 [Karabiner](https://pqrs.org/osx/karabiner/index.html.en) 更改快捷键映射，把想用的快捷键映射到某些软件的快捷键上，下面以欧路词典为例，讲一下过程。

欧路词典只提供这三个快捷键任意组合

![](https://i.imgur.com/Tojkf4h.jpg)

这三个键用到的地方太多了，似乎怎么组合都容易冲突，而且我个人习惯来说，之前用 [BetterTouchTool](https://www.boastr.net/) 把 `Shift+Command+W` 映射到了 三指轻拍 这个动作来取词，所以我准备把 `Shift+Command+W` 映射到欧路词典中设置的 `Control+Commmand`上用来取词

<!-- more -->

首先打开 `private.xml`

![](https://i.imgur.com/FoTxgCs.jpg)

根据 Karabiner 文档中的语法 https://pqrs.org/osx/karabiner/xml.html.en#keytokey-syntax
和 Karabiner 中预设的按键代码列表 https://github.com/tekezo/Karabiner/blob/version_10.18.0/src/bridge/generator/keycode/data/KeyCode.data
编写如下代码
``` xml
<item>
    <name>Shift+Command+W -> Control+Option</name>
    <appendix>Shift+Command+W -> Control+Command 用于欧路词典鼠标取词</appendix>
    <identifier>com.cielpy.eudiclookup</identifier>
    <autogen>
      __KeyToKey__ 
	  KeyCode::W, ModifierFlag::COMMAND_L | ModifierFlag::SHIFT_L,
      KeyCode::CONTROL_L, ModifierFlag::COMMAND_L
    </autogen>
  </item>
```
`__KeyToKey__` 下面第一行就是映射后的快捷键，第二行是映射到的快捷键，也就是用第一行的组合代替第二行的组合。
把以上代码放到 `<root>` 节点下，这样 `private.xml` 就编写完成了，保存 `private.xml` 后，返回 `Karabiner` 设置的第一页，点击 `Reload XML` 即可。

![](https://i.imgur.com/RgBhECQ.jpg)


Karabiner 还提供了很多组合方法，单一按键映射，组合键映射到单一键，组合键映射到组合键等，有需求的话可以研究一下。

另外 Karabiner 是一个开源软件，如此神器，如果觉得有用可以适当捐赠一下哈 :)

参考：
1. https://pqrs.org/osx/karabiner/xml.html.en
2. https://github.com/tekezo/Karabiner/blob/version_10.18.0/src/bridge/generator/keycode/data/KeyCode.data
3. http://www.jianshu.com/p/30a95c0727a9
