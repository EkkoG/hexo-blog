---
title: 使用 build configuration 管理环境切换和发布内部测试
tags:
  - iOS
  - 环境管理
categories:
  - 编程
date: 2018-01-28 21:23:36
---

开发中时常遇到一个问题，多环境切换，怎么做比较优雅？

### 代码中使用宏管理

通常情况，不同环境的区别仅仅是一个 base URL，或者再加上一些三方的 appkey，曾经在有限的认知里只知道可以通过宏变量来切换环境，需要切换环境时修改变量值，这样一个代码库一个时间点只能连接到一个环境，不改代码切环境？抱歉，不能。所以遇到上线，经常需要频繁的切环境来 debug，毕竟有些东西不可能直接上生产环境。修改代码并不麻烦，一行代码而已，问题出现切换环境的人身上，切几次就不知道当前所在的环境了，一不小心就将测试环境提交了审核（这事我还真干过一次！），带来的不安和甚至是项目造成损失都是有的。这个环境管理肯定会有更好的方案的，一定可以找到！

### 使用多 Target 和宏管理

后来发现 target 可以预定义宏变量，仿佛打开了新世界的大门

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnwnmj0y2jj31ao070ab7.jpg)

这样就可以把宏定义转移到不同的 target 配置里，多搞几个 target，需要的时候运行不同的 target 就可以了，不用再在上线前手动修改代码，也就不用担心出错，再也不会把测试环境提审了！直接完美！

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwnpddoo0j31ai078q47.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwnpr8chnj31au074wfs.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79ly1fnwnrj08arj30go0600to.jpg)


如上图，我们定义 PROD 宏，一个 target 中值为 0，一个为 1，在代码里可以根据不同的 PROD 值来连接到不同的服务器，需要切换环境时只需要运行不同的 scheme 即可。

但是，多个 target 有一个很明显的弊端，添加文件时需要包含在多个 target 里，如果没有，就会报错，而如果环境比较多，添加文件时画风一般是这样的

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnwnup2j67j31k01260zw.jpg)

通常我们开发项目大部分时间集中在测试环境中，默认 target 作为测试环境时一般不会有问题，Xcode 会默认选中默认 target，但是切环境里，经常发现编译出错，原因就在于添加文件时忘记添加到对应的 target 里了，我们可以使用 [AllTargets](https://github.com/poboke/AllTargets) 插件来解决这个问题，这个插件可以帮我们默认选中所有 target，几个人还可以，人多的话有同事不一定装这个插件，还是会出现漏加文件的情况。

编译错误还可以解决，如果 catagory 文件没有添加，编译器并不会报错，但是运行时如果调用 catagory 中的方法，就会崩溃！这是很致命的，所以这个方案也不是特别完美。

一次偶然的机会看到 [AppCoda Weekly](http://digest.appcoda.com/) 推送的一个 issue 中有提到 [Effective Environment Switching In iOS](https://blog.usejournal.com/effective-environment-switching-in-ios-6df0b08e9556) 简单翻阅了一下，发现这才是我想要的效果。

### 使用多配置和宏管理

默认情况下，Xcode 模板工程会创建两个配置，Debug 和 Release，开发时运行代码使用 Debug 配置，打包一般使用 Release 配置。如果简单粗暴的处理，可以运行 Debug 配置时连接测试环境，Release 时连接生产环境，而内测呢？打 Debug 包？不好，Debug 运行时和 Release 运行一些限制是不一样的，有些问题并不会在 Debug 配置下暴露，上线时才用 Release 包测试发现问题已经太晚了。Xcode 可以添加配置

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwozy56hqj30zk0lldl2.jpg)

注意左右列表，应选择功能管理，而是不对 target 管理，点击 + 号复制一个配置即可，我们复制一个 Release 配置，命名为 `InhouseRelease`，修改默认的 Release 为 `DistributionRelease`，在 `InhouseRelease` 配置下，我们可以连接到测试环境，并不带 DEBUG 标识，也就是和默认的 Release 配置打包效果是一样的，在 `DistributionRelease` 配置下，连接到生产环境，打包或运行代码时，只需要使用不同的配置即可，target 还是只有一个。

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwomhtie2j30nd048aal.jpg)

为方便我们选择配置，可以新建 scheme 来给每个 scheme 以不同的默认配置，比如新建一个 `MultipleBuildConfigurationDemoInhouse` scheme，关联的可执行文件是同一个 target，Archive 时使用 `InhouseRelease` 配置，而原有的 scheme 中，Archive 使用 `DistributionRelease` 配置

![](https://ws1.sinaimg.cn/large/006tNc79ly1fnwor9yjwoj30ow0e0wfo.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnwoia5nk5j30ow0e0q43.jpg)

这样三个配置，一个 target，两个 scheme，就可以管理三个环境，切换环境只需要选择相应的 scheme 即可，不需要修改任何代码，添加文件时也不需要小心有没有添加到多个 target 里。

![](https://ws1.sinaimg.cn/large/006tNc79ly1fnwouq99nlj308n030aab.jpg)

如果还有环境需要管理，比如预发布、外部专用环境等，也只需要添加一个配置，预定义不同的宏即可，简单可扩展。

### 使用多配置和 User-defined setting 管理

Xcode 还提供一个功能，称之为 `User-defined setting`，Xcode 的工程模板中有对这个功能的利用，看 `Info.plist` 可以注意到一些参数并没有写死，而是通过类型「环境变量」的形式引用

![](https://ws3.sinaimg.cn/large/006tNc79ly1fnwp4gej58j30zk0ll794.jpg)


我们可以添加自己的变量，并在 Info.plist 中引用。

点击下图中的 + 号，添加一个 `User-defined setting`，命名配置名为 `CPY_API_BASE_URL`, 对应上一节中三个配置，分别给三个值，在 Info.plist 中新建一个 key，命名为 `API_BASE_URL`，引用 `CPY_API_BASE_URL` 变量，如图

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwprys7vbj30zk0llagb.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwppu0efkj30mn021dg5.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79ly1fnwpl3dsqej30zk0ll0xc.jpg)

这样我们可以读取 Info.plist 中的 key `API_BASE_URL`，在使用不同的配置打包时，同一个 key 得到三个不同的值，以达到连接到不同的环境的目的。

### 题外话

使用 `User-defined setting` 可以做到一些有趣的事，比如想让不同的配置打出的包有不同的 Bundle name

添加一个 `User-defined setting` 名为 `CPY_BUNDLE_NAME` 的变量，三个配置分别给三个值，并在 Info.plist 中引用，如图

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnwpbm2pycj30ho023q33.jpg)

![](https://ws1.sinaimg.cn/large/006tNc79ly1fnwpcyzkdbj30zk0lln21.jpg)

这样通过不同的配置打出来的包安装在手机上有了不同的名字，同理可以设置可执行文件的名字等。

### 结语

管理配置的方式有很多，看哪一款适合自己的情况，我想上面四种方案应该能解决绝大部分情况，如有错误，还望指出。

### 参考资料

[Effective Environment Switching in iOS](https://blog.usejournal.com/effective-environment-switching-in-ios-6df0b08e9556)

