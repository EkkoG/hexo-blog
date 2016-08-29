---
title: UIView 等宽或等高排列
tags:
  - iOS
  - Autolayout
categories:
  - 编程
date: 2016-08-29 23:17:04
---

在 UI 开发中，时不时需要把几个按钮或者 UILabel 并排着排列，在以前用坐标系布局时都是靠算，很麻烦还容易出错，现在 Autolayout 这么方便，能不能使用 Autolayout 完成这个布局工作呢？试试看。

Autolayout 的布局规则是写 View 与 View 间的相对关系，我们来分析一下，如果要使 3 个 View 在一个容器 View 中均分需要满足哪些条件。

1. 最左边 View 到父 View 一定边距
2. 所有 View 宽度相等
3. 一个 View 到下一个 View 有一定边距
4. 最右边 View 到父 View 一定边距

如果想要这些 View 在容器中完全均分，上述的边距都为 0 即可。

按照上面的想法来实践下，以下代码中的几个方法是对 NSLayoutConstraint 实例方法 `constraintWithItem:attribute:relatedBy:toItem:attribute:multiplier:constant:` 的简单封装，字面意思，详情实现见 [UIView+CPYLayout.m](https://github.com/cielpy/CPYFlexibleViewDemo/blob/master/CPYFlexibleViewDemo/UIView%2BCPYLayout.m)

以下方法为 NSArray 的一个扩展，假设我们将所需要布局的 View 放在了一个数组中，并都 `addSubview` 到了一个容器 View 中。

```
    UIView *container = [[UIView alloc] init];
    [self.view addSubview:container];
    container.backgroundColor = [UIColor redColor];
    
    [[[[container cpy_alignYToSuperview] cpy_toHeight:100] cpy_leftToSuperview:0] cpy_rightToSuperview:0];
    
    NSMutableArray *arr = [NSMutableArray array];
    for (int i = 0; i < 8; i++) {
        UIView *v = [[UIView alloc] init];
        v.backgroundColor = [UIColor blueColor];
        [container addSubview:v];
        [arr addObject:v];
        [v cpy_constraintEqualTo:NSLayoutAttributeWidth toView:v toAttribute:NSLayoutAttributeHeight constant:0];
    }
    
    [arr cpy_flexibleWidthWithMargin:10 spacing:10];
```

```
@implementation NSArray (cpy_layout)
- (void)cpy_flexibleWidthWithMargin:(CGFloat)margin spacing:(CGFloat)spacing {
    for (NSInteger i = 0; i < self.count; i++) {
        UIView *tmp = self[i];
        
        if (i == 0) {
            // 设置第一 View 到父 View 的左边距
            [tmp cpy_leftToSuperView:margin];
        }
        
        if (tmp == self.lastObject) {
            // 最后一个，设置到父 View 右边距
            [tmp cpy_rightToSuperView:margin];
        }
        else {
            // 不是最后一个，设置宽度相等和到下一个 View 的间距
            UIView *next = self[i + 1];
            [[tmp cpy_constraintEqualTo:NSLayoutAttributeWidth toView:next toAttribute:NSLayoutAttributeWidth constant:0] cpy_rightToLeftToView:next constant:spacing];
        }
        // 设置居中
        [tmp cpy_alignYToSuperView];
    }
}
@end
```

效果如下：

![](https://ws3.sinaimg.cn/large/74681984gw1f7b1rgrf8bj20hs09yt8s)

也可以不设置容器 View 的宽度或者边距约束，让它根据子 View 的布局后的大小自行调整大小，但是这样的话，就需要设置子 View 到容器的边距约束，不然没办法完成布局。代码如下：

```
    UIView *container = [[UIView alloc] init];
    [self.view addSubview:container];
    container.backgroundColor = [UIColor redColor];
    
    // 只设置了居中，没有设置边距和宽高约束
    [container cpy_centerToSuperview];
    
    NSMutableArray *arr = [NSMutableArray array];
    for (int i = 0; i < 8; i++) {
        UIView *v = [[UIView alloc] init];
        v.backgroundColor = [UIColor blueColor];
        [container addSubview:v];
        [arr addObject:v];
        
        // 这里设置了宽度，和到父 View 的上下边距
        [[[v cpy_toWidth:20] cpy_topToSuperview:0] cpy_bottomToSuperview:0];
        [v cpy_constraintEqualTo:NSLayoutAttributeWidth toView:v toAttribute:NSLayoutAttributeHeight constant:0];
    }
    
    // 这里会设置子 View 到父 View 的左右边距
    [arr cpy_flexibleWidthWithMargin:0 spacing:10];
```

这样约束条件都满足了，运行看一下效果：

![](https://ws3.sinaimg.cn/large/74681984gw1f7b22fwniej20hs064t8o)

同理，也可以垂直等高排列：

```
    UIView *container = [[UIView alloc] init];
    [self.view addSubview:container];
    container.backgroundColor = [UIColor redColor];
    
    [container cpy_centerToSuperview];
    
    NSMutableArray *arr = [NSMutableArray array];
    for (int i = 0; i < 8; i++) {
        UIView *v = [[UIView alloc] init];
        v.backgroundColor = [UIColor blueColor];
        [container addSubview:v];
        [arr addObject:v];

        [[[v cpy_toWidth:20] cpy_leftToSuperview:0] cpy_rightToSuperview:0];
        [v cpy_constraintEqualTo:NSLayoutAttributeWidth toView:v toAttribute:NSLayoutAttributeHeight constant:0];
    }
    
    [arr cpy_flexibleHeightWithMargin:0 spacing:5];
```
效果如下：

![](https://ws3.sinaimg.cn/large/74681984gw1f7b3ek2c1gj20hs0gemx7)

嗯，还不错 :)

这样就不用算一大堆坐标，只需要设置几条约束就可以了，并且这样如果需要添加或者去掉一个 View 也比较方便。

完。

完整 Demo 见： https://github.com/cielpy/CPYFlexibleViewDemo


## 参考资料
1. [iOS Autolayout: two buttons of equal width, side by side](https://stackoverflow.com/questions/28148843/ios-autolayout-two-buttons-of-equal-width-side-by-side)

-- EOF --


