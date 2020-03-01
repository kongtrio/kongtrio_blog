## 一、grep 命令介绍

在我们日常工作中，grep命令应该是使用很经常的一个命令。它可以拿来过滤字符串，让我们快速的从一个文件中找到是否有匹配的行。但是很多同学应该都只是使用grep来做最简单搜索，其实grep有很多参数，提供了各种丰富的功能。比如反向匹配、只输出n次，打印匹配行的前n行、后n行等等。

使用姿势:

```shell
grep [OPTIONS] PATTERN [FILE...]
# 或者通过管道
less FILE | grep [OPTIONS] PATTERN
```

## 二、参数介绍

```shell
-V, --version 输出grep版本

```

## 三、实用demo

