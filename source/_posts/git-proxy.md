---
title: 设置 Git 代理
tags:
  - Git
  - 代理
  - GFW
categories:
  - 杂项
date: 2016-06-28 22:30:20
---

如果经常逛 GitHub，经常 clone GitHub 上某个仓库的代码，那么就会经常遇到一个问题，就是 clone 的时巨慢，几K/s 这样。

小一点的仓库可能等一下就过去了，如果是很大的仓库，如这个 Git 的源代码仓库，或者 `CocoaPods` 的 `Specs` 仓库，这些仓库的大小可能在几百M，`Specs` 仓库 clone 下来是300M 左右，如果是几十K的速度，等 clone 完成人就已经疯了，这也是为什么在配置 CocoaPods 后，运行 `pod setup` 命令时总是卡半天不动的原因所在。如果手上有代理，且代理的质量还不错，那么就可以考虑让 Git 走代理，这样可以提高 clone 代码时的速度，改善生活质量:)

<!-- more -->

通常我们 clone 代码时可以选择两种方式

```
//方式一：HTTP
https://github.com/git/git.git
//方式二：SSH
git@github.com:git/git.git
```

两种方式设置代理的方法是不同的，下面一一介绍。

## 设置 Git HTTP 代理

如果你手上的代理是 socks5 代理，如各平台的 `Shadowsocks` 客户端都提供一个本地的 socks5 代理，那么你可以这样设置，让 Git 通过 HTTP 链接 clone 代码时走 socks5 代理

```
//通过 http 链接 clone 代码时走 socks5 代理
git config --global http.proxy "socks5://127.0.0.1:6666"
//通过 https 链接 clone 代码时走 socks5代理
git config --global https.proxy "socks5://127.0.0.1:6666"
```

如果你手上的代理是 http 的，那么可以这样设置

```
//通过 http 链接 clone 代码时走 http 代理
git config --global http.proxy "http://127.0.0.1:6667"
//通过 https 链接 clone 代码时走 http 代理
git config --global https.proxy "http://127.0.0.1:6667"
```

设置完成后，可以 clone 一份代码试一下有没有效果。

这些设置最终会保存在用户目录下的 `.gitconfig` 文件中，打开这个文件可以看到类似的几行配置

```
[http]
    proxy = http://127.0.0.1:6152
[https]
    proxy = http://127.0.0.1:6152

```

如果端口有变动也可以直接在这里修改。

## 设置 Git SSH 代理

还有一种情况，我们通过 SSH 方法 clone 代码，提交代码，因为这样不用输入密码，通常我们会在自己的常用电脑上这么做。上面设置的 HTTP 代理对这种方式 clone 代码是没有影响的，也就是并不会加速，SSH 的代理需要单独设置，其实这个跟 Git 的关系已经不是很大，我们需要改的，是SSH 的配置。在用户目录下建立如下文件 `~/.ssh/config`，对 GitHub 的域名做单独的处理

```
# 这里必须是 github.com，因为这个跟我们 clone 代码时的链接有关
Host github.com
   # 如果用默认端口，这里是 github.com，如果想用443端口，这里就是 ssh.github.com 详见 https://help.github.com/articles/using-ssh-over-the-https-port/
   HostName github.com
   User git
   # 如果是 HTTP 代理，把下面这行取消注释，并把 proxyport 改成自己的 http 代理的端口
   # ProxyCommand socat - PROXY:127.0.0.1:%h:%p,proxyport=6667
   # 如果是 socks5 代理，则把下面这行取消注释，并把 6666 改成自己 socks5 代理的端口
   # ProxyCommand nc -v -x 127.0.0.1:6666 %h %p

```

根据代码中的注释，设置自己的代理，查了很多资料 socks5 可以配合 nc 使用，但是我这里试了下不行，各位可以试一下，暂时把这个方案留在这里。

经过上面的设置，现在不管是用什么方式 clone 代码，都会走代理了，这里还是强调一下，代理要速度快才会有加速效果，如果代理一般或者很慢，可能还不如不走代理。

## 写在后面

希望这些配置可以改善我们天朝程序员的生活质量吧:)

--EOF--

## 参考资料
* [Tutorial: how to use git through a proxy](https://cms-sw.github.io/tutorial-proxy.html)
* [Connecting to host by SSH client in Linux by proxy](https://unix.stackexchange.com/questions/68826/connecting-to-host-by-ssh-client-in-linux-by-proxy)
* [SSH over SSL, episode 2: replacing proxytunnel with socat](http://blog.chmd.fr/ssh-over-ssl-episode-2-replacing-proxytunnel-with-socat.html)
* [配置git使用proxy](http://leolovenet.com/blog/2014/05/28/git-and-proxy/)
* [Using SSH over the HTTPS port](https://help.github.com/articles/using-ssh-over-the-https-port/)

-- EOF --

