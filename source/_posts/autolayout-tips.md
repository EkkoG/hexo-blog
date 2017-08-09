---
title: 几个 Autolayout 技巧
tags:
  - Autolayout
  - iOS
  - 自动布局
categories:
  - 编程
date: 2016-08-23 22:51:50
---


iOS 设备的尺寸正在变的越来越多，在 iOS App 开发初期，只需要处理 3.5 寸 屏幕的布局，那个时候使用 frame 布局是唯一的方式，后来多了 4 寸屏幕 和 iPad 的 9.7 寸，iPad 上的 App 界面一般单独开始，写死坐标问题依旧可行，到再后来，iPad mini (7.9 寸），iPhone 6（4.7 寸），iPhone 6 plus（5.5 寸）紧跟着出现，写死坐标的布局方式的弊端开始显现，在多种尺寸上如何布局成了问题，这个时候苹果在 iOS 6 的时候推出的 Autolyout 布局方式变的流行，相对于写死坐标的布局方式，Autulayout 通过定义控件与控件之间的相对位置约束，在运行时根据不同的屏幕尺寸，得到不同的布局，更加灵活，可以更好的适应多种尺寸的设备。写一套约束，可以不同尺寸的设备上达到不同的布局效果，何乐而不为呢。

<!-- more -->

使用 Autulayout 可以有多种方式，如果习惯用 XIB 或者 Storyboard，那么可以在用可视化的方式，在 XIB 或者 Storyboard 中添加约束。如果习惯用编写代码的方式进行布局，那么可以选择使用 UIKit 中 `NSLayoutConstraint` 类的类方法 `constraintWithItem:attribute:relatedBy:toItem:attribute:multiplier:constant:` 进行布局，或者选择第三库如 [Masonry](https://github.com/SnapKit/Masonry),[PureLayout](https://github.com/PureLayout/PureLayout) 等等，它们对系统的布局方法进行了封装，语法相对来说更人性化一点，没有系统语言那多啰嗦。看个人喜好吧。

最近使用 Autulayout 比较多，有些场景下使用 Autolayout 比使用写死坐标的方法布局来的方便的多。下面举例来说我遇到的一些场景。

## 提示框

项目中经常要做一些异常提示，通常会定义一套风格相似的提示框，统一视觉，不会显得很杂乱，但是风格类似并不是说绝对相同，还是有一些差别的，举例来说，如果我们要定义的提示框有两种情况

情况 1：

![](https://ws3.sinaimg.cn/large/74681984gw1f743z6wd0lj208205na9y)

情况 2：

![](https://ws3.sinaimg.cn/large/74681984gw1f74445m5toj208006q0so)


有些提示需要给用户做选择，有些另是作为一种提示，提示后自动消失，不需要用户去操作，对于 UI 来说，只是有没有按钮的区别。对于提示框整体来说，需要解决的一个问题就是在不同的情况下，不显示按钮区域，并改变高度。

传统的写坐标的方法，布局方式大概如下：

1. 计算标题文字的高度，得到 h1
2. 计算提示语高度，得到 h2
3. 根据不同的情况，看需不需要按钮，如需要，加上按钮，得到按钮高度 h3，如不需要按钮，则 h3 为 0
4. 整个提示框的高度为 h1 + h2 + h3

如果我们用 Autolayout 布局，可以这样做：

1. 设计标题到父 View 顶部边距
2. 设置提示语到标题间距
3. 设置按钮到提示语的间距
4. 设置按钮到父 View 底部边距
5. 设置提示语到父 View 底部边距，并设置优先级为 UILayoutPriorityDefaultHigh

注意第 5 步中的优先级，默认的优先级是 UILayoutPriorityRequired，意思是必须满足，Hight 要比 Required 低一点。另外一点就是没有设置父 View 的高度约束，父 View 的高度由子 View 相对父 View 的约束以及子 View 的内容来产生。

设置好约束好，如果根据条件，如果需要显示按钮，刚以上约束不需要任何更改就可以满足条件，如果不需要按钮，那么可以把按钮移除掉，相应的，和按钮相关的约束也会移除，这时，提示语到父 View 底部优先级为 Hight 的约束则起作用。父 View 的高度自自应。

总的来说，我们只需要考虑在特定的条件下，把这个不需要的按钮移除掉即可。

这里就是优先级的一种应用，同样的道理我们还可以做一些其他的事，这里不再列举。

## 设置 UITableView 的 tableHeadView 或 tableFootView

开发中大量使用列表，很多情况下我们需要在列表上加一个「头」，如轮播图，用户信息之类的，如果我们用传统的设置坐标的方式，设置内容后可以立即产生坐标，这时直接给 tableView 的 tableHeadView 属性赋值即可，但是如果用 Autolayout 布局的自定义 View，创建并绑定数据后，其布局引擎并没有马上运行产生坐标，这个牵涉到一点 runloop 的问题，其布局会在当前的这个 runloop 结束时执行，但是如果用在 tableHeadView 上的话，必须上马上得到绝对的坐标的，那么我们可以提前这个布局过程吗？可以，只需要在设置数据后，调用该自定义 View 的 `setNeedsLayout` 方法，并接着调用 `layoutIfNeeded` 方法，则布局会提前到这个调用时完成，接着设置给 tableView 的 tableHeadView 属性即可。

整体代码如下：

```
CPYCustomTableHeadView *head = [[CPYCustomTableHeadView alloc] init];
// 绑定数据
head.data = data;

// 告诉布局系统需要马上布局
[head setNeedsLayout];
[head layoutIfNeeded];
    
self.tableView.tableHeaderView = head;
```

这样的用法在 iOS 9 上是可以正常布局并得到正确的 frame 的，但是在 iOS 8 上的表现和 iOS 9 上不同，需要另外处理一下。

```
CPYCustomTableHeadView *head = [[CPYCustomTableHeadView alloc] init];
// 绑定数据
head.data = data;

// 告诉布局系统需要马上布局
[head setNeedsLayout];
[head layoutIfNeeded];

CGFloat height = [headView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;

//update the header's frame and set it again
CGRect headerFrame = head.frame;
headerFrame.size.height = height;
headerFrame.size.width = CGRectGetWidth(self.view.bounds);
head.frame = headerFrame;
    
self.tableView.tableHeaderView = head;
```

## 写在后面

这里主要是 Autolayout 优先级的一个简单的应用，Autolayout 还可以做很多事，再遇到的话再写吧。

## 参考资料

[iOS AutoLayout - when does it Run?](https://stackoverflow.com/questions/22569104/ios-autolayout-when-does-it-run)

-- EOF --

