---
title: 使用 JavaScriptCore 进行 JS和OC之间的通信
tags:
  - JavaScriptCore
  - JavaScript
  - iOS
  - Objective-C
categories:
  - 编程
date: 2016-06-18 20:44:26
---

iOS 开发中，我们时不时的需要加载一些 Web 页面，一些需求使用 Web 页面来实现可以更可控，如上线后也可以发布更新，修改 UI 布局，或者修复 bug，这些 Web 页面的作用不止是展示，很大一部分是需要和原生代码实现的 UI 和业务逻辑发生交互的，那么不可避免的，就需要用一些方法来实现 Web 页面（主要是 JavaScript）和原生代码之间的通信，在 JavaScriptCore 出现之前，很多项目都在用 [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 作为 Web 页面和原生代码之间的一个桥梁（bridge)，来传输一些数据和方法的调用，如 [Facebook Messenger](https://www.facebook.com/mobile/messenger)，[Facebook Paper](https://facebook.com/paper) 等。

### WebViewJavascriptBridge 原理简述
WebViewJavascriptBridge 的原理是通过自定义 `scheme`，在加载一个特定标识的URL（ `wvjbscheme://__BRIDGE_LOADED__`）时在 UIWebView 的代理方法 `webView:shouldStartLoadWithRequest:navigationType:` 中拦截 URL 并通过 UIWebView 的 `stringByEvaluatingJavaScriptFromString:` 方法执行一段 [JS](https://github.com/marcuswestin/WebViewJavascriptBridge/blob/master/WebViewJavascriptBridge/WebViewJavascriptBridge_JS.m)，这个 JS 文件中声明了一些变量和方法，在通讯中作为一个桥梁，那么怎么通讯呢？

#### JS 中调用 OC 的方法

在 OC 中，实例化一个 `WebViewJavascriptBridge` 并调用 `registerHandler:handler:` 注册并监听一下事件，第一个参数是一个字符串，用来标识一个特定的事件，`handler` 是一个 block，方法内部将标识作为 `key`，`handler` 作为值保存。

```
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

当 JS 中需要调用 OC 的方法时，组装一个类似结构的数据，一个字符串作为标识，将需要传输的数据作为值并保存在一个全局数组中

```
var sendMessageQueue = [];
function _doSend(message, responseCallback) {
		if (responseCallback) {
			var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
			responseCallbacks[callbackId] = responseCallback;
			message['callbackId'] = callbackId;
		}
		// 主要就是这一行，将 message 保存到全局数组，供待会儿查询
		sendMessageQueue.push(message);
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

并触发一个特定的 URL（`wvjbscheme://__WVJB_QUEUE_MESSAGE__`），UIWebView 则在 `webView:shouldStartLoadWithRequest:navigationType:` 中拦截这个 URL，并执行一段 JS（`WebViewJavascriptBridge._fetchQueue();`）

```
function _fetchQueue() {
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    return messageQueueString;
}
```
查询 JS 中全局数组中的值，并转成 JSON 字符串返回，OC 中拿到 JSON 字符串，并解析，得到一个数组，遍历数组，根据数组中每个对象的 `handlerName` 查询 OC 中是否有注册这个事件，如果有注册，则根据 `handlerName` 取出保存在字典中的 block，并执行这个 block，block 可以接收一个 id 类型的参数，将 JS 全局数组中根据 `handlerName` 取出来的数据作为参数传入 block。这样就实现了从 JS 到 OC 中的数据传输。

#### OC 中调用 JS 的方法
OC 中调用 JS 的方法相对简单，因为 UIWebView 可以主动执行 JS，JS 中可以将需要监听的事件注册，同样是字符串作为标识，一个函数作为值，保存到一个全局对象中，在 OC 中主动执行特定的 JS 方法时，将数据封装成 JSON 字符串，传入标识符和数据，并遍历 JS 中保存 `handler` 的全局对象，看有没有注册相应的事件，如果有，根据 事件的名字得到一个函数并执行。实现了 OC 调用 JS 中的方法并向 JS 中传输数据。

### JavaScriptCore 时代的通讯

iOS 7 开始，苹果提供了一个叫作 JavaScriptCore 的框架，使用 JavaScriptCore 框架可以实现 OC 和 JS 的互相调用，而不需要依赖「桥」来实现，怎么通讯呢？

#### JavaScriptCore 中 OC 调用 JS 方法

在 JS 中定义一个方法

```
  function alertFunc() {
    window.alert("这是一个JS中的弹框！")
  }
```

在 `webViewDidFinishLoad:` 代理方法中，获取到 `JSContext` 对象

```
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    
    [context setExceptionHandler:^(JSContext *ctx, JSValue *expectValue) {
        NSLog(@"%@", expectValue);
    }];
    
    self.context = context;
}
```

在一个 button 的点击事件中可以根据 JS 定义的方法的名字获得一个 JSValue 类型对象，这个对象就是在 JS 中定义的方法，JSValue 对象通过调用 `callWithArguments:` 方法，执行这个 JS 方法。

```
- (IBAction)buttonClick:(UIButton *)sender {
    if (!self.context) {
        return;
    }
    
    JSValue *funcValue = self.context[@"alertFunc"];
    [funcValue callWithArguments:nil];
}
```

点击按钮时，效果如下。

![](https://o4zqhe4wo.qnssl.com/blog-img/1466263762855.png)

实现了 OC 中调用 JS 的方法。

#### JS 调用 OC 中的方法

在 OC 中，通过给 JSContext 的一个 `key` 赋值，值为一个 block，`key` 是 JS 中调用的方法的名字，代码如下：

```
    self.context[@"ocAlert"] = ^{
        // block 异步执行，如果涉及到 UI 的操作需要回到主线程操作
        dispatch_async(dispatch_get_main_queue(), ^{
            __strong typeof(weakSelf) strongSelf = weakSelf;
            UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"" message:@"这是OC中的弹框!" preferredStyle:UIAlertControllerStyleAlert];
            [alert addAction:[UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
                [alert dismissViewControllerAnimated:YES completion:^{
                    
                }];
            }]];
            [strongSelf.navigationController presentViewController:alert animated:YES completion:nil];
        });
    };
```

在 Web 页面中创建一个 button 并设置 button 的 onClick 事件调用 `ocAlert` 方法

```
<button onclick="ocAlert()">点击这里</button>
```

点击 Web 页面上的 button 按钮，效果如下

![](https://o4zqhe4wo.qnssl.com/blog-img/1466264261023.png)

实现了 JS 调用 OC 中的方法。

是不是方便了很多？

### 写在后面

嗯 ，一篇文章应该有个写在后面的。

以上当然只是 JavaScriptCore 框架的一个很小的应用，使用 JavaSciptCore 框架结合 Objective-C 的动态性可以做很多事，比如著名的热修复框架 [JSPatch](https://github.com/bang590/JSPatch) 就是这两者的结合。这里只是演示了 JS 和 OC 之间的方法调用，并没有传输数据，JavaScriptCore 框架是很容易的实现两者之间的数据传输的。具体做法可以参考参考资料。

苹果添加的这些新特性可以给开发带来很多便利，就是不知道有坑没有，嗯，且爬且珍惜吧。

### 参考资料
* [JavaScriptCore by Example](https://www.bignerdranch.com/blog/javascriptcore-example/)
* [ JavaScriptCore初探](https://hjgitbook.gitbooks.io/ios/content/04-technical-research/04-javascriptcore-note.html)
* [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)


