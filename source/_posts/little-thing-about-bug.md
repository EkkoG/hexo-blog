---
title: 一个 Bug，一件小事
tags:
  - Bug
  - 感悟
categories:
  - 编程
date: 2016-06-15 00:19:19
---

前两天同事想在项目中集成 [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)，选择了使用 [CocoaPods](https://cocoapods.org/) 安装，结果编辑 Podfile 并运行 `pod install` 命令安装后，无法编译运行，报错信息如下

```
/Users/xx/Desktop/code/PodTest/Pods/ReactiveCocoa/ReactiveCocoaFramework/ReactiveCocoa/RACTuple.h:10:9: 'metamacros.h' file not found
```

查了一些资料没能解决后问我，我看了一下发现这个文件是存在的，但是报的找不到文件的错误，并在目录中找到了两个一模一样的文件，另一个存在于 [Mantle](https://github.com/Mantle/Mantle) 中，为什么两个库会使用同一个文件呢，再查，其实这个文件是 [libextobjc](https://github.com/jspahrsummers/libextobjc) 库中的一个文件，RAC 和Mantle 都用到了这个库中的一些特性就都集成到了内部，但是同时安装两个库时就出现了冲突。然后我 Google 了这个文件，相信遇到这个问题的我们肯定不是第一个，当我点开一个链接的时候，越看越不对劲，这提问者不是我嘛。。。
https://www.v2ex.com/t/155591
贴子中并没有解决方案，然后回忆了一下，因为这个问题，搞了好几个小时，最终我选择了重装系统（。。。真的重装了系统，尤记得当时让同学在他的电脑上装了同样版本的 RAC，他那边是可以正常安装运行的，所以我各种重装，包括 ruby, CocoaPods 都重装了，还是不行，到 CocoaPods 的 spec 仓库的 issue 区提问，作者回复不是 CocoaPods 的问题就 close 了，现在 spec 仓库已经关了 issue 区。各种解决方案不行，最终下了结论，系统问题，重装！其实问题在哪呢，我去了 RAC 的 issue 区，查到了这个 issue
https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2909
然后看到了问题所在，因为2.x版本的 RAC 的 podspec 文件中有一个 prepare_command，如下

```
"prepare_command": "    find . \\( -regex '.*EXT.*\\.[mh]$' -o -regex '.*metamacros\\.[mh]$' \\) -execdir mv {} RAC{} \\;\n    find . -regex '.*\\.[hm]' -exec sed -i '' -E 's@\"
```

该命令将 RAC 中对 `metamacros.h` 的引用加上了前缀并重命名了 `metamacros.h` 为 `RACmetamacros.h`，这样就不会跟其他库冲突，用 issue 中自定义的 podspec 文件安装 RAC 后成功解决。

由于当时的环境已经不存在，不可复现，猜测是对 prepare_command 的执行出了点问题吧。

因为一个 bug 重装了系统，要不要因为同事遇到了相同的问题就把这事给忘了，如果要因为一个问题解决不了要重装系统，要三思啊。。。(

--EOF--


