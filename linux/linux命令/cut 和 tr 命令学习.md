[TOC]

## 一、cut命令学习

cut命令主要用来切割字符串，可以对输入的数据进行切割然后输出，它可以支持三种形势的切割：

- 按字节（bytes）进行切割 
- 按字符进行切割
- 按指定的分割符进行切割

在linux中，一些特殊字符（比如中文）会占多个字节，所以，通过字节分割和通过字符分割是不同的，这一点要搞清楚。

### 按字节进行切割

通过`-b`参数，我们可以按字节来切割字符串，使用命令格式如下：

`cut [-n] -b List inputfile`

```shell
# 获取第1个和第3个字节.
# 输出: hl
echo "hello world" | cut -b 1,3
# 获取第1个到第3个之间的字节
# 输出: hel
echo "hello world" | cut -b 1-3
# 如果有中文字符，就无法很好的通过-b来获取
# 输出: � 
echo "h和o" | cut -b 2
# 输出(一个中文汉字占3个字节): 和
echo "h和o" | cut -b 2-4
# 加上 -n ，遇到这种多个字节组成的字符就只会在最后一个字节处才会输出
# 输出(输出空字符): ""
echo "h和o" | cut -n -b 2
# 输出(第四个字节刚好是最后一个字节): 和
echo "h和o" | cut -n -b 4
```

### 按字符进行切割

通过`-c`参数，我们可以按字符来切割字符串，使用命令格式如下：

`cut -c List inputfile`

```shell
# 获取第1个和第3个字符.
# 输出: hl
echo "hello world" | cut -c 1,3
# 获取第1个到第3个之间的字符
# 输出: hel
echo "hello world" | cut -c 1-3
# 带中文也可以输出
# 输出: 和
echo "h和o" | cut -c 2
```

### 按指定字符进行切割

通过`-d`和`-f`配合，我们可以按指定字符来分割字符串，使用命令格式如下：

`cut -d 'DELIM' -f LIST`

```shell
# 按 "," 号分割，并输出第一列和第三列
# 输出： hello,ok
echo "hello,world,ok" | cut -d , -f 1,3
# 按 "," 号分割，并输出第1到第三列之间的数据
# 输出： world,ok
echo "hello,world,ok" | cut -d , -f 2-3
```

## 二、tr 命令学习

tr 命令主要用来替换字符。它的原理是对输入的数据**按字符**进行替换或者删除（也只能按字符来，不能根据单词来做替换）。

tr有几个常用的参数：

- `-c`:通过指定字符的补集来替换字符串（也就是反向匹配）
- `-d`:删除字符
-  `-s`:对连续重复的字符进行去重
- `-t`:忽略SET1中多出的字符

### 替换字符使用demo

tr会根据传入的两个SET的字符顺序来替换字符串，比如SET1的第一个字符是a,SET2的第一个字符是x。那么tr在运行时会将所有的a字符替换成x字符

```shell
# 根据字符的顺序进行匹配替换：h->x e->y l->c
# 输出： xyzzo worzd
echo "hello world" | tr "hel" "xyz"
# 如果SET2的length没有SET1，就会自动用SET2的最后一位补上
# 输出： xxxxo worxd
echo "hello world" | tr "hel" "x"
```

### -c，-d 参数使用demo

```shell
# 去除数字外的所有字符。-d表示删除匹配的字符，-c表示反向匹配
# 输出: 123
echo "hello 123 world" | tr -c -d "0-9"
# 没加-c的话输出: hello  world
echo "hello 123 world" | tr -d "0-9"
```

### -s 参数使用demo

```shell
# 去除连续重复的字符
# 输出: heo o aa
echo "hello ll aa" | tr -s "l" "o"
```

### -t 参数使用demo

```shell
# 去除SET1多余的字符
# 没加-t前，输出： xxxxo worxd
echo "hello world" | tr "hel" "x"
# 加了-t，输出: xello world
echo "hello world" | tr -t "hel" "x"
```

### tr中的一些转义符

所有的转义符如下：

```shell
\NNN 八进制值的字符 NNN (1 to 3 为八进制值的字符)
\\ 反斜杠
\a Ctrl-G 铃声
\b Ctrl-H 退格符
\f Ctrl-L 走行换页
\n Ctrl-J 新行
\r Ctrl-M 回车
\t Ctrl-I tab键
\v Ctrl-X 水平制表符
[:alnum:] 所有的字母和数字
[:alpha:] 所有字母
[:blank:] 水平制表符，空白等
[:cntrl:] 所有控制字符
[:digit:] 所有的数字
[:graph:] 所有可打印字符，不包括空格
[:lower:] 所有的小写字符
[:print:] 所有可打印字符，包括空格
[:punct:] 所有的标点字符
[:space:] 所有的横向或纵向的空白
[:upper:] 所有大写字母
```

demo:

```shell
# 所有小写字符转大写字符(两种方式)
# 输出: HELLO
echo "heLlo" | tr "[:lower:]" "[:upper:]"
echo "heLlo" | tr [a-z] [A-Z]
```

## 三、总结

其实cut和tr命令和awk与sed很像。cut基本就是awk的简单版本，而tr就是sed的简单版本。虽然awk和sed的功能很强大，但是一些比较简单的场景，其实使用cut和tr就足够了。