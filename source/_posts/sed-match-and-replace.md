---
title: 使用 sed 匹配和替换字符串
tags:
  - sed
  - 文本处理
categories:
  - 杂项
date: 2016-09-05 10:55:16
---


sed 可以方便的进行批量替换文件中字符串操作

```
sed "s/ww2/ws3/g" *.m
```

以上命令就是将当前目录下所有 .m 文件中的 ww2 替换为 ws3，并打印出来。

sed 中 如果需要覆盖当前文件，可以加上 -i 参数
例如：

```
sed -i "s/ww2/ws3/g" *.m
```

如果加了 -i 参数，作用为生成一个 -i 后面参数后缀的文件，如修改 a.m，使用 -i.c 后，会生成一个 a.mc 文件，并应用修改，只加 -i 后面不跟值，大意就是生成一个和当前名字一样的文件，也就是覆盖了，在 OS X 上不可以不加参数值，所以给一个没有任何内容的空字符串解决问题，变成以下命令

```
sed -i "" "s/ww2/ws3/g" *.m
```

参考：
  1. [In-place edits with sed on OS X](https://stackoverflow.com/questions/7573368/in-place-edits-with-sed-on-os-x)
  2. [sed 简明教程](http://coolshell.cn/articles/9104.html)

