---
title: 安装 Jenkins 遇到的那些坑
tags:
  - 持续集成
  - Jenkins
categories:
  - 编程
date: 2016-08-27 22:50:33
---



之前写了一篇 [持续集成那点事](https://imciel.com/2016/08/13/ios-ci/) 其中后半部分只讲了下简单的安装步骤，后来我又在另一台电脑上安装的时候，又遇到了几个坑，这里记录一下，供可能遇到同样情况的人参考。

注：下文中以安装 Jenkins LTS 2.7.2 为例

## 选择哪种安装方式安装？

Jenkins 的安装方式有几种：

1. 通过 pkg 安装器安装 https://jenkins.io/content/thank-you-downloading-os-x-installer/#stable
2. 直接下载 jenkins.jar http://mirrors.jenkins-ci.org/war-stable/latest/jenkins.war
3. 使用 Homebrew 安装

其中使用 pkg 安装虽然是一键，但是比较坑的是安装在 Shared 用户下，这样的话很多本机的配置如 SSH key 等会有一些权限问题，环境变量使用也不是很方便，一些配置需要在 Shared 用户上单独配置，好处是干净，但是 Xcode 已经是在当前用户下了，Shared 用户的配置只能干净一部分吧，干脆点，安装在当前用户，不用 Shared 用户了。这样很多配置是不用在 Jenkins 中配置的，如上方所详的 SSH key，以及各种环境变量都是在当前用户下，配置好后在 Jenkins 中不需要再配置。

<!-- more -->

直接运行 jar 是最简单的方式，不能再后台运行，总是开一个终端来运行 Jenkins 万一不小心关掉了就不好了，如果要后台运行，就需要使用 nohop 等辅助才能可以，不如使用 brew，brew 安装 Jenkins 只需要一条命令，当然前提是本机已经安装了 Homebrew，安装命令如下

```
brew install jenkins-lts
```

不追求新特性的话还是使用 LTS 版本吧，稳定性优先。

安装完成后可以使用 brew services 管理 Jenkins 启动，看帮助

```

$ brew services --help
usage: [sudo] brew services [--help] <command> [<formula>|--all]

Small wrapper around `launchctl` for supported formulae, commands available:
   cleanup Get rid of stale services and unused plists
   list    List all services managed by `brew services`
   restart Gracefully restart service(s)
   start   Start service(s)
   stop    Stop service(s)

Options, sudo and paths:

  sudo   When run as root, operates on /Library/LaunchDaemons (run at boot!)
  Run at boot:  /Library/LaunchDaemons
  Run at login: /Users/ciel/Library/LaunchAgents
```

其实 brew services 是对 launchctl 的一个包装，我们可以使用如下命令启用 Jenkins

```
brew services start jenkins-lts
```

同理可以 restart stop 等。

如果我们需要在非本机以外的电脑上访问 Jenkins，那么需要改一下配置文件，其配置文件在 `/usr/local/opt/jenkins-lts/homebrew.mxcl.jenkins.plist`，打开该文件，找到 `httpListenAddress` 后的 ip 地址，修改成本机 IP 或者 0.0.0.0 就可以。不好的是，如果版本升级了，这个配置文件需要重新修改一次。:(

另外提一句，有了该 plist 文件后，可以不使用 brew services 也可以启动或停止服务，上文说 brew services 是对 launchctl 的包装，那么可以直接用 launchctl 的吧？当然，参考 

https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/launchctl.1.html

## 设置 Git 仓库

如果我们使用了 pkg 安装，那么要解决的一个问题就是对 Git 仓库的访问权限，在 Jenkins 中可以配置 Git 的账户密码或者 ssh key，但是如果我们有一个打包脚本，脚本中也有拉取代码操作，我们想可以手动打包，也可以使用 Jenkins 定期运行打包呢？如果打包脚本中有 git pull 之类的操作，会提示有权限错误，这里是不很懂在 Jenkins 页面中配置的 ssh key 的运作方式，按理来说如果是配置 Shared 用户的 ssh key 的话，理应有同样的访问权限的。

我没有继续在 Shared 用户下搞，安装在了当前用户下，当前用户有权限访问 Git 仓库的话，Jenkins 是不用配置权限相关的东西的（ssh key 或者账号密码），同样，打包脚本一般在当前用户下运行，也不会有访问权限问题。

## 设置构建周期

构建周期一般有两种设置方式，如果需要定期检查代码并构建，那么可以设置 Poll SCM 为

![](https://i.imgur.com/8ELUVDW.jpg)

图中配置为每 5 分钟检查一次代码更新，有新的代码的话就执行构建。

如果需要定时打包之类的操作，可以设置如下

![](https://i.imgur.com/VerUBsY.jpg)

这样就是在每天晚上 21 点执行构建。

## 打包签名问题

之前的文章中，构建脚本使用的 xctool，xctool 正常情况下可以编译，执行测试，Analyze 等，有一问题 xctool 解决起来比较麻烦，就是签名问题，因为编译也必须保证签名的正确，而 xctool 不能指定证书和 Provisioning File，因为我的环境中已经使用 Fastlane 打包，Fastlane 可以顺便解决证书和 Provisioning File 问题，参见 https://codesigning.guide/ ，嗯。。。其实我这里还没用上这个高级玩意，只是在部署 Jenkins 的那台电脑上已经部署好了发布证书，Fastlane 直接导出一个 Release 包就可以了，可以导出代码就是正常的，验证了代码的可用性，基本的可用性。

可以定义一个 lanes 来导出 IPA

```
    gym(
      output_directory: path,
      scheme: "Test",
      export_method: "ad-hoc",
    )
```

## RVM 和 Fastlane

如果使用了 RVM 安装 ruby，并使用 RVM 安装的 ruby 安装了 fastlane，在 Jenkins 环境中，RVM 没有激活，所以也找不到 fastlane，进而也打不了包，所以要解决一下这个环境问题。在构建脚本中加上这几行，用来激活 RVM

```
#!/bin/bash -e
export TERM=xterm
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
# Use the correct ruby
rvm use 2.3.1

type rvm | head -n 1
```

查看一下日志，看还有没有 rvm commmand not found 之类的提示，如果正常的话，执行 `type rvm | head -n 1` 的时候会输出

```
rvm is a shell function from /Users/xx/.rvm/scripts/cli
```

## 设置邮件模板

邮件模板可以在 Jenkins 的系统配置里设置默认的，一般我们在不同的情况下发不同的邮件，比如构建失败时，会 log 发给开发者（这里会自动提取 git 提交者设置的邮件），并提示开发者构建失败了，请立即修改，成功了就不必提示。如果需要发测试包，则需要在构建成功后，发送给测试都，并附上更新日志和下载地址，如果打包失败同需要提示开发者。这些操作可以通过定义不同的触发器，在 Jenkins 项目中的 Post-build Actions 中，给不同的 triggers 设置不同的邮件模板，如失败时，给开发者的模板：

![](https://i.imgur.com/pf2C70d.jpg)

成功时给测试者的模板：

![](https://i.imgur.com/YOMkIPb.jpg)

这里提一下 Git log 的提取，Jenkins 可以通过参数得到从上次构建成功后本次构建的 commit message，在邮件中可以通过 ${CHANGES} 参数获取到，并进行格式化，如果只需要一行一个 commit message ，就可以按照图中的方法设置，这里附上我设置的发给测试者的模板：

```
$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:

Changes: <br>
${CHANGES, format="-- %m <br>"} <br>

Fir.im 链接：https://fir.im/xxxx

Check console output at $BUILD_URL to view the results.
```

这里之所以用 <br> 换行，是因为我默认的邮件格式是 html，如果是 plaintext，也可以使用 \n 换行。

## SMTP 的设置

邮件模板再好看，不能发还是白搭，我们需要一个方式给把邮件发出去，SMTP 可以做到，一般的邮件服务商都提供这种通用的发邮件方式，在 Jenkins 的系统配置里，添加 Extended E-mail Notification 配置，并设置 SMTP server，这里以 QQ 邮箱为例，QQ 邮件强制使用 SSL，这里注意一下

![](https://i.imgur.com/lY4oRRZ.jpg)

顺便提一句，再往下可以定义默认模板：

![](https://i.imgur.com/1Bwj3N2.jpg)

需要在 Jenkins 配置中设置系统管理员邮箱为发邮件的邮箱，猜测是做了一个什么认证。

![](https://i.imgur.com/bCevqeC.jpg)

在我的网络环境中，有一个奇怪的问题是我设置的 QQ 邮箱服务器提示连不上，但是我用 [Python 脚本发邮件](https://jasonhzy.github.io/2016/06/15/python-email/) 可以发送，同样试了企业邮箱也不行，换成我自定义域名的 yandex 邮件才可以，没有做其他修改。

## 写在后面

又到了写在后面时间，本文记录了一些安装 Jenkins 的时候的一些比较坑的地方，以及在一些场景下 Jenkins 的应用，希望对有类似需求的朋友有所帮助。

## 参考资料

* [launchctl manual](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/launchctl.1.html)
* [Using RVM with Jenkins](https://mattconnolly.wordpress.com/2012/01/29/using-rvm-with-jenkins/)
* [A new approach to code signing](https://codesigning.guide/)
* [Python SMTP发送邮件](https://jasonhzy.github.io/2016/06/15/python-email/)


--EOF--

