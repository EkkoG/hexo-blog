---
title: 理解 Autorelease Pool
tags:
  - iOS
  - 内存管理
categories:
  - 编程
date: 2016-05-11 00:26:47
---

之前被问到两个问题

* Autorelease Pool 中有什么对象？
* Autorelease Pool 中的对象在什么时候释放？  

之前对这两个问题不是很确定，查了点资料，其中苹果文档中有这样一段

> In a reference-counted environment (as opposed to one which uses garbage collection), an NSAutoreleasePool object contains objects that have received an autorelease message and when drained it sends a release message to each of those objects. Thus, sending autorelease instead of release to an object extends the lifetime of that object at least until the pool itself is drained (it may be longer if the object is subsequently retained). An object can be put into the same pool several times, in which case it receives a release message for each time it was put into the pool.

文档中说的很明确，在引用计数环境中（而不是垃圾回收机选）Autorelease Pool 中包含了收到 `autorelease` 消息的对象，并在 「倾倒」 Autorelease Pool 的时候，给其中的每个对象发送 `release` 消息。  

之前对 Autorelase Pool  是否影响局部变量不是太确定，文档显示，Autorelease Pool 只包含收到 `autorelease` 消息的对象，局部变量的生命周期应该有持有者管理，Autorelease Pool 只管 `autorelease` 对象。如果对象创建后没有 `release` 则造成内存泄露，Autorelease Pool  并不会自动帮开发者处理这些造成内存泄露的对象。

系统会维护一个 Autorelease Pool，还可以显式创建 Autorelease Pool 。

打开 Xcode 创建的工程中的 main.m，可以看到如下代码

```
int main(int argc, char * argv[])
{
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
这里可以看到整个 App 生命周期存在的这个 Autorelease Pool，没有显示创建Autorelease Pool 的话，那些标记为 `autorelease` 的对象就归它管了，那么这些对象在什么时候释放呢？不可能是在 App 生命周期结束的时候吧。这里引用 sunnyxxx 的博文

> 在没有手加Autorelease Pool的情况下，Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop

背后的机选涉及到了 runloop 和 Autorelease Pool 的底层实现，这里我理解不深，详情可以看 sunnyxxx 的博文 [http://blog.sunnyxx.com/2014/10/15/behind-autorelease/#Autorelease原理](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/#Autorelease原理) 有例子验证有分析，值得一看。

显式创建 Autorelease Pool 的语法如下

```
@autoreleasepool {

}
```

这个 Autorelease Pool 在执行完毕后会倾倒，大括号内的标记为 `autorelease ` 的变量会在这个时刻收到 `release` 消息，对象马上就会释放了。

关于其中的机制，参考资料中的两篇文章讲的足够详细，可以细细研究一番。

学海无涯，且行且学吧:)



* [黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
* [Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)
* [NSAutoreleasePool](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSAutoreleasePool_Class/index.html#//apple_ref/doc/uid/TP40003623)