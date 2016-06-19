---
title: 我的路由器自动翻墙方案 shadowsocks+dns2socks+pdnsd+dnsmasq
tags:      
    - 翻墙
    - GFW
    - 网络
categories:      
    - 杂项
date: 2016-01-22 18:28:22
---



大概从12年底开始学习使用 GoAgent 翻墙开始，到后来开翻墙软件成了上网的起手式，最后把翻墙软件部署到了路由器上，连接路由器的所有设备不用配置自动翻墙。就像这位推友说的
<!-- more -->

![](https://ww4.sinaimg.cn/large/74681984gw1f08gjaes3hj20hj0crq5b.jpg)

这里介绍下我现在在用的方案。

## 准备

### 硬件准备

可以刷 OpenWrt 的路由器一台

我目前在用的路由器是 Netgear WNDR 4300 ， 配有16M ram和128M rom，双频，电商打折的时候300块左右。

### 软件准备

1. [shadowsocks-libev]( https://github.com/shadowsocks/openwrt-shadowsocks) 提供本地 socks5代理给 dns2socks 使用
2. [shadowsocks-libev-spec]( https://github.com/shadowsocks/openwrt-shadowsocks) 提供本地透明代理
3. pdnsd [http://members.home.nl/p.a.rombouts/pdnsd/](http://members.home.nl/p.a.rombouts/pdnsd/) 转发 DNS 请求给上游 DNS，并修改 TTL ，提供缓存功能，这里上游 DNS 是 dns2socks 提供的 DNS
4. dns2socks [http://sourceforge.net/projects/dns2socks/](http://sourceforge.net/projects/dns2socks/) 转发 DNS 请求给 socks5代理
5. dnsmasq-china-list: [https://github.com/felixonmars/dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list) 收集国内常见域名，走国内 DNS 服务器解析，其他走 dns2socks 解析
6. shadowsocks-luci [http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/](http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/) 提供 Luci 界面，方便配置

软件都用了最新的版本。shadowsocks 可以看项目主页中提供的预编译 ipk 下载地址选择下载。

pdnsd 可以找 OpenWrt 的资源 [点击下载](https://downloads.openwrt.org/barrier_breaker/14.07/ar71xx/nand/packages/oldpackages/pdnsd_1.2.9a-par-a8e46ccba7b0fa2230d6c42ab6dcd92926f6c21d_ar71xx.ipk)

dns2socks 2.0版 没有现成的 ipk，这里我编译了一下，只编译了 rampis 和 ar71xx 两种型号 CPU 的适配版下载，

rampis版下载: https://imciel.com/resource/dns2socks_2.0-20151206_ramips_24kec.ipk

ar71xx 版下载: https://imciel.com/resource/dns2socks_2.0-20151206_ar71xx.ipk

## 给路由器刷 OpenWrt 

WNDR 4300刷 OpenWrt 相对简单，不需要解锁 U-boot，这里假设是新路由器，也就是目前的固件是原厂的，那么刷机的主要步骤如下：

1. 下载openwrt-ar71xx-nand-wndr4300-ubi-factory.img固件，最新版下载链接：[点击下载](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/nand/openwrt-15.05-ar71xx-nand-wndr4300-ubi-factory.img) （注意这里是 ubi-factory）
2. 进路由器后台，找到『固件升级』，然后上传 openwrt-ar71xx-nand-wndr4300-ubi-factory.img 固件，之后路由器会自动由成 OpenWrt，等待重启完成就可以了。

更详细的步骤和注意方式请参考这篇文章：http://dlmao.com/wndr4300-%E6%8A%98%E8%85%BE-openwrt-%E8%AE%B0.html

## 软件安装
OpenWrt 刷成功后，登录后台，配置网络（拔号、静态IP等等），用 ssh 登录后台，Mac 或者 Linux 系统可以在终端里执行 ssh 命令来登录，如：
`ssh root@192.168.1.1`
Windows 建议用 putty http://www.putty.org/

出现类似如下界面就是登录成功了
![](https://o4zqhe4wo.qnssl.com/blog-img/1465400772360.png)

此时需要执行 `opkg update` 更新软件包，更新完成后，用 scp 命令把下载好的软件包传到路由器中，命令如下
`scp shadowsock-libev-spec.ipk pdnsd.ipk dns2socks.ipk root@192.168.1.1:/tmp`
这个命令会把当前目录下的三个 ipk 文件传到 ip 地址为192.168.1.1的路由器的/tmp 目录
scp 的具体用法参考这篇文章：http://www.cnblogs.com/peida/archive/2013/03/15/2960802.html
传输完成后，换到 ssh 登录的页面，`cd /tmp` 切换到 `/tmp` 目录，`ls` 看一下三个 ipk 是否在这个目录，如果不存在检查下上面执行 `scp` 命令的时候是不是哪里报错没看到，或者目录写错了，如果存在，那么执行安装命令，如下
`opkg install shadowsock-libev-spec.ipk pdnsd.ipk dns2socks.ipk`
如果 `opkg update` 时正常，那么这个时候应该是会正常安装成功的。

dns2socks 需要一个 socks5 代理，其原理就是将 DNS 解析请求通过 socks 代理解析，这里 socks5代理用 shadowsocks-libev 提供，由于已经 安装了 shadowsocks-libev-spec，如果直接安装 shaodowsokc-libev 会和 shadowsocks-libev-spec 的部分文件冲突，那么就手动安装 shadowsocks-libev，方法：
1. 在电脑上重命名 shadowsocks-libev.ipk 为 shadowsocks-libev.zip
2. 解压 zip
3. 进入解压出来的目录
4. 解析 data.tar.gz
5. 进入解压出来的 data 目录
6. 下载 shadowsocks-local https://gist.github.com/cielpy/75f3e1d891a88e176f34
7. 找到 data/usr/bin/ss-local
8. 将 shadowsocks-local 和 ss-local 用 scp 传到路由器上
9. 将 shadowsocks-local 复制到路由器的 /etc/init.d/ 目录
10. 将 ss-local 复制到/usr/bin/ 目录

至此，安装软件完成

## 软件配置
* 配置 shadowsocks 账号，登录路由器后台，服务-shadowsocks，对应填上自己的 shadowsocks账号就行了
  ![](https://ww1.sinaimg.cn/large/74681984gw1f08pim5dlvj20gj0eo3z4.jpg)
* 设置 dns2socks 下载脚本 https://gist.github.com/cielpy/798f1a0e9ee01ffc8f4a 并上传到路由器 /etc/init.d/ 目录
* 设置 pdnsd 下载脚本 https://gist.github.com/cielpy/b1356b009c887ce22de4 并上传到路由器 /etc/init.d/ 目录
* 下载 dnsmasq-china-list 并解压，将里面的 `.conf` 文件上传到路由器的 /etc/dnsmasq.d/ 目录，若无这个目录，新建一个即可
* 在/etc/dnsmasq.conf 中加上以下几行
```
server=127.0.0.1#7453
no-resolv
conf-dir=/etc/dnsmasq.d
```
* 在路由器中（ssh 登录到路由器的那个界面）运行命令 `wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt` 
获取国内的 IP 段，我们在 shadowsocks 中指定了忽略的 IP 段文件，我们还没有下载这个 IP 段文件，使用这个命令将这个文件保存到了 `/etc/chinadns_chnroute.txt`

至此配置完毕。
这里解释下各个脚本的关系，之前下载的 shadowsocks-local 脚本中，读取了 shaodowsocks-libev-spec 的shadowsocks 服务器配置，并监听 `2080` 端口，这样修改过服务器后，需重启 shadowsocks-local 服务，就可以自动获取刚刚更换的服务器，DNS 也会转发到新的服务器上解析。
下载的 dns2socks 脚本中，socks5代理设置为了 shadowsocks-local 的`2080` 端口，并监听 `1053` 端口
下载的 pdnsd 脚本中，上游 DNS 设置为了本地的 `1053` 端口的 DNS，也就是 dns2socks，并监听 `7453` 端口，这样在本地 `7453` 有一个自定义的 DNS 服务器
/etc/dnsmasq.d/中的 `.conf` 文件指定了国内常见域名走 `114.114.114.114` 解析，这个 DNS 可以根据自己的情况替换，用国内的都可以，看自己喜好。
/etc/dnsmasq.conf 中添加的几行意思是，指定本地的 DNS 为 `127.0.0.1#7453`，也就是 pdnsd，`no-resolv` 作用为不查询运营商的 DNS，如果不指定这个，dnsmasq 服务会用运营商的默认 DNS进制解析 `conf-dir=/etc/dnsmasq.d` 作用为读取这个目录下的配置文件，这里的配置文件一般指定了哪些域名走特定的 DNS。

## 运行软件
`/etc/init.d/shadowsocks start` 启动 shadowsocks-libev-spec 透明代理服务
`/etc/init.d/shadsocks-local start` 启动本地 socks5代理
`/etc/init.d/dns2socks start` 启动 dns2socks 服务
`/etc/init.d/pdnsd start` 启动 pdnsd 服务
`/etc/init.d/dnsmasq restart` 重启 dnsmasq 服务，使新的配置文件生效

## 完结
至些所有服务配置去年完毕，理论上应该成功了，总共用了4个软件，都是开源软件，shadowsocks-libev-spec ,shadowsocks-libev, dns2socks, pdnsd, 一起工作，解决翻墙问题。

## 感谢
感谢各位开发者的劳动，有你们，才能让我们方便的浏览互联网上的任何信息。

## 参考
1. [Shadowsocks + ChnRoute 实现 OpenWRT 路由器自动翻墙](https://cokebar.info/archives/664)
2. [WNDR4300 折腾 openwrt 记](http://dlmao.com/wndr4300-%E6%8A%98%E8%85%BE-openwrt-%E8%AE%B0.html)
3. [科学上网之五----dns2socks](http://www.bubuko.com/infodetail-624247.html)
4. [加速OpenWRT路由器的DNS解析 – pdnsd代替dnsmasq](https://cokebar.info/archives/734)


