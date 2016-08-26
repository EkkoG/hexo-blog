---
title: iOS UIView 圆角和加边框方式总结
tags:
  - iOS 开发
  - 圆角处理
categories:
  - 编程
date: 2016-08-27 22:51:50
---


iOS 中时常需要把某个 View 圆角处理，这样界面看起来更圆融，开发中用到过很多种方式做圆角处理，这里就总结一下。

### 最简单的

最简单的方式就是设置每个 View 自带的 layer 的属性即可

```
view.layer.cornerRadius = 8.0f;
```

![](https://ww3.sinaimg.cn/large/74681984gw1f77j6k27llj20hs0git90)

如果该 View 有子 View，会是这种状况

![](https://ww3.sinaimg.cn/large/74681984gw1f77j9loszdj20hs0gkdg2)



如果需要切掉除了保留部分以外的子 View，那么需要加上

```
view.clipsToBounds = YES;
```

![](https://ww3.sinaimg.cn/large/74681984gw1f77jaik660j20hs0hkt92)

如果需要边框，也简单，加上 layer 的边框设置就可以。

```
    view.layer.borderWidth = 10.0f;
    view.layer.borderColor = [UIColor yellowColor].CGColor;
```

但是如果细看，会发现边框有严重的黑边，特别是当 View 背景色比较深的时候。

![](https://ww3.sinaimg.cn/large/74681984gw1f77kapky3dj20uo0xuq3v)

这个方法好处就是简单，哪里需要圆角就在哪里设置就可以了。当然也是有缺点的，这种方式处理的圆角很模糊，特别是在视图比较小的时候，质量不高，当背景色比较深的时候有边框有黑边现象，而且如果使用了 `clipToBounds`，则会触发离屏渲染（这个一个很大的坑，有兴趣的话可以详细了解下），造成很严重的卡顿问题，特别是在 UITableViewCell 的子 View 中这样使用，掉帧会很严重。

![](https://ww3.sinaimg.cn/large/74681984gw1f7781hpzf7j20dm0b4mxd)

### 设置 mask layer

使用贝赛尔曲线，并根据其路径，得到一个「遮罩 layer」，将其设置为 View 自带 layer 的 mask，盖掉其他部分，剩余中间的部分。也可以达到圆角效果。

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

如果我们将这个 masklayer 加到自带 layer 上而不是设置为其 mask，代码如下：

```
    UIBezierPath* maskPath = [UIBezierPath bezierPathWithRoundedRect:v.bounds byRoundingCorners:UIRectCornerAllCorners cornerRadii:CGSizeMake(halfW, halfW)];
    
    CAShapeLayer* maskLayer = [CAShapeLayer layer];
    maskLayer.frame = view.bounds;
    maskLayer.fillColor = [UIColor whiteColor].CGColor;
    maskLayer.path = maskPath.CGPath;
    maskLayer.backgroundColor = [UIColor greenColor].CGColor;
    
    [view.layer addSublayer:maskLayer];
```

运行后看到效果如下：

![](https://ww3.sinaimg.cn/large/74681984gw1f77jz0biaxj20hs0j03yv)

一个只有四个角的 layer 盖到原来的 layer 上，达到圆角效果。

因为是盖住四角达到的效果，所以不用设置 `maskToBounds` 也可以去掉子 View 超出中心圆的部分，但是同样会触发离屏渲染。用的时候应当小心。


如果要加上边框，可以在 self.layer 上加一个圆环 layer 达到边框效果，代码如下：


```
- (void)setRoundedCorners:(UIRectCorner)corners borderWidth:(CGFloat)borderWidth borderColor:(UIColor *)borderColor cornerSize:(CGSize)size {
    UIBezierPath* maskPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds byRoundingCorners:corners cornerRadii:size];
    
    CAShapeLayer* maskLayer = [CAShapeLayer layer];
    maskLayer.fillColor = [UIColor blueColor].CGColor;
    maskLayer.frame = self.bounds;
    maskLayer.path = maskPath.CGPath;
    
    self.layer.mask = maskLayer;
    
    if (borderWidth > 0) {
        CAShapeLayer *borderLayer = [CAShapeLayer layer];
        
        // 用贝赛尔曲线画线，path 其实是在线的中间，这样会被 layer.mask（遮罩层)遮住一半，故在 halfWidth 处新建 path，刚好产生一个内描边
        CGFloat halfWidth = borderWidth / 2.0f;
        CGRect f = CGRectMake(halfWidth, halfWidth, CGRectGetWidth(self.bounds) - borderWidth, CGRectGetHeight(self.bounds) - borderWidth);
        
        borderLayer.path = [UIBezierPath bezierPathWithRoundedRect:f byRoundingCorners:corners cornerRadii:size].CGPath;
        borderLayer.fillColor = [UIColor clearColor].CGColor;
        borderLayer.strokeColor = borderColor.CGColor;
        borderLayer.lineWidth = borderWidth;
        borderLayer.frame = CGRectMake(0, 0, CGRectGetWidth(f), CGRectGetHeight(f));
        [self.layer addSublayer:borderLayer];
    }
```

我们会看到这里 borderLayer 的坐标很奇怪，这里解释一下，如果我们这样设置坐标，用以下方式添加 borderLayer，会是什么效果呢？

```
    UIBezierPath* borderPath = [UIBezierPath bezierPathWithRect:v.bounds];
    
    CAShapeLayer* borderLayer = [CAShapeLayer layer];
    borderLayer.path = borderPath.CGPath;
    borderLayer.fillColor = [UIColor clearColor].CGColor;
    borderLayer.strokeColor = [[UIColor blackColor] colorWithAlphaComponent:0.5].CGColor;
    borderLayer.lineWidth = 10;
    borderLayer.frame = v.bounds;
    [v.layer addSublayer:borderLayer];
```
效果是这样的：

![](https://ww3.sinaimg.cn/large/74681984gw1f77ku0l1mvj20hs0isq30)

如果放大了看，是这样的

![](https://ww3.sinaimg.cn/large/74681984gw1f77kuvjl4oj20uo0xsgm3)

边框刚好骑在了边界上，如果这时候我们在使用 maskLayer 做圆角，那骑在边界上的边框有外面一半将被吃掉，只剩下一半，所以要把 borderLayer 往里挪一半的边框的距离，避免让 maskLayer 吃掉外面那部分边框。

这个方法使用也是比较简单，并且可以使用贝赛尔曲线达到任意形状的「剪切」，也可以根据情况选择要「切」的角，比如如果只需要切右上和右下两个角，那么只需要改一下贝赛尔曲线：

```
    UIBezierPath* maskPath = [UIBezierPath bezierPathWithRoundedRect:view.bounds byRoundingCorners:UIRectCornerTopRight | UIRectCornerBottomRight cornerRadii:CGSizeMake(halfW, halfW)];
```

不好的是，上一种方式的缺点，这个方式也都有。:(

### 生成圆角背景图片方式

```
    CGFloat w = 200;
    UIView *v = [[UIView alloc] initWithFrame:CGRectMake(0, 0, w, w)];
    v.backgroundColor = [UIColor redColor];
    [self.view addSubview:v];
    v.center = self.view.center;
    
    UIImage *img = [UIImage imageWithColor:v.backgroundColor andSize:v.bounds.size];
    img = [img roundedWithBorderWidth:10 borderColor:[UIColor greenColor]];
    UIImageView *imageView = [[UIImageView alloc] initWithImage:img];
    [v insertSubview:imageView atIndex:0];
    v.backgroundColor = [UIColor clearColor];
```

效果如下：

![](https://ww3.sinaimg.cn/large/74681984gw1f77lwgcd3ij20hs0iimxv)

首先根据 View 的大小和背景色，生成一张图片，生成图片的方法如下：


```
@implementation UIImage (AGExtension)

+ (UIImage *)imageWithColor:(UIColor *)color andSize:(CGSize)size
{
    CGRect rect = CGRectMake(0.0f, 0.0f, size.width, size.height);
    UIGraphicsBeginImageContextWithOptions(rect.size, NO, [UIScreen mainScreen].scale);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(context, [color CGColor]);
    CGContextFillRect(context, rect);
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}
@end
```

再将这张图片圆角处理，并加上边框，最后创建一个 UIImageView，设置该 imageView 的 image 为圆角处理后的图片，并插入 View 的最底层，造成一种圆角处理的假象。

圆角处理的代码如下：

```
@implementation UIImage (AGExtension)

- (UIImage *)roundedWithBorderWidth:(CGFloat)borderWidth borderColor:(UIColor *)borderColor {
    CGFloat inset = 1;
    CGFloat width = self.size.width;
    CGFloat height = self.size.height;
    CGFloat cornerRadius;
    UIBezierPath *maskShape;
    if (width > height) {
        cornerRadius = height / 2.0 - inset;
        maskShape = [UIBezierPath bezierPathWithRoundedRect:CGRectMake((width-height)/2.0 + inset, 0 + inset, height-2*inset, height-2*inset) cornerRadius:cornerRadius];
    }else{
        cornerRadius = width / 2.0 - inset;
        maskShape = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0+inset, (height-width)/2.0+inset, width-2*inset, width-2*inset) cornerRadius:cornerRadius];
    }
    
    UIGraphicsBeginImageContextWithOptions(self.size, NO, [UIScreen mainScreen].scale);
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    
    CGContextSaveGState(ctx);
    CGContextAddPath(ctx, maskShape.CGPath);
    CGContextClip(ctx);
    
    CGContextTranslateCTM(ctx, 0, height);
    CGContextScaleCTM(ctx, 1.0, -1.0);
    CGContextDrawImage(ctx, CGRectMake(0, 0, width, height), self.CGImage);
    CGContextRestoreGState(ctx);
    
    if (borderWidth > 0) {
        [borderColor setStroke];
        CGFloat halfWidth = borderWidth / 2.0;
        UIBezierPath *border = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(halfWidth, halfWidth, self.size.width - borderWidth , self.size.width - borderWidth)];
        CGContextSetShouldAntialias(ctx, YES);
        CGContextSetAllowsAntialiasing(ctx, YES);
        CGContextSetLineWidth(ctx, borderWidth);
        CGContextAddPath(ctx, border.CGPath);
        CGContextStrokePath(ctx);
    }
    
    UIImage *resultingImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return resultingImage;
}

@end
```

这里使用贝赛尔曲线加边框的坐标和第二种方式的类似。

这种方法的好处是不会触发离屏渲染，生成图片和圆角都由 CPU 处理，且边框清晰，没有黑边，唯一的缺点应该就是不能处理子 View，在四角处的子 View 不会被「切」掉，毕竟是一个圆角背景造成的圆角假象，对子 View 没有什么影响力。

### 写在后面
暂时这三种方式都不能满足所有需求。

### 参考资料
* [How is the relation between UIView's clipsToBounds and CALayer's masksToBounds?](https://stackoverflow.com/questions/1177775/how-is-the-relation-between-uiviews-clipstobounds-and-calayers-maskstobounds)
* [iOS 高效添加圆角效果实战讲解](http://www.jianshu.com/p/f970872fdc22)



