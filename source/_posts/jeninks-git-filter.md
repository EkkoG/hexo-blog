---
title: Jenkins 中 对 Git 日志过滤
tags:
  - Jenkins
  - 持续集成
  - sed
categories:
  - 编程
date: 2016-09-05 00:37:01
---

Jenkins 中想对某次构建中的 Git 的日志进行过滤后使用，首先需要拿到上次构建成功的 Git HASH 值，Jenkins 提供了这样的一条链接，可以拿到上次构建成功时的一些信息：

```
http://<host>/job/<job_name>/lastSuccessfulBuild/api/xml
```

可以使用 curl 拿到 XML 文件的内容：

```
curl --silent --user $USER:$API_TOKEN $URL | grep "<lastBuiltRevision>" | sed 's|.*<lastBuiltRevision>.*<SHA1>\(.*\)</SHA1>.*<branch>.*|\1|'
```

`lastBuiltRevision` 就是上次构建成功时的 Git commit 的 HASH 值，注意这条链接访问需要权限，$USRR 参数是 Jenkins 用户名，$API_TOKEN 是用户密码。

Jenkins 也提供了参数给出上次构建成功时的 Git commit HASH 值，及 GIT_PREVIOUS_SUCCESSFUL_COMMIT，在执行构建时可以使用这个参数。

Git 提供了工具对日志进行格式，及 [Git - pretty-formats](https://git-scm.com/docs/pretty-formats)，接下来可以拿到两个 commit 之间的 log 并进行格式化：

```
LOG=$(git log --pretty="%s" $GIT_PREVIOUS_SUCCESSFUL_COMMIT..HEAD)
```

HEAD 就是上一次 commit 了，由于我们只需要日志信息，不需要其他，所以只保留了 `%s` 占位符，这样 LOG 变量中就保存了上次构建成功到最新一次提交间的日志，一条一行。

接下来去掉不想要的 LOG，这里简单的去掉了不以 `[` 开头的行：

```
CLEAN_LOG=$(echo "$LOG" | sed '/^[^\\[].*$/d')
```

这里有个坑，如果不用双引号包裹 `$LOG` 参数，则输出 `$LOG` 参数时会去掉换行信息，用空格隔开每一条，最终成一行，那样用 sed 命令删除行是无效的。

由于我要用在 Jenkins 邮件中，邮件是 HTML 格式的，所有我在末尾加了 `<br>` 标签：

```
LOG_WIHT_NEWLINE_TAG=$(echo "$CLEAN_LOG" | sed 's/$/&<br>/g')
```

还是用的 sed 命令，一个匹配和替换过程。

接下来翻转一下，因为通过 git log 拿到的日志是从最近一次提交到上次构建成功时的顺序，而我希望是按时间顺序来排列，所以翻转一下，还是 sed 命令：

```
FLIP_LOG=$(echo "$LOG_WIHT_NEWLINE_TAG" | sed '1!G;h;$!d')
```

嗯，这个命令我还不是很懂，Google 查到的，见参考资料。

接下来需要将所有行转成一行，之前没有转是因为方便对每一行处理，这里处理基本完成了，可以转成一行方便存储，同时也因为需要使用 [Envinject](https://wiki.jenkins-ci.org/display/JENKINS/EnvInject+Plugin) 注入环境变量，而变量是不能换行的，所有这里要转成一行，命令：

```
FLAT_LOG=$(echo "$FLIP_LOG" | tr "\n" " ";echo)
```

由于项目使用 Jira 管理 Bug，如果把日志中的 Jira bug 替换成链接就更好了，Jira 的链接格式固定，都是 http://jira.domain.com/browse/ProjectBug-xxx，可以对日志中的 `ProjectBug-xxx` 进行匹配，xxx 是数字，所以规则就是 `ProjectBug-` 后连续 N 个数字，使用正则匹配这一部分可以使用以下正则：

```
ProjectBUG-[0-9]\{1,\}
```

其实正则支持使用 `+` 进行连续匹配，但是在 sed 中使用 `+` 进行连续匹配时匹配不到内容，所以使用与 `+` 等效的 `{1,}` 进行匹配，意思为前面一规则连续 1 个到无限多个，命令如下：

```
LOG_WITH_LINK=$(echo $FLAT_LOG | sed 's/\(ProjectBug-[0-9]\{1,\}\)/<a  href="http:\/\/jira.domain.com\/browse\/\1">\1<\/a>/g')
```

注意规则中的 `{1,}` 括号要进行转义，刚开始用的时候没有转义匹配不到内容。且在替换的内容中，`/` 也要进行转义，sed 使用 `/` 进行不同作用的区域的分割符区分。

到这里日志的处理加工就完成了，写入到参数列表中：

```
echo FILTERED_CHANGES="$(echo "$LOG_WITH_LINK")" >> build.properties
```

使用 `>>` 对文件进行追加，使用 `>` 进行覆盖，这里看自己情况选择就好。

按照 [Jenkins 变量传递](https://ws3.sinaimg.cn/large/74681984gw1f7gttdvfffj21kw0exjty.jpg) 的方法设置，在之后的步骤中，就可以使用 `FILTERED_CHANGES` 变量了。

整个过程就是一个面向 Google 折腾的过程，由于对这些替换匹配操作都不熟，所以一步一查，最终达到了效果。

### 参考

1. [How to generate changelog: git log since last Hudson build?](https://stackoverflow.com/questions/2798703/how-to-generate-changelog-git-log-since-last-hudson-build)
2. [Git - pretty-formats Documentation](https://git-scm.com/docs/pretty-formats)
3. [Add environment variable for last successful build commit](https://issues.jenkins-ci.org/browse/JENKINS-27570)
4. [[bash]删除文件中含特定字符串的行](http://blog.csdn.net/joeblackzqq/article/details/6881967)
5. [如何删除空行,总结一下? ](http://bbs.chinaunix.net/thread-557345-1-1.html)
6. [Linux管道和过滤器](http://c.biancheng.net/cpp/html/2732.html)
7. [两个sed小技巧](http://easwy.com/blog/archives/two-sed-tips/)
8. [linux shell变量输出换行问题解决办法](http://www.ahlinux.com/shell/14215.html)
9. [如何实现一个文本的倒序输出？](http://bbs.chinaunix.net/thread-181010-1-1.html)
10. [Linux Sed命令详解+如何替换换行符"\n"(很多面试问道)](http://blog.csdn.net/hello_hwc/article/details/40118129)
11. [linux shell 用sed命令在文本的行尾或行首添加字符](http://www.cnblogs.com/aaronwxb/archive/2011/08/19/2145364.html)
12. [linux 下将多行合并成一行的办法](http://www.chenqing.org/2012/09/all-lines-to-one-in-linux.html)
13. [Support regex/replace attribute in ${CHANGES} token to modify changes message](https://issues.jenkins-ci.org/browse/JENKINS-23691)
14.  [正则表达式 进阶（一）-- 匹配多连续字符、位置匹配、子表达式使用](http://blog.csdn.net/wzzfeitian/article/details/8842371)
15. [正则表达式](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Regular_Expressions)
16. [sed命令详解](http://www.cnblogs.com/edwardlost/archive/2010/09/17/1829145.html)
17. [sed 匹配替换](https://wiki.imciel.com/sandry/sed%E5%8C%B9%E9%85%8D%E6%9B%BF%E6%8D%A2)
18. [Jenkins 变量传递](https://wiki.imciel.com/sandry/jenkins-var-inject)

--EOF--


