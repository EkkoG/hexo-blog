---
title: 反向代理 GitHub Pages 并配置博客 HTTPS 访问
tags:
  - 博客
  - Nginx
  - HTTPS
  - 反向代理
categories:
  - 杂项
date: 2016-05-09 23:01:36
---

使用 GitHub 的 [Pages](https://pages.github.com/) 服务可以很方便的搭建个人博客，安全稳定，加上有很多开源的静态博客生成器 https://www.staticgen.com/ 大部分只需要在本地安装，生成网站，并发布到 Pages 的 repo 中即可，还是挺方便的。基本上配置好后，只管写文章就好。

静态博客有很多好处，不需要数据库，页面加载速度快（当然这根服务器有很大关系），大部分生成器支持 Markdown，自定义主题，可以部署在任何服务器上，只要可以在公网访问就可以了。GitHub 的 Pages 服务足够稳定也很方便，唯一的不足大概就是自定义域名的时候不能启用 HTTPS 访问吧，看样子 GitHub 一时半会儿是不会支持这个特性，那还是自己来吧。 [Let's Encrypt](https://letsencrypt.org/) 已经结束 Beta 很久，就想着把博客搬到 VPS 上，启动 HTTPS，前两天脑补了一个方案

![](https://o4zqhe4wo.qnssl.com/blog-img/1462799893920.png)

今天就试着动手折腾一下。

<!-- more -->

不过。。。没有用之前提到的那个方案，想着搭一个反向代理就可以了，关于反向代理可以看这里 http://z00w00.blog.51cto.com/515114/1031287 

反向代理的主要作用为

* 加密和SSL加速
* 负载均衡
* 缓存静态内容
* 压缩
* 减速上传
* 安全
* 外网发布

这里主要就是用了加密的作用。也可以做冗余部署，方法这里有介绍 https://blog.jamespan.me/2015/10/26/ha-deployment-for-blog/ 不过我暂时不准备做这个。

还是说回反向代理。

### 安装 Nginx，添加虚拟主机，配置反向代理
环境：Ubuntu 16.04
#### 安装 Nginx

安装最新稳定版的 Nginx 可以通过添加 PPA 的方式，命令如下

```
add-apt-repository ppa:nginx/stable
apt-get update
apt-get install nginx
```
#### 添加虚拟主机
安装比较新的 Nginx 版本的话，可以直接在 /etc/nginx/conf.d/ 目录下新建一个 .conf 配置文件就可以，Nginx 自动包含这个文件夹下的 .conf 文件，这里新建一个 domain.com.conf 文件，内容如下

```
 server {
        listen   80; ## 监听 IPv4 80 端口
        server_name domain.com;
}
```

配置完成后，运行 `service nginx reload` 命令，以加载刚刚添加的配置

在你使用的域名 DNS 解析服务提供商（DNSPod 等）处，添加一条 A 记录，指向 VPS 的 IP，配置完成后，稍微等待一下 DNS 记录生效，打开 domain.com ，可以看到 **Welcome to nginx!**，说明虚拟主机添加成功。

#### 配置反向代理

配置反向代理只需要修改 domain.conf 文件，添加一条 `proxy_pass` 规则就可以，修改完成后如下

```
 server {
        listen   80; ## 监听 IPv4 80 端口
        server_name domain.com;
        
         location / {
		        ## 这里用 HTTPS 比较好，代理服务器和源服务器间也是加密通讯
                proxy_pass https://yougihubname.github.io; 
                
                proxy_redirect     off;
       		    proxy_set_header   Host                       $host;
                proxy_set_header   X-Real-IP               $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}
```

配置完成后，再次运行 `service nginx reload` 命令，以加载刚刚修改的配置，完成后可以打开 domain.com 并刷新，看一下，如果是 https://yougithubname.github.io 的内容，刚反向代理配置成功。

关于多加的几个参数

> proxy_set_header Host $host 设置请求头的Host为反向代理服务器的Host
> 
> proxy_set_header X-Real-IP $remote_addr 设置请求头的X-Real-IP为客户端真实IP
> 
> proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for 把请求来源的IP添加到请求头的X-Forwarded-For字段
> 
> X-Forwarded-For:简称XFF头，它代表客户端，也就是HTTP的请求端真实的IP，只有在通过了HTTP代理或者负载均衡服务器时才会添加该项。 它不是RFC中定义的标准请求头信息，在squid缓存代理服务器开发文档中可以找到该项的详细介绍。 标准格式如下：X-Forwarded-For: client1, proxy1, proxy2。

到这里反向代理就配置完毕了。

### 配置 HTTPS

#### 申请证书

Let's Encrypt 将申请证书配置证书自动化，开源免费，好东西。社区有开发者制作了更简单的脚本，可以一键生成证书。

首先 clone 脚本

`git clone https://github.com/lukas2511/letsencrypt.sh.git`

cd 到脚本目录，新建一个 domain.txt，内容为你想要启用 HTTPS 访问的域名，举例

```
domain.com
```

复制一份 config.sh 到工具目录下

`cp docs/examples/config.sh.example  config.sh`

取消 WELLKOWN 的注释，并修改为

```
WELLKNOWN="/var/www/letsencrypt"
```

修改 Nginx 的 domain.com.conf 配置，增加验证所需的路径配置

```
server {
        listen   80; ## 监听 IPv4 80 端口
        server_name domain.com;
        
        location /.well-known/acme-challenge {
                alias /var/www/letsencrypt;
	    }
  
        location / {
		        ## 这里用 HTTPS 比较好，代理服务器和源服务器间也是加密通讯
                proxy_pass https://yougihubname.github.io; 
                
                proxy_redirect     off;
       		    proxy_set_header   Host                       $host;
                proxy_set_header   X-Real-IP               $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}
```
确保 `/var/www/letsencryp` 目录有写入权限，然后就可以运行脚本 `./letsencrypt.sh`，权限没问题的话，证书应该可以正常生成，生成的证书在 cert/domain.com/ 下。

#### 配置 HTTPS

修改 domain.com.conf 配置文件，增加一个监听 443端口的 server 配置，配置完成后 domain.com.conf 内容如下

```
server {
        listen   80; ## 监听 IPv4 80 端口
        server_name domain.com;
        
        location /.well-known/acme-challenge {
                alias /var/www/letsencrypt;
	    }
  
         location / {
		        ## 这里用 HTTPS 比较好，代理服务器和源服务器间也是加密通讯
                proxy_pass https://yougihubname.github.io; 
                
                proxy_redirect     off;
       		    proxy_set_header   Host                       $host;
                proxy_set_header   X-Real-IP               $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}

server {
		listen   443 ssl; ## listen for ipv4; this line is default and implied
        ssl on;

        server_name imciel.com;
        ## 这里路径为 fullchain.pem 文件的路径，文件可以随意放，确保位置正确即可
        ssl_certificate /etc/nginx/letsencrypt/live/domain.com/fullchain.pem;
        ## 这里路径 和 fullchain.pem 文件的路径作用一样
        ssl_certificate_key /etc/nginx/letsencrypt/live/domain.com/privkey.pem;
  
         location / {
		        ## 这里用 HTTPS 比较好，代理服务器和源服务器间也是加密通讯
                proxy_pass https://yougihubname.github.io; 
                
                proxy_redirect     off;
       		    proxy_set_header   Host                       $host;
                proxy_set_header   X-Real-IP               $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
}
```

配置完成后，再次运行 `service nginx reload` 命令，加载刚刚修改的配置，完成后可以试着访问 https://domain.com 看看运行结果，可以看到 https://yourgithubname.github.io 的内容的话，就是一切正常。配置完成。

### 写在后面

经过这些折腾，就开启了博客网站的 HTTPS 访问，最后一个痛点解决了。

通过反向代理，还可以做冗余部署，在一台服务器宕掉的时候可以访问另一台，甚至可以配置多台反向代理，通过添加多条 DNS 记录，保证反向代理可用，不过这样做有些成本，而且对于我这种小站来说似乎没有太大必要。

随后可能准备在 VPS 上按照之前脑补的方案将静态文件 clone 到 VPS 上，部署一个网站作为反向代理的另一个上游，这样在反向代理和上游之前消耗的时间应该会少一些，加快访问速度。今天试着做的时候遇到一权限问题，有时间了会把这个方案折腾一下。

写的很长，有点啰嗦，仅供大写参考吧。明白怎么做跟讲清楚怎么做还是有不小的差距，还要继续加油:)


### 参考资料

* [静态博客高可用部署实践](静态博客高可用部署实践)
* [Nginx 虚拟主机 VirtualHost 配置](http://www.neoease.com/nginx-virtual-host/)
* [How To Configure Nginx as a Web Server and Reverse Proxy for Apache on One Ubuntu 14.04 Droplet](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-web-server-and-reverse-proxy-for-apache-on-one-ubuntu-14-04-droplet)
* [HowTo: Use Nginx As Reverse Proxy Server](http://www.cyberciti.biz/tips/using-nginx-as-reverse-proxy.html)
* [Setting up Nginx Reverse Proxy server on Debian Linux](https://linuxconfig.org/setting-up-nginx-reverse-proxy-server-on-debian-linux)
* [【大型网站技术实践】初级篇：借助Nginx搭建反向代理服务器](http://www.cnblogs.com/edisonchou/p/4126742.html)
* [nginx反向代理配置](http://www.nginx.cn/927.html)
* [Nginx配置多域名反向代理](https://hustlibraco.github.io/2015/12/03/nginx%E9%85%8D%E7%BD%AE%E5%A4%9A%E5%9F%9F%E5%90%8D%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86/)
* [How To Create an SSL Certificate on Nginx for Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-nginx-for-ubuntu-14-04)
