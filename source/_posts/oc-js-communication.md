---
title: 使用 JavaScriptCore 实现 JS和OC间的通信
tags:
  - JavaScriptCore
  - JavaScript
  - iOS
  - Objective-C
  - WebViewJavascriptBridge
categories:
  - 编程
date: 2016-06-18 20:44:26
---

iOS 开发中，我们时不时的需要加载一些 Web 页面，一些需求使用 Web 页面来实现可以更可控，如上线后也可以发布更新，修改 UI 布局，或者修复 bug，这些 Web 页面的作用不止是展示，很大一部分是需要和原生代码实现的 UI 和业务逻辑发生交互的，那么不可避免的，就需要用一些方法来实现 Web 页面（主要是 JavaScript）和原生代码之间的通信，在 JavaScriptCore 出现之前，很多项目都在用 [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 作为 Web 页面和原生代码之间的一个桥梁（bridge)，来传输一些数据和方法的调用，如 [Facebook Messenger](https://www.facebook.com/mobile/messenger)，[Facebook Paper](https://facebook.com/paper) 等。

<!-- more -->

## WebViewJavascriptBridge 原理简述
WebViewJavascriptBridge 的原理是通过自定义 `scheme`，在加载一个特定标识的URL（ `wvjbscheme://__BRIDGE_LOADED__`）时在 UIWebView 的代理方法 `webView:shouldStartLoadWithRequest:navigationType:` 中拦截 URL 并通过 UIWebView 的 `stringByEvaluatingJavaScriptFromString:` 方法执行一段 [JS](https://github.com/marcuswestin/WebViewJavascriptBridge/blob/master/WebViewJavascriptBridge/WebViewJavascriptBridge_JS.m)，这个 JS 文件中声明了一些变量和方法，在通讯中作为一个桥梁，那么怎么通讯呢？

### JS 调用 OC 中的方法

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

### OC 调用 JS 中的方法
OC 中调用 JS 的方法相对简单，因为 UIWebView 可以主动执行 JS，JS 中可以将需要监听的事件注册，同样是字符串作为标识，一个函数作为值，保存到一个全局对象中，在 OC 中主动执行特定的 JS 方法时，将数据封装成 JSON 字符串，传入标识符和数据，并遍历 JS 中保存 `handler` 的全局对象，看有没有注册相应的事件，如果有，根据 事件的名字得到一个函数并执行。实现了 OC 调用 JS 中的方法并向 JS 中传输数据。

## JavaScriptCore 时代的通讯

iOS 7 开始，苹果提供了一个叫作 JavaScriptCore 的框架，使用 JavaScriptCore 框架可以实现 OC 和 JS 的互相调用，而不需要依赖「桥」来实现，怎么通讯呢？

### OC 和 JS 间的 方法调用

#### OC 调用 JS 中的方法

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

![](https://ws3.sinaimg.cn/large/74681984gw1f89q4nyaewj20hs0vkjsc.jpg)


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


![](https://ws3.sinaimg.cn/large/74681984gw1f89q5066ydj20hs0vkt9j.jpg)


实现了 JS 调用 OC 中的方法。

是不是方便了很多？还可以传值。

代码见： https://github.com/cielpy/CPYJSCoreDemo/tree/v0.1

### OC 和 JS 间传值和获取方法返回值

#### JS 传值给 OC，并获取 OC 方法的返回值

如果需要将一个 JS 字符串传给 OC 中使用，可以做如下修改：

JS 中：

```
	<body>
    <button onclick="buttonClick()">点击这里</button>
	</body>
  
    
	function buttonClick() {
	var ocReturnValue = ocAlert("js ahhh")
	window.alert("oc return value is " + ocReturnValue)
	}
```

OC 中：

```
    self.context[@"ocAlert"] = ^JSValue *(JSValue *string){
        NSLog(@"%@", [string toString]);
        return [JSValue valueWithObject:@"oc ahh" inContext:weakSelf.context];
    };
```

这里需要注意一个比较隐蔽的循环引用问题，JSValue 对 JSContext 对象和它所管理的 JS 对象都是强引用，所以用 weak 规避一下循环引用问题。

点击网页中的按钮时，控制台输出：

```
2016-09-28 21:39:08.524 CPYJSCoreDemo[91467:881668] js ahhh
```

观察一下弹窗中的值，可以看到 block 中的值在弹窗中出现

![](https://ws3.sinaimg.cn/large/74681984gw1f89nhk9jh2j20hs0vk0tp.jpg)


#### OC 中传值给 JS，并获取 JS 方法的返回值

类型的，同样只需要做简单的修改，就可以把一个 OC 中的字符串传给 JS 中。

OC 中先获取 JS 方法，使用 `callWithArguments:` 方法调用 JS 的方法，参数以数组的方式传入：

```
    JSValue *funcValue = self.context[@"alertFunc"];
    JSValue * jsReturnValue = [funcValue callWithArguments:@[[JSValue valueWithObject:@"ahh" inContext:self.context]]];
    
    NSLog(@"js return value is %@", [jsReturnValue toString]);
```

JS 中：

```
 function alertFunc(string) {
   window.alert("这是一个JS中的弹框！" + string)
   return "js ahhh"
 }
```

点击原生界面中的 Button 时，弹窗如下：

![](https://ws3.sinaimg.cn/large/74681984gw1f89mti4barj20hs0vkq3v.jpg)

控制台打印如下：

```
2016-09-28 22:01:31.225 CPYJSCoreDemo[93696:906495] js return value is js ahhh
```

代码见： https://github.com/cielpy/CPYJSCoreDemo/tree/v0.2

### 获取不同类型的 JS 变量

JS 中的类型和 OC 中的类型不尽相同，但是基本上都有对应，`JSValue` 作为一个中间桥梁来转换不同类型的值：

```
   Objective-C type  |   JavaScript type
 --------------------+---------------------
         nil         |     undefined
        NSNull       |        null
       NSString      |       string
       NSNumber      |   number, boolean
     NSDictionary    |   Object object
       NSArray       |    Array object
        NSDate       |     Date object
       NSBlock (1)   |   Function object (1)
          id (2)     |   Wrapper object (2)
        Class (3)    | Constructor object (3)
```

可以把 OC 的各种类型的变量通过 `+ (JSValue *)valueWithObject:(id)value inContext:(JSContext *)context` 之类的方法，转成 `JSValue` 传给 JS 环境，也可以获取 JS 环境中的变量为一个 `JSValue` 对象，再通过诸如 `- (id)toObject;` 之类的方法转成 OC 类型的变量。

这里演示一下获取 JS 中的变量：

JS 中声明变量：

```
    var jsString = "js string"
    var jsBool = false
    var jsInt = 100
    
```

OC 中获取变量并转换：

```
    JSValue *jsString = self.context[@"jsString"];
    NSLog(@"js string var is %@", [jsString toString]);
    
    JSValue *jsBool = self.context[@"jsBool"];
    NSLog(@"js bool var is %d", [jsBool toBool]);
    
    JSValue *jsInt = self.context[@"jsInt"];
    NSLog(@"js int var is %d", [jsInt toInt32]);
```

控制台输出如下：

```
2016-09-28 22:40:09.420 CPYJSCoreDemo[96754:943326] js string var is js string
2016-09-28 22:40:09.420 CPYJSCoreDemo[96754:943326] js bool var is 0
2016-09-28 22:40:09.421 CPYJSCoreDemo[96754:943326] js int var is 100
```

代码见： https://github.com/cielpy/CPYJSCoreDemo/tree/v0.3

### OC 和 JS 间的异步回调

#### JS 回调给 OC

如果 JS 中有些工作需要异步完成，可能在完成后，通过回调把结果通知到 OC 中，修改代码如下：

OC 中：

```
	void (^block)(void) = ^{
		NSLog(@"js 回调了");
	};
    
	NSLog(@"按钮被点击了");
	JSValue *funcValue = self.context[@"alertFunc"];
	[funcValue callWithArguments:@[[JSValue valueWithObject:block inContext:self.context]]];
```

JS 中：

```
      function alertFunc(callback) {
        setTimeout(callback, 2000);
      }
```

首先创建一个 block 作为回调函数，之后通过 JSContext 拿到要执行的 JS 函数，并使用 `callWithArguments:` 调用该 JS 函数，调用时，把 block 包装成 JSValue 对象传入 JS 函数中，这个 block 在 JS 环境中将是一个函数，JS 中的函数，在被调用 JS 函数中，设置一个延时调用传入的回调函数（也就是 OC 中创建的那个 block），执行，并点击 OC 中的按钮点击事件以触发调用 JS 函数，观察控制台，打印输出出下：

```
2016-09-28 23:04:02.729 CPYJSCoreDemo[99296:971254] 按钮被点击了
2016-09-28 23:04:04.800 CPYJSCoreDemo[99296:971359] js 回调了
```

两秒钟后，block 被执行了。

#### OC 回调给 JS

那么反过来，如果 OC 中有些任务需要长时间处理，想异步回调给 JS 可以么？当然，可以，代码如下：


```
	function ocCallback() {
		window.alert("oc 回调了")
	}
    
	function buttonClick() {
		window.alert("js 按钮被点击了")
		ocAlert(ocCallback)
	}
```

声明一个函数，在调用 OC 的方法时传入这个函数，JS 中的工作就完成了，接下来在 OC中，修改如下：

```
    self.context[@"ocAlert"] = ^(JSValue *calback){
        sleep(2);
        [calback callWithArguments:nil];
    };
```

同样接收一个 JSValue 参数，但是这个 JSValue 对象其实是一个特殊的对象，是一个函数，打印这个 JSValue 对象如下：

```
Printing description of calback:
function ocCallback() {
      window.alert("oc 回调了")
    }
```

就是在 JS 中声明的那个函数了，一个函数类型的 JSValue 没有一个 toBlock 之类的方法来转换成 OC 中的 block 或者方法来执行，而是如上文中提到的 `callWithArguments:` 方法调用，同样，可以传入参数，只作通知用的话，一个没有参数的方法就可以达到目的了。

两秒钟后，JS 弹窗出下：

![](https://ws3.sinaimg.cn/large/74681984gw1f89pv00p8lj20hs0vkt9l.jpg)

代码见： https://github.com/cielpy/CPYJSCoreDemo/tree/v0.4

### JSExport 协议

除了上文经常用到的使用 Block 方式交互，还有另一种方式，JSExport 协议，我们可以定义一个继承于 JSExport 协议的协议，如下：

```
@protocol JSBridgeProtocol <JSExport>

- (NSInteger)add:(NSInteger)a b:(NSInteger)b;

@end
```

这里定义了一下简单的 add 方法，参数为两个 NSInteger 类型变量。

再定义一个类实现这个自定义的协议，如下：

```
@interface JSBridge : NSObject<JSBridgeProtocol>


@end


@implementation JSBridge

- (NSInteger)add:(NSInteger)a b:(NSInteger)b {
    return a + b;
}

@end
```

简单的实现一下这个协议，两个变量相加并返回结果。

在 OC 中，创建对象并赋值给 JS 环境，如下：

```
    self.context[@"ocObj"] = [[JSBridge alloc] init];
```

这样，在 JS 环境中就可以使用这个 `ocObj` 变量了。

在 JS 中，修改代码如下：

```
      function alertFunc() {
        window.alert("这是一个JS中的弹框！" + ocObj.addB(3,5))
      }
```

在 OC 中，获取这个 JS 函数并调用：

```
    JSValue *funcValue = self.context[@"alertFunc"];
    [funcValue callWithArguments:nil];
```

弹窗结果如下：

![](https://ws3.sinaimg.cn/large/74681984gw1f8aaciyetjj20hs0vkdgc.jpg)

可以看到 add 方法的运算结果。

通过这样的一个对象，我们可以定义一些复杂或者单独操作的一些业务逻辑，不用都挤在 ViewController 里。对代码的可维护性有一定的好处的。

代码见： https://github.com/cielpy/CPYJSCoreDemo/tree/v0.5


## 写在后面

嗯 ，一篇文章应该有个写在后面的。

以上只是 JavaScriptCore 框架的一个小的应用，使用 JavaSciptCore 框架结合 Objective-C 的动态性可以做很多事，比如著名的热修复框架 [JSPatch](https://github.com/bang590/JSPatch) 就是这两者的结合。

苹果添加的这些新特性可以给开发带来很多便利，就是不知道有坑没有，嗯，且爬且珍惜吧。

使用 JavaScriptCore 实现通讯的 demo 放到了 GitHub，地址如下：
https://github.com/cielpy/CPYJSCoreDemo

## 参考资料
* [JavaScriptCore by Example](https://www.bignerdranch.com/blog/javascriptcore-example/)
* [ JavaScriptCore初探](https://hjgitbook.gitbooks.io/ios/content/04-technical-research/04-javascriptcore-note.html)
* [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)
* [JavaScriptCore 使用](http://www.jianshu.com/p/a329cd4a67ee)


-- EOF --

