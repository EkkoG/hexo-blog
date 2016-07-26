---
title: 创建私有 CocoaPods 仓库
tags:
  - CocoaPods
  - iOS
categories:
  - 编程
date: 2016-07-25 22:45:33
---

iOS 开发中我们经常用 CocoaPods 来安装第三方库，CocoaPods 可以帮助我们处理依赖，管理版本等等，还是挺方便的，如果我们公司内部有一个比较独立的组件，不想开源但是又想用 CocoaPods 来管理，可不可以呢？当然是可以的，CocoaPods 也提供了比较好的支持，可以自定义源，这样只要我们把 Pods 源和组件的源代码的权限放到私有的空间里，其他和公开仓库的作法是类似的。下面就简单说一下怎么创建私有的 CocoaPods 仓库。

下文中的 CocoaPods 使用的是 1.0.0 版本

### 创建样板工程

CocoaPods 提供了工具可以创建一个样板工程，在终端中 cd 到一个方便操作的目录，运行如下命令：

```
$ pod lib create CPYPrivatePodsDemo
```

CPYPrivatePodsDemo 根据自己的需要替换，这个就是组件的名字了。

CoccoaPods 会问几个问题

1. 要使用的语言
2. 是否需要样例工程
3. 是否需要一个测试框架
4. 是否需要基于 View 的测试
5. 还有类的前缀

我们这里不对这些选项做深入的探讨，我们只需要一个 Example demo 工程，前缀必选的，其他选 No 

![](https://ww3.sinaimg.cn/large/74681984gw1f66ke400ngj20qk0nvqbj)


完成后在当前目录会出现一个 `CPYPrivatePodsDemo` 目录，cd 到这个目录中，我们可以看到如下文件

![](https://ww3.sinaimg.cn/large/74681984gw1f66k0gknsaj208a03eq36)

这样我们就得到一个样板工程，完成创建后 CocoaPods 会打开样例工程的 `.xcworkspace` 文件。

### 把组件相关的类放到工程中

把组件相关的类放到 `CPYPrivatePodsDemo/Classes` 目录中，这是里之所以要放到这个目录下是因为 Example 工程是中生成 Podfile 文件中指定了这个目录是源文件地址，我们按照默认的走就好。这里我们创建了一个测试文件和一个头文件，导入了这个测试文件，共三个文件放到 `Classes` 文件夹，如下：

![](https://ww3.sinaimg.cn/large/74681984gw1f66k6nqmuvj20y0070q4t)

之后我们需要更新 Example 工程，在终端中 cd 到 Example 目录下并运行 `pod install` 命令

![](https://ww3.sinaimg.cn/large/74681984gw1f66k878ogcj20ku04qmym)

CocoaPods 会更新 pod，这里直接从之前的 `Classes` 文件夹中获取文件更新，完成后我们再看回 Xcode 是上目录树，添加的三个文件出现在了 `Development Pods` 下

![](https://ww3.sinaimg.cn/large/74681984gw1f66kg3k76gj20m60noq62)

此时我们可以在测试一下，在 `ViewController` 导入这个框架，看看能不能创建 `CPYTestView`

![](https://ww3.sinaimg.cn/large/74681984gw1f66krdecl5j21260h042q)

嗯。。。好像编译通过了，这样就完成一半了，剩下的就是把这个 pod 发布，放到一个大家都可以访问的地方。

### 发布组件到 Git 仓库中

上面完成后，我们就可以把创建出来的这个模板创建放到远程的 Git 仓库中，这里我们在 Coding.net 上创建一个仓库，然后把代码 push 到远程

```
git commit -m "first commit"
git remote add origin https://git.coding.net/cielpy/CPYPrivatePodsDemo.git
git push -u origin master
```

Podfile 指定的版本号的话在 `pod install` 时会找指定的 tag，所以我们这里需要打一个 tag

```
git tag -m "first release" 0.1.0
git push --tags     #推送tag到远端仓库
```

### 编辑 podspec 文件

接下来我们要编辑 podspec 文件，在样板工程的根目录下有一个 CPYPrivatePodsDemo.podspec 文件，用文本编辑器打开，需要 summary, homepage 和 source 字段，其中 source 字段是刚刚把源码上传到的 Git 仓库地址，需要用 HTTPS 链接，source_files 字段和默认的一样，不用修改，因为我们之前就是把源文件放到这个目录的。

```
#
# Be sure to run `pod lib lint CPYPrivatePodsDemo.podspec' to ensure this is a
# valid spec before submitting.
#
# Any lines starting with a # are optional, but their use is encouraged
# To learn more about a Podspec see http://guides.cocoapods.org/syntax/podspec.html
#

Pod::Spec.new do |s|
  s.name             = 'CPYPrivatePodsDemo'
  s.version          = '0.1.0'
  s.summary          = 'A private pod test demo'

# This description is used to generate tags and improve search results.
#   * Think: What does it do? Why did you write it? What is the focus?
#   * Try to keep it short, snappy and to the point.
#   * Write the description between the DESC delimiters below.
#   * Finally, don't worry about the indent, CocoaPods strips it!

  s.description      = <<-DESC
TODO: Add long description of the pod here.
                       DESC

  s.homepage         = 'https://coding.net/u/cielpy/p/CPYPrivatePodsDemo'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'Cielpy' => 'beijiu572@gmail.com' }
  s.source           = { :git => 'https://git.coding.net/cielpy/CPYPrivatePodsDemo.git', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '8.0'

  s.source_files = 'CPYPrivatePodsDemo/Classes/**/*'
  
  # s.resource_bundles = {
  #   'CPYPrivatePodsDemo' => ['CPYPrivatePodsDemo/Assets/*.png']
  # }

  # s.public_header_files = 'Pod/Classes/**/*.h'
  # s.frameworks = 'UIKit', 'MapKit'
  # s.dependency 'AFNetworking', '~> 2.3'
end

```

### 本地测试 podspec 文件是否可用

用 pod 提供的工具检查 podspec 文件是否合法，命令如下：

```
pod lib lint CPYPrivatePodsDemo.podspec
```

如果结果显示如下，说明是合法的

![](https://ww3.sinaimg.cn/large/74681984gw1f66mmrlax5j209b02r0t3)

然后我们在另建一个工程，运行 `pod init` 初始化 Podfile，加入 pod 并指定 podspec 路径，这里指定本地的 podspec 文件的路径。

```
use_frameworks!

target 'PodTest' do
pod 'CPYPrivatePodsDemo', :podspec => '/Users/xx/Downloads/CPYPrivatePodsDemo/CPYPrivatePodsDemo.podspec'
end
```

然后执行 `pod install`，顺利的话就可以安装成功了。如图

![](https://ww3.sinaimg.cn/large/74681984gw1f66n0ipzjfj20qb04dmyw)

### 发布 podspec

接下来剩最后一步，这们指定文件绝对路径是不科学的，我们不可能每次更新文件都修改一个 podspec 文件再 `pod install` 吧，方法就是发布 podspec 文件到远程，既然我们要做私有的 pod，就不能放在公共的仓库里，这里我们新建一个仓库，专门放我们的私有 podspec 文件。

在 coding.net 上另外建一个名字为 spec 的仓库，作为我们私有的 podspec 专用仓库，然后在本地添加一个新的源

```
pod repo add private https://git.coding.net/cielpy/spec.git
```

然后发布我们刚才编辑的 podspec 文件

![](https://ww3.sinaimg.cn/large/74681984gw1f66n76lxkzj20er08njt0)

显已添加到远程仓库中。

去远程仓库中看看

![](https://ww3.sinaimg.cn/large/74681984gw1f66n8g04kej20dm06l0t2)

有了！

### 最后一步

剩最后一步了，替换掉原来测试工程中的 Podfile 中指定的 podspec 路径，改成如下的正常的方式：

```
source 'https://git.coding.net/cielpy/spec.git'

use_frameworks!

target 'PodTest' do
pod 'CPYPrivatePodsDemo'
end

```

不指定版本号会取最新的一个版本，然后 `pod update` 更新版本的 podspec 并安装，正常的话，得到如下结果。

![](https://ww3.sinaimg.cn/large/74681984gw1f66nc3xqxsj20lo05lmyl)

到这里就完成了。

因为我们的仓库都是私有的，所以在哪里需要安装的话，需要有对这两个私有仓库的访问权限就好，发布公有的类似，只是发布到了官方的 podspec 仓库。

### 参考资料
[使用 Cocoapods 创建私有 podspec](http://blog.wtlucky.com/blog/2015/02/26/create-private-podspec/)

--EOF--


