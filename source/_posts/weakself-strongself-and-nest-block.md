---
title: 关于 weakSelf 和 strongSelf
tags:
  - ios
  - block
categories:
  - 编程
date: 2017-12-23 00:45:17
---

### 关于 weakSelf 和 strongSelf

block 带来了一定程度的便利性，在作为回调使用时，时常会遇到内存问题，即「引用循环」，这在 Objc 中是一个绕不开的话题。

试着看如下场景：


```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	v.callbackBlock = ^{
		[self doSomething];
	}
}
```

当 `viewDidLoad` 方法执行时，创建一个 block 并赋值给对象 `v` 的 `callbackBlock` 属性，`callbackBlock` 捕捉 `self`，`self` 持有 `self.view`, v 在 `addSubview` 后成为 `self.view` 的子 view 而被 `self.view` 持有，这样就形成了一个引用循环，如下：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fmpzcj8g1bj30bd05zq30.jpg)

<!-- more -->

而如果 `CustomView` 对象不被创建时，不会造成引用循环：

```
DemoViewController.m

- (void)viewDidLoad {
	[super viewDidLoad];
}

- (void)setup {
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	v.callbackBlock = ^{
		[self doSomething];
	}
}
```

我们可以打破这个循环


```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	__weak __typeof__(self) weakSelf = self;
	v.callbackBlock = ^{
		[weakSelf doSomething];
	}
}

```

![](https://ws4.sinaimg.cn/large/006tNc79gy1fmpzibrsxfj30ax05tt8r.jpg)

`callbackBlock` 不直接引用 `self`，影响 `self` 的释放，由此打破循环，所有对象可以正常释放。

而当下面情况出现时，行为开始变得奇怪起来：

```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	__weak __typeof__(self) weakSelf = self;
	v.callbackBlock = ^{
		[weakSelf doSomething];
		[weakSelf doAnoterThing];
	}
}
```

若 `callbackBlock` 开始执行时，`DemoViewController` 对象已经释放，`callbackBlock` 中 `weakSelf` 始终为 `nil`，不会造成什么影响，若在 `callbackBlock` 开始执行时 `DemoViewController` 对象还存在，在 `doSomething` 时，`weakSelf` 一定不为 `nil`，而在 `doAnoterThing` 时就不一定了，我项目中出现复杂的场景时会造成 `callbackBlock` 执行行为不一致，导致一些奇怪的问题，这个时候我可以利用引用计数的特性来解决，方案变成了下面：

```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	__weak __typeof__(self) weakSelf = self;
	v.callbackBlock = ^{
		__typeof__(self) strongSelf = weakSelf;
		[strongSelf doSomething];
		[strongSelf doAnoterThing];
	}
}
```

在 `callbackBlock` 开始执行时，产生一个对 `weakSelf` 的强引用，这样在 `callbackBlock` 的作用域范围内，`self` 不可能被释放，这样 `callbackBlock` 的执行行为在所有情况下是一致的，避免产生一些奇怪的问题。

那么如果 block 出现嵌套呢？

```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	__weak __typeof__(self) weakSelf = self;
	v.callbackBlock = ^{
		__typeof__(self) strongSelf = weakSelf;
		[strongSelf doSomething];
		[strongSelf doAnoterThing];
		obj.objCallbackBlock = ^{
			[strongSelf doObjSomething];
			[strongSelf doObjAnotherThing];
		}
	}
}
```

`objCallbackBlock` 捕捉的是 `strongSelf`，而 `strongSelf` 实际上还是指向对 `self` 的弱引用，`strongSelf` 可以保证在 `callbackBlock` 执行期间不释放，但是在 `objCallbackBlock` 执行期间并不能保证一定存在，所以还是会有执行不一致问题，所以在 `objCallbackBlock` 中再次对 `weakSelf` 进行一次强引用：

```
DemoViewController.m

- (void)viewDidLoad {
 	[super viewDidLoad];
	CustomView *v = [[CustomView alloc] init];
	[self.view addSubview:v];
	__weak __typeof__(self) weakSelf = self;
	v.callbackBlock = ^{
		__typeof__(self) strongSelf = weakSelf;
		[strongSelf doSomething];
		[strongSelf doAnoterThing];
		obj.objCallbackBlock = ^{
			__typeof__(self) strongSelf = weakSelf;
			[strongSelf doObjSomething];
			[strongSelf doObjAnotherThing];
		}
	}
}
```

以保证所有 block 的执行行为一致。

### 参考资料

- [@weakify/@strongify in nested blocks](https://github.com/jspahrsummers/libextobjc/issues/45)
- [I finally figured out weakSelf and strongSelf](https://dhoerl.wordpress.com/2013/04/23/i-finally-figured-out-weakself-and-strongself/)
- [深入分析 Objective-C block、weakself、strongself 实现原理](https://www.jianshu.com/p/a5dd014edb13)

