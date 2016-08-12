---
title: RAC 中的函数式编程 和 链式调用
tags:
  - ReactiveCocoa
  - 函数式编程
  - 响应式编程
  - 链式调用
categories:
  - 编程
date: 2016-06-11 21:27:00
---

更新 2016.8.13
这篇文章对函数编程的理解有误，望各位看官慎重参考！

---

最近用 [RAC](https://github.com/ReactiveCocoa/ReactiveCocoa) 比较多，RAC 被称为函数响应式编程（[FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming)）框架，各种教程中对响应式的特性介绍比较多，对函数式特性的介绍不怎么多，一直不太明白 RAC 的函数式编程的特性体现在哪里，今天想起来一个概念，链式调用，查了一下后才明白 RAC 中的函数式编程体现在哪里。

### 链式调用

链式调用的一个特点就是返回值为一个 block，block 其实是一个匿名函数，定义方法如下：

```
returnType (^blockName)(parameterTypes)
```

有参数，有返回值，可以用 `blockName()` 执行这个 block，当然需要传入定义好的参数。

block 可以作为一个方法的参数或者返回值，链式调用中，定义的方法返回值为一个 block，这个block的类型为

```
returenType (^)(parameterType)
```

如下方中的 `readBook` 方法，返回值为一个 `CPYPerson * (^)(NSString *)` 类型的block

```
- (CPYPerson * (^)(NSString *bookName))readBook{
    return ^ id (NSString *bookName) {
        NSLog(@"I read a book named %@",bookName);
        return self;
    };
};

- (CPYPerson * (^)(NSString *))drink{
    return ^ id (NSString *drink) {
        NSLog(@"I drink %@", drink);
        return self;
    };
};
```

调用方法如下


```
    CPYPerson *kevin = [[CPYPerson alloc] init];
    kevin.readBook(@"River town").drink(@"Cola");
```

当 readBook 方法被调用时，实际上是获取到了一个 `CPYPerson * (^)(NSString *)` 类型的 block，block 的调用方式就是括号里加参数，那么 `person.readBook(@"River town")` 就完成了 `readBook` 方法的调用，由于这个 block 返回了一个 `CPYPerson` 实例，我们可以用这个实例接着做其他事，这样的调用像一条链子一样，一口气做完一个对象想做的所有事都可以。

### RAC 中的函数式编程

在链式调用的方法中，我们看到可以将一个 block 作为一个返回值的形式，在需要的地方获取到这个 block 并执行它，block 中定义了所需要做的操作，那么可不可以把 block 当作参数，定义特定类型的 block，比如一个 `^id(id value)` 类型的 block，只需要这个 block 最终的返回值，至于这个 block 中对于这个 value做了什么操作并不关心，相当于一个工厂，只提供原料，至于中间的工序如何不关心，只关心最终得到一个产品就好了。

我们在使用 RAC 框架时经常会用到这一样个方法：

```
    [[ClassA doSomething] map:^id(id value) {
    
    }];
```

`doSomething` 方法返回值为一个 `RACSignal`，一个信号可以调用 `map` 方法，将这个信号中的 value 拿出来，并要求一个返回值，这个值可以根据 value 计算得到，至于怎么计算，可以根据业务定制，经过计算得到的值作为返回值返回，map 中会根据这个返回值生产一个新的信号。总结的来说，就是抽象出来需要做什么 (block 中的逻辑）。

这就是我现在理解的一点函数式编程的应用吧，函数（block) 作为一个参数存在，方法中可以根据这个拿到这个参数并执行，以得到返回值。

理解比较浅，有错误的地方还望指正。

在查找资料的过程中读到阮一峰老师的函数式编程的文章，[传送门](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html) 对照一下 RAC 框架中并不止有点体现了函数式编程的思想，不过还不是太懂，还在琢磨当中。

### 参考资料
* [函数式编程初探](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)
* [objc利用block实现链式编程方法](http://www.cnblogs.com/xiaobajiu/p/4772114.html)
* [How Do I Declare A Block in Objective-C?](http://fuckingblocksyntax.com/)
* [FP in Scala(二)：OOP和λ-演算的结合 Part IV：Stream(流)](http://bit.ly/1VSaCiA)

--EOF--

