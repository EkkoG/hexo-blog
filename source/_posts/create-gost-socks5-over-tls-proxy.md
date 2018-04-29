---
title: 使用 Gost 创建 SOCKS5-TLS 代理
tags:
  - Gost
  - 代理
categories:
  - 翻墙
date: 2018-04-29 20:29:21
---

首先安装 Docker，安装教程参考官方文档 https://docs.docker.com/install/

安装完成后，拉取 Gost 镜像

```
docker pull ginuerzh/gost
```

安装 [acme.sh](https://github.com/Neilpang/acme.sh)，用于申请证书，申请证书的过程参见 README，写的很详细，RSA 或者 ECC 证书都是可以的

申请完证书后安装证书，这里将证书安装在 `/etc/gost/`，这个路径在使用 docker 运行 Gost 时会用到

```
acme.sh --install-cert -d "domain.com" --fullchain-file /etc/gost/cert.pem --key-file /etc/gost/key.pem
```

创建 secrets.txt 文件，用于多用户认证，格式为一行一对用户名密码，空格隔开，如

```
user0 pass0
user1 pass1
```

这里为了方便将 `secrets.txt` 文件也放在 `/etc/gost/`，方便挂载，可以根据自己情况修改

运行 Gost

```
[sudo] docker run -d -p 8888:8888 --name gost -v /etc/gost/:/mnt ginuerzh/gost -L="socks5+tls://:8888?cert=/mnt/cert.pem&key=/mnt/key.pem&secrets=/mnt/secrets.txt"
```

其中 -v 参数用于挂载当前用户的 `/etc/gost/` 到 Gost 运行环境的 `/mnt` 目录，这样 Gost 程序就可以读取证书和用户信息文件，8888 是端口，这个根据自己的情况适当修改即可

运行后查看 Gost 运行状态，如果出现以下提示，就运行成功了


```
~ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS                    NAMES
0b95ed614bb1        ginuerzh/gost       "/bin/gost -L=socks5…"   21 minutes ago      Up 21 minutes            0.0.0.0:5877->5877/tcp   gost
```

参考资料：

- [acme.sh](https://github.com/Neilpang/acme.sh)
- [Gost 文档](https://docs.ginuerzh.xyz/gost/getting-started/)
- [简单记录下 Socks5 Over TLS, HTTPS and HTTP2 隧道代理的建立](https://medium.com/@rampage_router/%E7%AE%80%E5%8D%95%E8%AE%B0%E5%BD%95%E4%B8%8B-socks5-over-tls-https-and-http2-%E9%9A%A7%E9%81%93%E4%BB%A3%E7%90%86%E7%9A%84%E5%BB%BA%E7%AB%8B-8876d62bafc9)
- [Install Docker](https://docs.docker.com/install/)
- [v2ray socks5-tls 配置示例](http://dsh.li/blog/15220682257014.html)

