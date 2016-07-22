---
title: RAC 中一个 retain cycle 问题
tags:
  - iOS
  - RAC
  - 内存管理
  - 引用循环
categories:
  - 编程
date: 2016-07-22 23:07:09
---

今天使用 RAC 注册通知时，遇到一个不是很明显的 `retain cycle` 问题，使用场景是在一个 `ViewController` 中注册一个通知，代码如下：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    [[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIApplicationDidBecomeActiveNotification object:nil] takeUntil:[self rac_willDeallocSignal]] subscribeNext:^(id x) {
        self.title = @"";
    }];
}
```

如果这样使用了，那么这个 `ViewController` 就释放不了了，为什么呢，翻了一下源码，看看 `rac_addObserverForName` 是怎么运行的。

`rac_addObserverForName` 是怎么运行的呢，通常我们如果需要在一个 `ViewController` 中监听一个事件的话会把 `ViewController` 自身作为一个监听者（observer)，RAC 中并不是这样，RAC 中 `rac_addObserverForName` 方法的代码如下：


```
@implementation NSNotificationCenter (RACSupport)

- (RACSignal *)rac_addObserverForName:(NSString *)notificationName object:(id)object {
	@unsafeify(object);
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		@strongify(object);
		id observer = [self addObserverForName:notificationName object:object queue:nil usingBlock:^(NSNotification *note) {
			[subscriber sendNext:note];
		}];

		return [RACDisposable disposableWithBlock:^{
			[self removeObserver:observer];
		}];
	}] setNameWithFormat:@"-rac_addObserverForName: %@ object: <%@: %p>", notificationName, [object class], object];
}

@end
```

这里创建了一个信号，并把这个信号返回，信号创建时需要一个 `block`，`block` 中包含中有一些操作，如注册通知等，但这个时候 `block` 中的操作并没有执行。

看 `RACSignal` 的 `createSignal` 方法怎么实现的，如下：

```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	return [RACDynamicSignal createSignal:didSubscribe];
}
```
其实是创建一个 `RACDynamicSignal` 对象，`RACDynamicSignal` 是 `RACSignal` 的子类，`RACDynamicSignal` 的 `createSignal` 方法怎么实现的呢？

```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
```

我们看到传进来的 `block` 赋值给一个叫 `didSubscribe` 的属性，那么这个 `block` 在什么时候被调用？往下看。

一个信号在调用 `subscribeNext` 时会创建 `subscriber`

```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
	NSCParameterAssert(nextBlock != NULL);
	
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
	return [self subscribe:o];
}
```

`RACSubscriber` 的初始化方法如下，这个 `nextBlock` 被赋值给 `RACSubscriber` 的一个属性

```
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
	RACSubscriber *subscriber = [[self alloc] init];

	subscriber->_next = [next copy];
	subscriber->_error = [error copy];
	subscriber->_completed = [completed copy];

	return subscriber;
}
```

`subscriber` 创建后，会调用信号的 `subscribe` 方法，并将得到的返回值返回。

```

`RACDynamicSignal` 重写了父类 `RACSignal` 的 `subscribe` 方法

- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
```

这里很重要的一个操作就是 `self.didSubscribe(subscriber)`，在 `createSignal` 时传入的 `block` 被执行了，也就是说在一个信号调用 `subscribeNext` 方法时，`createSignal` 中传入的 `block` 中的一大坨操作被执行，这里主要就是监听指定的通知。

如果接收到注册的通知，`subscriber` 被执行一个方法

```
[subscriber sendNext:note];
```

这个时候我们就熟悉了，是封装信号的那一套东西。这样整个流程就走通了。

那么我们来看一下几个关键的地方的对象引用关系。

`subscribeNext` 中创建的 `subscriber` 持有 nextBlock，nextBlock 捕捉 self 并引用。

信号创建时的 `block` 中 创建了一个监听者（observer)，`[NSNotificationCenter defaultCenter]` 实例引用了 `observer`，`observer` 引用 `usingBlock`，`usingBlock` 又捕捉到了 `subscriber`，由此两条引用链条串了起来，形成了如下的引用关系：

NSNotificationCenter 实例 -> observer -> usingBlock -> subscriber -> nextBlock -> self

信号创建时有一个操作

```
[RACDisposable disposableWithBlock:^{
			[self removeObserver:observer];
}]
```

这个意思是在信号完成时移除 `observer`，信号在什么时候完成呢，`takeUntil` 的信号参数完成时 `rac_addObserverForName` 中创建的信号完成，而 `takeUntil` 的参数是在 `ViewController` 释放时才可以完成，由此就可以看到：

1. `rac_addObserverForName` 中创建的信号在 `ViewController` 对象释放前，在信号创建时添加的 `observer` 将一直存在
2. 该 `observer` 间接引用 `ViewController` 对象，`ViewController` 对象引用计数始终不可能为 0，也就是释放不了

由此一个死循环产生，这个 `ViewController` 就释放不了了。

转了一大圈，产生了一个死循环的坑 :(

解决方法是挺简单的，使用 RAC 的方法注册通知时在 `block` 内使用 weak 对象来打破死死循环就好了，代码如下：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    @weakify(self);
    [[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UIApplicationDidBecomeActiveNotification object:nil] takeUntil:[self rac_willDeallocSignal]] subscribeNext:^(id x) {
        @strongify(self);
        self.title = @"";
    }];
}
```

由于 RAC 中大量使用 `block`，虽然可以写出更简单易读的代码，但是也隐藏了一些坑，使用时还是要小心。

最后，如有理解偏差，还望指正。

--EOF--




