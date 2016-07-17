---
title: 折腾记：上传图片到微博图床 Alfred Workflow 制作
tags:
  - 图床
  - 博客
  - Alfred
  - Workflow
  - Python
  - 博客
  - 工具
categories:
  - 杂项
date: 2016-07-17 14:51:45
---

用 Markfown 写博客经常需要上传图片，最开始的时候用[微博图床 Chrome 插件](https://github.com/Suxiaogang/WeiboPicBed)，如果需要上传图片，需要截图或者复制文件，打开 Chrome，找到插件，打开插件，粘贴，得到 URL，在 Markdown 编辑器中插入，步骤烦琐，大部分需要鼠标操作，效率低。后来发现了一个 Alfred Workflow，[markdown-img-upload](https://github.com/tiann/markdown-img-upload)，这个 workflow 可以读取剪切板中的图片并上传到七牛，得到 URL 后复制，整个上传过程精简了许多，但是有一个问题，如果用自己的域名，需要备案，如果用七牛提供的测试域名，哪天不提供了，或者换了咋办，服务稳定性（这里指域名）是个问题。昨天又发现了一个图床服务，[天喵是个好图床](https://www.tmall.casa/file/)，用的是天猫的资源，有 Alfred workflow，需要激活 File Action 才可以上传，另外，不知道天猫的接口，上传图片要经过这个网站，倒闭了怎么办。综上，三个服务的优缺点如以下图表：

|  | Alfred workflow 上传到七牛 | Alfred workflow 上传到天猫图床 | 浏览器插件上传到新浪 |
| --- | :-: | :-: | :-: |
| CDN | 有 | 有 | 有 |
| 复制粘贴上传 | 支持 | 不支持 | 不支持 |
| 直接调用平台 API | 是 | 否 | 是 |
| 需要登录 | 否 | 否 | 是 |
| API 稳定性 | 域名可能变 | 由于不是直播调用平台API，中间平台可能倒闭 | 公用上传接口可能停用 |

### 想要的工作流

我想要一个怎样的工作流呢？

1. 可以复制粘贴上传
2. 上传完成后将 URL 复制到剪切板
3. 不需要登录（或者可以自动登录）
4. API 尽量稳定

基于第 4 点，上传到天猫图床最不稳定，且找不到公共 API 文档，排除
同样基于第 4 点，七牛给的测试域名不稳定，排除

目前来说，最好的组合是，使用新浪的公用接口，读取剪切板图片并上传，生成 URL 复制到剪切板

### 制作

方案想好了，具体就需要查相关的资料和有没有什么资源可以利用。
新浪的图片上传服务地址如下：http://picupload.service.weibo.com/interface/
这个服务需要登录才可以使用，想了一下，列出来需要解决的问题如下：

1. 模拟登录新浪微博并保存 cookie
2. 读取剪切板，拿到图片数据
3. 提交表单到新浪
4. 根据返回结果，生成图片 URL

再想想我身上的技能包，感觉是时候捡起来好久没用的 Python 技能了。

已知的可以解决问题的的方案有如下：

* 第 1 点可以参考 https://github.com/trytofix/hexo_weibo_image/blob/master/weibo_util.py
* 第 2 点可以参考七牛 workflow 的读取如何操作
* 上传操作和 URL 生成参考资料同第 1 点

基本上整个流程的重点部分是有东西可以参考的，那么就动手吧。

#### util 模块制作

参考 [markdown-img-upload](https://github.com/tiann/markdown-img-upload) 中的 util 模块，搞了一个类似的 util 模块，供整个 workflow 通知用户使用。

#### 提示输入配置信息以及读取配置

微博需要登录，这样就需要得到用户名密码，Alfred 通过交互输入得到参数没有查到方法，想来也比较麻烦，简单点可以像 markdown-img-upload 一样，上传前检查 cookie 状态，如果没有 cookie 说明没有登录，通过让用户编写简单的配置文件，在下次上传时读取配置文件中的用户名和密码这样的方式，进行模拟登录，同样参考 markdown-img-upload 中的提示用户填写配置文件的的逻辑和读取逻辑，得到了配置模块和读取用户信息的方法。

#### 模拟登录微博

得到用户名和密码后，就可以登录了，hexo_weibo_image 中的微博登录是包含在上传过程中的，由于要做的这个 workflow 需要读取用户配置，这样就需要一个接口用来登录和检查登录状态，给 hexo_weibo_image 的微博模块添加了两个方法，一个检查登录状态，一个专门用来登录，得到了微博模块 [weibo.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/master/weibo.py)。

#### 得到剪切板文件路径

markdown-img-upload 中有一个 clipboard 模块，展示了怎么样读取剪切板中的文件，以及压缩文件，转换格式等，而上传到微博相对简单，只需要得到一个文件路径即可，上传中对图片数据 base64 编码即可上传，截取了 clipboard 模块中的将剪切板中的文件保存到临时目录并读取文件的逻辑，得到一个简化版的 clipboard 模块[clipboard.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/master/clipboard.py)，这里踩了一个坑，临时文件写入后会马上自动删除，如果只返回临时文件的路径，则在上传时文件已经不在了，所以这里需要把文件返回，需要上传时，取文件路径即可。

#### 整合各模块

几个模块都准备好了，整合各模块就可以达成需求了。
整个流程就是


```flow
st=>start: 开始
e=>end: 得到 URL并拷贝到剪切板
shortcut=>operation: 键入快捷键
hasLogin=>condition: 检查登录状态
config=>subroutine: 配置
hasConfigFile=>condition: 配置文件存在
readConfig=>operation: 读取配置
generateConfig=>operation: 生成初始配置文件
openEditor=>operation: 打开编辑器
login=>operation: 登录
loginSuccess=>condition: 登录成功
deleteConfig=>operation: 删除配置文件
getFilePath=>operation: 获取文件路径
upload=>operation: 上传图片
uploadSuccess=>condition: 上传成功
pasteURL=>operation: 复制 URL 到剪切板

st->shortcut->hasLogin
hasLogin(yes)->getFilePath->upload->uploadSuccess
uploadSuccess(yes)->e
uploadSuccess(no)->shortcut
hasLogin(no)->config->hasConfigFile
hasConfigFile(yes)->readConfig->login->loginSuccess
loginSuccess(yes)->deleteConfig->getFilePath(right)
loginSuccess(no)->shortcut
hasConfigFile(no)->generateConfig->openEditor
```


最终得到 main 模块 [main.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/b4bcfd440641c19859a2903bd09405234d212810/main.py)


### 问题来了
这样一样，整个流程就走通了，这样得到 URL 后，仅将 URL 复制到了剪切板，能不能复制一个 Markdown 格式的字符串并自动填入图片链接呢？那样岂不是更省事了，目前的流程复制操作包含在了上传完成操作里，没办法区分，那么就把上传独立出来，上传模块负责登录状态检查和上传，得到 URL 后返回 URL，失败则返回空，最终得到一个上传模块 [upload.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/master/upload.py)

想得到普通的只有图片地址时，调用 [get_image_url.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/master/get_image_url.py) 模块，会将 URL 复制到剪切板。

如果想得到 Markdown 格式的，则调用 [get_markdown.py](https://github.com/cielpy/WeiboPictureWorkflow/blob/master/get_markdown.py)，会将 一个 Markdown 格式的字符串复制到剪切板。

### 写在后面

到这里 workflow 就制作完毕了，我设置了两个快捷键。
![](https://ww3.sinaimg.cn/large/74681984gw1f5xbkf7f9oj20nh0geabb
)

Ctrl + V 只复制图片 URL，Ctrl + B 复制 Markdown 格式字符串并插入图片 URL。

这样在需要插入图片时就不需要那么多烦琐的步骤了，复制图片或者截图，根据需求按相应的快捷键，得到图片 URL 或者 Markdown 字符串，粘贴就好了。:)

--EOF--


