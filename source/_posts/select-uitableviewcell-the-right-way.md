---
title: 选中 UITableViewCell 及保存选中状态的正确方式
tags:
  - iOS
categories:
  - 编程
date: 2016-06-15 15:14:52
---

在开发过程中，经常用到一个控件就是 `UITableView` ，我们时常会需要处理一个 `cell` 的选中状态，以给用户一些提示：「我选中了这个」，如果尝试在 `tableView:cellForRowAtIndexPath:` 方法调用 `setSelected:animated:` 方法，代码如下：

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"cell" forIndexPath:indexPath];
    [cell setSelected:YES animated:YES];
    return cell;
}
```

运行后会发现是没有效果的，`cell` 最终会是未选中状态，可以在自定义的 `UITableViewCell` 子类中重写 `setSelected:animated:` 方法，加上断点看到，每个 `cell` 的 `setSelected:animated:` 方法会被调用三次

<!-- more -->

我们同时可以在这个断点触发时看到左侧的调用栈信息

`cell` 在初始化或者重用时，调用 `-_configureCellForDisplay:forIndexPath:` 方法，这个方法会做一些附带操作，其中会调用 `setSelected:animated:` 方法，将 cell 的选中状态置为未选中。

![](https://i.imgur.com/zRjlkMp.jpg)

![](https://i.imgur.com/c8jhf4o.png)

在 `tableView:cellForRowAtIndexPath:` 中调用 `setSelected:animated:` 方法将 `cell` 设置为了选中
![](https://i.imgur.com/0o5XSSl.png)

之后又在 `-_configureCellForDisplay:forIndexPath:` 中调用了`setSelected:animated:` 方法，将 `cell` 设置为了未选中
![](https://i.imgur.com/VtiQwnf.png)

可以通过实现 `UITableViewDelegate` 的 `willDisplayCell:forRowAtIndexPath:` 方法，在 `cell` 即将显示的时候，对相应 indexPath 的 cell 的选中状态进行设置，这个方法会在 `-_configureCellForDisplay:forIndexPath:` 后调用。

UITableView 会在再次显示到屏幕上时将已选中的 cell 选中状态置为未选中，若需要保存选中状态则需要自己实现，保存选中的 `cell` 的 `IndexPath`，并在 `viewWillAppear:` 中调用 `selectRowAtIndexPath:animated:scrollPosition:` 方法将保存的 `NSIndexPath` 数组中对应的 `cell` 选中。

--EOF--

## 参考资料
1. http://stackoverflow.com/a/25128477
2. http://stackoverflow.com/a/30736675

-- EOF --

