---
title: 制作一个查询 ip 的 Alfred Workflow
tags:
  - Alfred
  - Workflow
categories:
  -  杂项
date: 2016-03-27 15:37:35
---

Alfred 真是 Mac 平台的神器，可以很方便的执行一些任务，比如查询 IP，我用了 [kodango](https://twitter.com/dangoakachan) 的 IP 查询 Workflow [Lookup IP](https://github.com/dangoakachan/lookup-ip)，可以很方便的查到当前的 IP，效果如下
![](http://ww1.sinaimg.cn/large/74681984gw1f2bgrwq5jrj20eu033aa3.jpg)
但是我有另一个需求，我在路由器上部署了透明代理，某些时候不稳定要切换，切几次就忘了当前用的哪个，想查一下的时候，之前我的是打开 [https://www.astrill.com/what-is-my-ip-address.php](https://www.astrill.com/what-is-my-ip-address.php)，因为我的透明代理方案是国内外分流，这个网站在国外，而且被墙，所以打得开就是代理没问题，还可以看一下当前的代理 IP，但是这样就比较麻烦，要打开浏览器并且打开网页，本着能少做一个动作绝不多做的原则(哪来的原则...)，想起来 Alfred+Workflow 这个神器组合，还有自己早就忘的差不多的 Python 技能，仿照着 dangoakachan 的 Workflow，做了一个查询国外代理 IP 的 Workflow，效果如下
![](http://ww3.sinaimg.cn/large/74681984gw1f2bgy73dqkj20f0036t8q.jpg)
还是挺方便的。

Workflow 的制作方法，还是挺简单的，流程就是，查询数据-封装数据为指定格式-返回给 Alfred 就可以了，看 dangoakachan 的 Workflow 源码的时候，看到用了一个封装好的库，https://github.com/deanishe/alfred-workflow ，这个库就是把 Workflow 的制作进制了封装，发送请求，解析数据，封装数据，返回数据给 Alfred 一条龙服务，还是相当方便的。而且文档齐全，按照文档一步一步做，基本没有什么问题。

我主要参考了这个教程 http://www.deanishe.net/alfred-workflow/tutorial_1.html
简单描述一下我制作的过程如下
1. 创建新的 Workflow
![](http://ww3.sinaimg.cn/large/74681984gw1f2bh70k943j20f80blgms.jpg)
里面的必填项是 Workflow Name，其他诸如描述等可填可不填
2. 安装 alfred-workflow
首先在 Finder 中找到 Workflow 的位置，右键 workflow, Show in Finder 就可以了
![](http://ww1.sinaimg.cn/large/74681984gw1f2bhbfha9dj20hw0ayq5m.jpg)
根据教程里的安装 alfred-workflow 有两种方法
首先从 Github 下载最新的包 https://github.com/deanishe/alfred-workflow/releases
方法一是把 zip 直接放到 Workflow 目录，并在 Python 脚本最上方加上 `sys.path.insert(0, 'alfred-workflow-X.X.X.zip') `
方法二是解压 zip，把解压出来的 workflow 文件夹放进 Workflow 目录
我选择了第二种，简单直接嘛
到这就安装好了
3. 写代码
参照 http://www.deanishe.net/alfred-workflow/tutorial_1.html#id6
只需要改动 main 方法部分，把数据改成自己想要的就可以
我这里用了 http://ip-api.com/docs/api:json 的 API
由于 alfred-workflow 的封装，其实自己是不需要做什么工作的，最终修改出来的代码如下
```
import sys
from workflow import Workflow, ICON_WEB, web

API = "http://ip-api.com/json"

def main(wf):
    data = web.get(API).json()
    if data['status'] == 'success':
        wf.add_item(title=data['query'],
                     subtitle=data['timezone'],
                     icon=ICON_WEB)
    wf.send_feedback()


if __name__ == '__main__':
    # Create a global `Workflow` object
    wf = Workflow()
    # Call your entry function via `Workflow.run()` to enable its helper
    # functions, like exception catching, ARGV normalization, magic
    # arguments etc.
    sys.exit(wf.run(main))
```

main 方法中，首先拿到 data，由于已知这个 API 返回的是 json 格式的数据，所以直接调用了 alfred-workflow 的 json() 方法，处理成 json 类型数据，判断了请求是否成功，拿到自己想显示的字段，组装成一个 item 并加入到 workflow 实例中，最后 send_feedback 就可以了。

到这里就折腾完了，再来看一下效果
![](http://ww3.sinaimg.cn/large/74681984gw1f2bgy73dqkj20f0036t8q.jpg)
嗯，还不错嘛。

Workflow 源码见 https://github.com/cielpy/lookup-ip

嗯，没事动动手写些小玩意，还是挺不错的。:)
