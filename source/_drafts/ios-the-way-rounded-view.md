---
title: iOS 圆角方式总结
tags:
  - 无
categories:
  - 无
---

iOS 中时常需要把某个 View 圆角处理，这样界面看起来更圆融，开发中用到过很多种方式做圆角处理，这里就总结一下。

### 最简单的

最简单的方式就是设置每个 View 自带的 layer 的属性即可

```
view.layer.cornerRadius = 8.0f;
```

如果需要切掉除了保留部分以外的子 View，那么可以加上

```
view.clipsToBounds = YES;
```

这个方法好处就是简单，哪里需要圆角就在哪里设置就可以了。当然也是有缺点的，这种方式处理的圆角很模糊，质量不高，而且如果使用了 `clipToBounds`，则会触发离屏渲染（这个一个很大的坑，有兴趣的话可以详细了解下），造成很严重的卡顿问题，特别是在 UITableViewCell 的子 View 中这样使用，掉帧会很严重。

![](https://ww3.sinaimg.cn/large/74681984gw1f7781hpzf7j20dm0b4mxd)

### 设置 mask layer


```
UIView+CPYExtension.m

- (void)setRoundedCorners {
	    UIBezierPath* maskPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds byRoundingCorners:corners cornerRadii:size];
    
    CAShapeLayer* maskLayer = [CAShapeLayer layer];
    maskLayer.frame = self.bounds;
    maskLayer.path = maskPath.CGPath;
    
    self.layer.mask = maskLayer;
}
```
同样的，如果需要不显示除保留区域外的 View 的话，也需要设置 `view.clipsToBounds = YES;`，当然同样会触发离屏渲染，圆角效果基本相当，还是有点模糊。

### 参考资料
* [How is the relation between UIView's clipsToBounds and CALayer's masksToBounds?](https://stackoverflow.com/questions/1177775/how-is-the-relation-between-uiviews-clipstobounds-and-calayers-maskstobounds)

