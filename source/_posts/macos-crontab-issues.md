---
title: macOS 计划任务的一些问题
tags:
  - macOS
categories:
  - 杂项
date: 2016-12-12 17:01:45
---

最近在搞一个小东西，搞这个东西呢，想达到的效果如下：

1. 定时更新仓库代码
2. 根据最新一条 commit 的 message 中是否包含特定字符串来决定是否打包
3. 打包过的 commit 不再打包
4. 打成成功后过滤一下与上次打包之前 commit 的日志，并作为邮件内容发送给特定的人
5. 打包失败了把日志发给我自己

大体上功能就是这些，不过有一些细节要处理，诸如打包要生成的文件放的目录存在与否，打包成功后的 commmit sha1 值记录方式等，搞定这些后，剩下最后一步，打包机没有放在公网，也就没有 IP 地址，所以也不能使用 Webhook 来及时知道仓库有代码更新，只好轮询了，脚本负责更新代码和打包，定时轮询这个交给系统的计划任务就好，但是这最后一步远没有想的那么简单。

<!-- more -->

## 计划任务环境中的环境变量

踩到的第一个坑是环境变量问题，如果只在当前用户的计划任务里添加一条：

```
*/1 * * * * echo $PATH > /tmp/cron.log
```

等待其自动运行后，log 文件是空的，PATH 变量为空，如果这个时候脚本中用到了一些自己安装的CLI 程序，那么就会出现 `command not found` 错误，因为我使用了 fir-cli 来上传 IPA 包，所以要手动添加一个环境变量，我添加了几个，

```
PATH="/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin"
```

这样脚本就可以找到相应的 CLI 程序，正常使用相关的服务

## Ruby 兼容性问题

原本我的系统中使用了 rvm 安装了最新的 ruby，打包时遇到了错误，Google 一下，找到了解决方法---使用系统的 Ruby，至少在脚本用应该使用系统自带的 Ruby。
在执行 `xcodebuild` 前，将 Ruby 切换为系统版本：

```
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
rvm use system
```

## 钥匙串访问

这个是最坑的一个了。

平常我们不管是新建证书还是使用别人导出的 p12 文件导入证书，默认都是导入了登录钥匙串，但是如果在计划任务的环境中输出一下钥匙串列表，会发现少了这个登录钥匙串，代码如下：

```
*/1 * * * * security list-keychains > /tmp/cron.log
```

得到结果如下：

```
    "/Library/Keychains/System.keychain"
```

而我们如果在终端里手动输出一下，一般至少会得到以下两个：

```
    "/Users/ciel/Library/Keychains/login.keychain-db"
    "/Library/Keychains/System.keychain"
```

而我们的证书是存在 `login.keychain` 中的，这会导致在计划任务里运行 xcodebuild 进行导出 archive 操作的时候，总是找不到证书来签名，报错如下：

```
No valid iOS Distribution signing identities belonging to team 9ATJxxxx were found.
```

找了一些方法，挨个试了下，最简单的就是把相关的证书拖到系统钥匙串中，至少有两个：
![](https://ws3.sinaimg.cn/large/74681984gw1fao5gvoy1yj20il08vjst.jpg)
![](https://ws3.sinaimg.cn/large/74681984gw1fao5iyqwgmj20kt09gmxz.jpg)
一个是打包所用的证书，一个是使用的这个证书的签发者的证书：
![](https://ws3.sinaimg.cn/large/74681984gw1fao5k911m5j20e90c0gnk.jpg)

因为计划任务的环境中本身就读得到系统钥匙串，所以 archive 导出的时候就可以正常找到证书，完成导出过程。

以上是解决这个问题的办法，成功的一个，也是我在用的一个，还有一些失败的方法，这里列举一下，供参考。

### 解锁登录钥匙串

查的很多资料中都介绍了可以使用解决的方式来让计划任务中读取登录钥匙串的内容，命令如下：

```
security unlock -p password /path/to/Login.keychain
```

一方面这个方法需要把密码写在脚本中，或者保存在环境变量中，不是很安全，一方法我这里试了下，还是读不到登录钥匙串中的证书，同样报错。试了几次不行后放弃了这个方案。

### 自定义钥匙串

这个是我认为的最佳方案，不过可惜的是没有成功。

在钥匙串管理中我们可以新建一个钥匙串专门放打包所使用的证书，这样可以隔离其他不想暴露的信息，即使把密码写在脚本中也问题不大。同样避免了把证书放在系统钥匙串中这个不是很干净的做法。

不过遇到的问题和上面一样，可以解锁但是不能读取，最后还是放弃了这个方案。

如果以上问题有解决的话我再更新一下。

## 写在后面

折腾完后感觉还是不错的，基本达到了我想要的效果，这样如果需要打包的话几乎不需要手动操作，只需要改一下版本号（或者 build 号）并 push 代码，计划任务定时更新代码，并检测到符合打包要求就可以进行打包并上传了。省心省事。

## 参考资料
* [xcodebuild: “No applicable devices found.” when exporting archive](http://stackoverflow.com/a/33041110/1841463)
* [Missing certificates and keys in the keychain while using Jenkins/Hudson as Continuous Integration for iOS and Mac development](Missing certificates and keys in the keychain while using Jenkins/Hudson as Continuous Integration for iOS and Mac development)
* [给iOS工程增加Daily Build](http://blog.devtang.com/2012/02/16/apply-daily-build-in-ios-project/)
* [OS X下让supervisor开机启动，以及权限、环境变量、codesign问题](OS X下让supervisor开机启动，以及权限、环境变量、codesign问题)







