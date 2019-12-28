---
title: 使用 Autolayout 实现动态高度 UITableViewCell
tags:
  - Autolayout
  - 动态高度Cell
categories:
  - 编程
date: 2016-09-05 21:21:53
---

在 Frame 布局时代，如果要实现一个动态高度的 Cell，需要给 Cell 绑定数据后，根据内容的展示情况计算得到 Cell 的高度，最好再加一个高度缓存，因为这种计算在 UITableView 滑动时代价还是比较高昂的。那么配合 Autolayout 可不可以实现动态高度 Cell 呢？当然是可以的。

<!-- more -->

分析一下，和之前的一篇文章《[几个 Autolayout 技巧](https://imciel.com/2016/08/23/autolayout-tips/)》中的分析类似，如果约束可以限制内容的显示范围，在绑定数据后，Autolayout 理应得到整个父 View 的显示范围，这样就得到了父 View 的高度，虽然 UITableViewCell 有点特殊，但是它作为一个 View 应该也适用这样的道理。

试着写了个 Demo，Cell 类似微博的时间线，最上方是一个用户名 Label，往下依次是标题，内容，和一个按钮，最终效果如下：

![](https://i.imgur.com/oCDSoWs.jpg)


内容是一个随机长度的字符串，按照之前的想法，这样一个简单的 Cell 的约束应该怎么样设置呢。

首先用户名 Label 到 Cell 左边一定距离，上面一定距离，右边一定距离，一行显示，这样有上左右三个边距和一个高度，确定了用户名 Label 的显示范围。

标题 Label 到 Cell 左边一定距离，上方到用户名 Label 一定距离，到 Cell 右边一定距离，多行显示。同样由于上方的高度在有内容时确定，标题 Label 相当于有了上左右边距和一个高度，在坐标系中的位置也就确定了。

内容 Label 同标题一样的道理。

最终剩下一个按钮，主要是想确认父 View（也就是 Cell）的高度，所以按钮到内容有一个固定边距，下方到父 View 有一个固定边距，居中对齐，宽度根据内容自由延伸，对高度没有影响。

最终各个 Cell 的约束主要如下：

```
    // 用户名
    [[[self.nameLabel cpy_topToSuperview:8] cpy_leftToSuperview:8] cpy_rightToSuperview:8];
    // 标题
    [[[self.titleLabel cpy_topToView:self.nameLabel constant:8] cpy_leftToSuperview:8] cpy_rightToSuperview:8];
    // 内容
    [[[self.contentLabel cpy_topToView:self.titleLabel constant:8] cpy_leftToSuperview:8] cpy_rightToSuperview:8];
    // 按钮
    [[[[self.retweetButton cpy_topToView:self.contentLabel constant:8] cpy_bottomToSuperview:8] cpy_toWidth:80] cpy_alignXToSuperview];
```

其中的自定义方法为到 View 或者父 View 的间距，完整的代码见：https://github.com/cielpy/CPYDynamicCellDemo/blob/v0.1/CPYDynamicCellDemo/CPYDemoTableViewCell.m

这样做还不够，这样 UITableView 并不知道这个 Cell 的高度，UITableView 也不会去根据约束去计算这个 Cell 的高度，需要设置一个属性，告知 UITableView 这个 Cell 是自适应大小的，你按钮约束来就可以了，这个属性就是：

```
    self.tableView.estimatedRowHeight = 550;
```

查看这个属性的文档如下：

>When you create a self-sizing table view cell, you need to set this property and use constraints to define the cell’s size.

>The default value is 0, which means there is no estimate.

默认值为 0，UITableView 不会去估算 Cell 的高度，当创建了一个自适应大小的 Cell 时，需要设置这个属性，并且要定义好 Cell 的约束。嗯，约束我们已经定义了，只需要打开这个开关，让 UITableView 去估算就好了，另外，这个估算代价是很大的，所以最好给一个合适的值，减小估算的范围。

运行后，效果图如上方提到的。

如果需要隐藏一部分呢？也可以。

比如当内容为空时隐藏内容部分，在最开始添加约束时先不添加按钮的上方约束，因为这个时候还不确定它要跟哪个控件有间距，可以在绑定数据时做一个判断，如果内容为空，则设置一个到标题的约束，并移除内容 Label，这样会移除内容 Label 相关的约束，在内容不为对空时，设置对内容 Label 的约束。


```
- (void)setUser:(CPYUserModel *)user {
    self.nameLabel.text = user.name;
    self.titleLabel.text = user.tweet;
    if (user.retweet.length == 0) {
        [self.contentLabel removeFromSuperview];
        [self.retweetButton cpy_topToView:self.titleLabel constant:8];
    }
    else {
        self.contentLabel.text = user.retweet;
        [self.retweetButton cpy_topToView:self.contentLabel constant:8];
    }
}
```

最终效果如下，注意第二个 Cell 是没有内容的。

![](https://i.imgur.com/qM5W60f.jpg)

这样一个动态高度的 Cell 就实现了，设置好约束后还是挺简单的。

完整 Demo 见：https://github.com/cielpy/CPYDynamicCellDemo

这样有个问题就是因为其中的计算方法是不可控的，想实现缓存因为拿不到高度，所以实现不了，只能祈祷苹果实现的比较高效。对性能要求很高的界面可能并不适应这样的方法。一般的界面应该够用了。

完。

--EOF--

