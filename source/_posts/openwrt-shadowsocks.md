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

大概从12年底开始学习使用 [GoAgent](https://github.com/goagent/goagent) 翻墙开始，到后来开翻墙软件成了上网的起手式，最后把翻墙软件部署到了路由器上，连接路由器的所有设备不用配置自动翻墙。就像这位推友说的
<!-- more -->

![](https://ww3.sinaimg.cn/large/74681984gw1f08gjaes3hj20hj0crq5b.jpg)

也瞎折腾了几年了，这里介绍下我现在在用的方案。

## 方案介绍

本文主要使用 shadowsocks-libev 提供的透明代理功能，将 shadowsocks-libev 部署于路由器，使透明代理运行于服务器，部署完成后，达到连接该路由器的设备都可以自动翻墙的效果。

shadowsocks-libev 主要根据 IP 来判断是否需要走代理，可以设置一个 [IP-CIDR](https://www.wikiwand.com/en/Classless_Inter-Domain_Routing) 列表，列表内的 IP 不走代理，列表外的 IP 全部走代理，这样做的好处是不需要额外的维护需要走代理的域名列表（如 GFWList），只需要偶而更新一下 IP-CIDR 列表就好，这个列表很少有变动，所以部署好后就不怎么需要维护了。当然这样的方案对代理服务器的要求稍高，通常我们将 shadowsocks-libev 的 IP-CIDR 列表设置为所有国内 IP，这样一来所有非国内的 IP 都会走代理，如果代理服务器的速度不好，访问国外网站的时候会有减速效果，相反如果代理服务器质量不错，那么访问国外的网站有不错的加速效果。

由于这规则基于 IP 来判断，所以必须保证域名解析的时候没有问题，也就是必须返回正确的 IP，接下来就是必须解决的问题 --- DNS 污染。shadowsocks-libev 的部署相对简单，本文中后半部分主要解决的问题就是 DNS 污染以及一些 CDN 的解析设置。

方案的大概原理介绍完了，接下来就开始吧。


## 硬件准备

可以刷 OpenWrt 的路由器一台

我目前在用的路由器是 Netgear WNDR 4300， 配有 128M RAM 和 128M ROM，双频，电商打折的时候300块左右，本人已经用了一年多，还是比较稳定的。

## 软件准备

1. [shadowsocks-libev]( https://github.com/shadowsocks/openwrt-shadowsocks) 提供本地 SOCKS5代理和本地透明代理
2. DNS2SOCKS [https://sourceforge.net/projects/DNS2SOCKS/](http://sourceforge.net/projects/DNS2SOCKS/) 通过 SOCKS5 代理请求 DNS
3. pdnsd [http://members.home.nl/p.a.rombouts/pdnsd/](http://members.home.nl/p.a.rombouts/pdnsd/) 转发 DNS 请求给上游 DNS，并修改 TTL ，提供缓存功能，这里上游 DNS 是 DNS2SOCKS 提供的 DNS
4. dnsmasq-china-list: [https://github.com/felixonmars/dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list) 收集国内常见域名，走国内 DNS 服务器解析，其他走 DNS2SOCKS 解析
5. shadowsocks-luci [http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/](http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/) 提供 Luci 界面，方便配置

本文中 shadowsocks-libev 使用 2.4.8 版本，shadowsocks-libev 项目主页中提供的预编译 ipk 下载地址选择下载。

pdnsd 可以找 OpenWrt 的资源 [点击下载](https://downloads.openwrt.org/barrier_breaker/14.07/ar71xx/nand/packages/oldpackages/pdnsd_1.2.9a-par-a8e46ccba7b0fa2230d6c42ab6dcd92926f6c21d_ar71xx.ipk)

DNS2SOCKS 2.0版 没有现成的 ipk，这里我编译了一下，只编译了 rampis 和 ar71xx 两种型号 CPU 的适配版下载，

rampis版下载: https://imciel.com/resource/dns2socks_2.0-20151206_ramips_24kec.ipk

ar71xx 版下载: https://imciel.com/resource/dns2socks_2.0-20151206_ar71xx.ipk

## 给路由器刷 OpenWrt 

WNDR 4300刷 OpenWrt 相对简单，不需要解锁 U-boot，这里假设是新路由器，也就是目前的固件是原厂的，那么刷机的主要步骤如下：

1. 下载openwrt-ar71xx-nand-wndr4300-ubi-factory.img固件，最新版下载链接：[点击下载](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/nand/openwrt-15.05-ar71xx-nand-wndr4300-ubi-factory.img) （注意这里是 ubi-factory）
2. 进路由器后台，找到『固件升级』，然后上传 openwrt-ar71xx-nand-wndr4300-ubi-factory.img 固件，之后路由器会自动由成 OpenWrt，等待重启完成就可以了。

更详细的步骤和注意方式请参考这篇文章：http://dlmao.com/wndr4300-%E6%8A%98%E8%85%BE-openwrt-%E8%AE%B0.html

## 软件安装

### ssh 连接路由器

OpenWrt 刷成功后，需要先登录后台，配置网络（拔号、静态IP等等），使路由器正常访问网络，然后用 ssh 登录后台，Mac 或者 Linux 系统可以在终端里执行 ssh 命令来登录，如：

```
ssh root@192.168.1.1
```

Windows 建议用 putty http://www.putty.org/

出现类似如下界面就是登录成功了

![](https://ww3.sinaimg.cn/large/74681984gw1f79g740emcj20zo0jqgsn)

### 传输 ipk 文件到路由器并安装

首先需要更新软件源，命令如下：

```
opkg update
```

更新完成后，用 scp 命令把下载好的软件包传到路由器中，命令如下：

```
scp shadowsock-libev-spec.ipk pdnsd.ipk dns2socks.ipk root@192.168.1.1:/tmp
```

以上 scp 命令会把当前目录下的三个 ipk 文件传到 ip 地址为192.168.1.1的路由器的 `/tmp` 目录下，注意替换自己的 ipk 名字和路径（可使用绝对路径），这里文件名只是举一个例子。

Unix-like 系统中一般都内置了 scp 命令，Windows 可以使用 [WinSCP](https://winscp.net/eng/download.php)，如果你还不熟悉 scp 的使用，可以参考这篇博客，http://www.cnblogs.com/peida/archive/2013/03/15/2960802.html

传输完成后，换到 ssh 登录的页面，切换目录到 `/tmp`

```
cd /tmp
```

可以使用 `ls` 命令确认一下之前的三个 ipk 是否在于这个目录，如果不存在检查下上面执行 `scp` 命令的时候是不是哪里报错没看到，或者目录写错了，一切顺利的话，ipk 文件应该出现这个目录下，接下来安装这三个 ipk，命令如下：

```
opkg install shadowsock-libev-spec.ipk pdnsd.ipk dns2socks.ipk
```

如果 `opkg update` 时正常，那么这个时候应该是会正常安装成功的。同样这里的文件名只是例子，请根据自己的情况自行替换文件名或者文件路径，如果当前文件夹下只有这三个 ipk 文件，并且想一次性安装这三个 ipk 文件，可以使用如下命令：

```
opkg install *.ipk
```

之前之所以使用 `opkg update` 命令更新软件源，是因为 shadowsocks-libev 还有一些依赖需要安装，这些依赖可以在官方维护的软件源中找到，故使用 `opkg update` 更新一下软件源，安装 shadowsocks-libev 的时候可以正确安装依赖。

DNS2SOCKS 需要一个 SOCKS5 代理，其原理就是将 DNS 解析请求通过 SOCSK5 代理解析，这里 SOCKS5 代理用 shadowsocks-libev 提供，之前版本中，shadowsocks-libev-spec 不提供本地 SOCKS5 代理，所以需要另外安装一个 shadowsocks-libev，现在最新版本的已经同时提供透明代理和 SOCKS5 代理，省心多了。

至此软件安装部分就完成了，接下来需要配置各种软件，使它们配合起来工作。

## 软件配置

### 配置 shadowsocks 账号和本地 SOCKS5 代理

登录路由器后台，服务-影梭，切到服务器管理选项卡，填上自己的 shadowsocks账号即可，可以配置多个服务器，方便一个挂了的时候切换到另一个:)

![](https://ww3.sinaimg.cn/large/74681984gw1f79i16p15bj20ep0eggms)

在基本设置中，可以设置选用的服务器，以及设置 SOCKS5 代理使用哪个 shadowsocks 服务器，并设置 SOCKS5 代理端口，留给下文中的 DNS2SOCKS 用。

![](https://ww3.sinaimg.cn/large/74681984gw1f79icll52kj20ei0lf0u7)

如果透明代理的 UDP 服务器处提示 tproxy 模块缺失，则需要安装一下 tproxy 模块，命令如下：

```
opkg update
opkg install iptables-mod-tproxy
```

可能需要重启路由器才能生效。


### 配置 DNS2SOCKS 启动脚本

DNS2SOCKS 安装完成只有一个可执行文件，我们可以把 DNS2SOCKS 改造成像普通的 OpenWrt 软件那样，使用 `/etc/init.d/` 下的脚本 `start`, `stop`, `restart`，方法如下，在 `/etc/init.d/` 目录下新建文件 DNS2SOCKS，内容如下：

```
#!/bin/sh /etc/rc.common
# Copyright (C) 2006-2011 OpenWrt.org

START=90

SERVICE_USE_PID=1
SERVICE_WRITE_PID=1
SERVICE_DAEMONIZE=1

start() {
        service_start /usr/bin/dns2socks 127.0.0.1:LOCAL_SOCKS5_PORT 8.8.8.8:53 127.0.0.1:LOCAL_DNS2SOCKS_PORT -q
}

stop() {
        service_stop /usr/bin/DNS2SOCKS
}
```

并设置可执行权限：

```
chmod +X /etc/init.d/dns2socks
```

注意脚本中的两个参数需要自己修改，`LOCAL_SOCKS5_PORT` 为自己的 SOCKS5 端口，上面设置 shadowsocks 的时候有设置这个端口，替换正确即可，`LOCAL_DNS2SOCKS_PORT` 为 DNS2SOCKS 监听的端口。

如此，我们就可以用以下命令启动 DNS2SOCKS 服务：

```
// 开启服务
/etc/init.d/dns2socks start
// 停止停止
/etc/init.d/dns2socks stop
// 重启服务
/etc/init.d/dns2socks restart
```

### 设置 pdnsd 启动脚本

同样的问题，pdnsd 也是没有启动脚本的，用同样的方法，在 `/etc/init.d/` 目录下新建文件 `pdnsd`，内容如下：

```
#!/bin/sh /etc/rc.common

START=65
NAME=pdnsd
DESC="proxy DNS server"

PDNSD_LOCAL_PORT=7453
LOCAL_DNS_PORT=1053

DAEMON=/usr/sbin/pdnsd
PID_FILE=/var/run/$NAME.pid
CACHEDIR=/var/pdnsd
CACHE=$CACHEDIR/pdnsd.cache


USER=nobody
GROUP=nogroup

start() {
       gen_cache

       start_pdnsd
}

stop() {
       kill `cat $PID_FILE` > /dev/null 2>&1
       rm -rf $PID_FILE
       rm -rf /var/pdnsd/
}

restart() {
       stop
       sleep 2
       start
}

gen_cache()
{
       if ! test -f "$CACHE"; then
               mkdir -p `dirname $CACHE`
               dd if=/dev/zero of="$CACHE" bs=1 count=4 2> /dev/null
               chown -R $USER.$GROUP $CACHEDIR
       fi
}

       
start_pdnsd() {
cat > /etc/pdnsd.conf <<EOF
global {
        perm_cache=2048;
        cache_dir="/var/pdnsd";
        pid_file = /var/run/pdnsd.pid;
        run_as="nobody";
        server_ip = 127.0.0.1;
        server_port = $PDNSD_LOCAL_PORT;
        status_ctl = on;
        query_method = udp_tcp;
        min_ttl=1d;
        max_ttl=1w;
        timeout=10;
        neg_domain_pol=on;
        proc_limit=2;
        procq_limit=8;
}
server {
        label= "dns";
        ip = 127.0.0.1;
        port = $LOCAL_DNS_PORT;
        timeout=6;
        uptest=none;
        interval=10m;
        purge_cache=off;
}
EOF
    $DAEMON -d -p $PID_FILE
}

```

并设置可执行权限：

```
chmod +X /etc/init.d/pdnsd
```

注意 pdnsd 启动脚本中的两个变量的配置，`LOCAL_DNS_PORT` 为 DNS2SOCKS 的端口，也就是上文中设置 DNS2SOCKS 时设置的 `LOCAL_DNS2SOCKS_PORT`，`PDNSD_LOCAL_PORT` 为 pdnsd 监听的端口，实际上我们最终使用的是这个端口的 DNS，没有直接使用 DNS2SOCKS 提供的 DNS，因为 pdnsd 可以缓存 DNS。

这里提一下 pdnsd 是作为一个转发器存在的，这里设置了 DNS2SOCKS 的 DNS 为 pdnsd 的上游 DNS，如果觉得 DNS2SOCKS 不好，想换成如 Google Public DNS 或者 OpenDNS 也是可以的，只需要更改一下启动脚本中 server 块的 IP 和端口，Google Public DNS 只有 53 端口提供，可以省略端口，OpenDNS 提供 5353 端口，非 53 端口 DNS 需要显式指定端口。另外，使用国外的 DNS 还是会被污染，需要指定 `global` 设置中的 `query_method` 为 `tcponly`，使用　TCP 方式请求 DNS 还是不被污染的。

### 配置 dnsmasq-china-list 并设置 dnsmasq
在 dnsmasq-china-list 主页下载 ZIP 并解压，将里面的 `.conf` 文件上传到路由器的 /etc/dnsmasq.d/ 目录，若无这个目录，需要新建一个。这些配置文件的主要作用是在文件中指定一些常用的域名使用后面的 DNS 来解析，可以看到里面都类似这样的规则：

```
server=/baidu.com/114.114.114.114
```

这样百度和其子域名就会使用 114.114.114.114 来解析，而不会走下面的配置。

接下来需要在/etc/dnsmasq.conf 中加上以下几行
```
// 设置 DNS 转发，路由器接到 DNS 解析请求后，会转给这个 DNS，也就是 pdnsd
server=127.0.0.1#7453
// 不使用 ISP 的 DNS，如果不设置这个，路由器会自己找 ISP 要一个 DNS 来解析域名
no-resolv
// 这里指定配置文件路径，就是上面的 dnsmasq-china-list 列表，这个列表的优先级略高，如果这里有指定 DNS，将不会走 127.0.0.1#7453
conf-dir=/etc/dnsmasq.d
```

### 更新 chnroute IP 列表

在路由器中（ssh 登录到路由器的那个界面）执行如下命令：

```
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chinadns_chnroute.txt
```

这个命令主要是去 [APNIC](https://www.apnic.net/) 拿所有的 IP 列表，并过滤出来中国大陆的 IP-CIDR 列表，生成一个文件在 `/etc/chinadns_chnroute.txt`，最后这个路径可以自己设置，不过这个路径要在 shadowsocks 界面配置，打开路由器后台，找到 服务-影梭，并切换到访问控制选项下，做如下设置：

![](https://ww3.sinaimg.cn/large/74681984gw1f79j17jq5dj20de0a8gmi)

这个路径就是刚才请求中国大陆 IP-CIDR 列表时保存的那个路径了。

至此配置完毕。

## 运行软件

```
/etc/init.d/shadowsocks start //启动 shadowsocks
/etc/init.d/dns2socks start //启动 DNS2SOCKS 服务
/etc/init.d/pdnsd start //启动 pdnsd 服务
/etc/init.d/dnsmasq restart // 重启 dnsmasq 服务，使新的配置文件生效
```

## 完结

到这里就完成了整个方案的配置，shadowsocks-libev 提供了透明代理，并提供 SOCKS5 代理 给 DNS2SOCKS 使用，路由器在收到 DNS 解析请求后转发给 pdnsd，pdnsd 向 DNS2SOCKS 请求 DNS，得到结果后缓存起来，也避免了 DNS 污染，再设置一些常用的域名走国内的 DNS 解析，这样一来，整个方案就走通了。

## 感谢

感谢各位开源项目维护者的辛苦劳动，有你们，才能让我们方便的浏览互联网上的任何信息。

## 修订记录

1. 2016 年 1 月 22 日，基于 openwrt-shadowsocks-liebev 2.3.x 第一次整理
2. 2016 年 8 月 28 日，基于 openwrt-shadowsocks-libev 2.4.8 重新修订

## 参考
1. [Shadowsocks + ChnRoute 实现 OpenWRT 路由器自动翻墙](https://cokebar.info/archives/664)
2. [WNDR4300 折腾 openwrt 记](http://dlmao.com/wndr4300-%E6%8A%98%E8%85%BE-openwrt-%E8%AE%B0.html)
3. [科学上网之五----DNS2SOCKS](http://www.bubuko.com/infodetail-624247.html)
4. [加速OpenWRT路由器的DNS解析 – pdnsd代替dnsmasq](https://cokebar.info/archives/734)

-- EOF --


