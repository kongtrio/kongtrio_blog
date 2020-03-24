[TOC]

## 一、grep 命令介绍

在我们日常工作中，grep命令应该是使用很经常的一个命令。它可以拿来过滤字符串，让我们快速的从一个文件中找到是否有匹配的行。但是很多同学应该都只是使用grep来做最简单搜索，其实grep有很多参数，提供了各种丰富的功能。比如反向匹配、只输出n次，打印匹配行的前n行、后n行等等。

基本使用姿势:

```shell
grep [OPTIONS] PATTERN [FILE...]
# 或者通过管道
cat FILE | grep [OPTIONS] PATTERN
```

## 二、grep、egrep、fgrep区别

其中egrep和fgrep是grep的衍生物。其中`grep -E`等价于 egrep，`grep -F `等价于fgrep。

这三个命令的区别在于匹配串匹配内容的的行为不同：

**grep**

使用`基础的正则表达式`去匹配内容

**egrep**

使用`扩展的正则表达式`去匹配内容

**fgrep**

使用固定字符串进行匹配，也就是不使用正则表达式，匹配串是什么就是什么内容。可以理解为java的contain方法，因此速度比grep和egrep都快。

### 基础的正则表达式和扩展的正则表达式 

顾名思义，扩展的正则表达式就是在基础的正则表达式上做了一些扩展以及语法增强。

其中最主要的区别在于元字符的使用方式上，在BRE(basic regular expression)中,很多元字符都需要加上转义字符'\':

```shell
.* 表示匹配任意长度的任意字符 (这个和ERE没区别)
\? 匹配其前面的字符0或1次
\+ 匹配其前面的字符至少1次 (非贪婪模式)
\{n\} 匹配前面的字符n次
\{m,n\} 匹配前面的字符至少m次，至多n次
\{,n\} 匹配前面的字符至多n次
\{n,\} 匹配前面的字符至少n次
\(a-z\) 分组匹配
\| 表示或,比如 \(C\|c\)ar 可以匹配 car或者Car
```

而在ERE(extended regular expression)中:

```shell
.* 表示匹配任意长度的任意字符
? 匹配其前面的字符0或1次
+ 匹配其前面的字符至少1次 (非贪婪模式)
{n} 匹配前面的字符n次
{m,n} 匹配前面的字符至少m次，至多n次
{,n} 匹配前面的字符至多n次
{n,} 匹配前面的字符至少n次
(a-z) 分组匹配
| 表示或,比如 (C|c)ar 可以匹配 car或者Car
```

另外，ERE中还可以使用一些特殊的转义来表达字母或者字符，比如可以用\d来表示数字，\w表示英文字母，而这些在BRE中是不支持的。

**附录(一些特殊的字符匹配)**

```shell
[^] 匹配指定范围外的任意单个字符
[:alnum:] 字母和数字
[:alpha:] 代表任何英文大小写字符，亦即 A-Z, a-z
[:lower:] 小写字母 [:upper:] 大写字母
[:blank:] 空白字符（空格和制表符）
[:space:] 水平和垂直的空白字符（比[:blank:]包含的范围广）
[:cntrl:] 不可打印的控制字符（退格、删除、警铃...）
[:digit:] 十进制数字 [:xdigit:]十六进制数字
[:graph:] 可打印的非空白字符
[:print:] 可打印字符
[:punct:] 标点符号
```

> grep还有衍生命令  zgrep，zegrep，zfgrep 都是用于查找zip格式的文件内容的(不用解压文件)

## 三、参数介绍

也可以直接通过`man grep`查看

```shell

-A num, --after-context=num  显示匹配行的后N行
-a, --text 以ASCII的编码来读取所有的文件，如果没有这个参数，在读取二进制文件时，会报错:Binary file ... matches
-B num, --before-context=num  显示匹配行的前N行
-b, --byte-offset  显示匹配串在匹配到的行中的位置(单位不是字节,是byte)
-C[num, --context=num]  显示匹配行的前后N行,等价于 -A N -B N
--colour=[when, --color=[when]] 让匹配串有颜色,可以通过设置环境变量GREP_COLOR=always来实现
-D action, --devices=action 定义要怎么处理linux设备文件,默认值是read，表示当成普通文件来读,可以设置为skip跳过设备文件
-d action, --directories=action 定义要怎么处理目录，read表示当成文件来读取,skip表示跳过目录,recurse表示递归往下读(等于-R或者-r参数)
-e pattern, --regexp=pattern 指定匹配串，可以通过-e参数指定多个，来达到要匹配多个匹配串的目的。比如 grep -e "a" -b "c" test.txt
--exclude 根据文件名来排除一些文件（支持通配符），不进行匹配。比--include优先级高
--exclude-dir 排除目录，优先级比--include-dir高
-F, --fixed-strings  用固定字符串匹配，等价于fgrep
-f file, --file=file  从文件中获取匹配串
-H  在匹配行前输出文件名 
-h, --no-filename  不在匹配行前输出文件名
--help  输出帮助信息
-I 忽略二进制文件
-i, --ignore-case  匹配时忽略大小写,grep默认是大小写敏感的
--include  只搜索指定的文件。
--include-dir  只搜索指定的目录
-J, --bz2decompress  解压bzip文件后再查找匹配串 
-L, --files-without-match  把未匹配到的文件输出
-l, --files-with-matches  只输出匹配到的文件名
--mmap  使用mmap来读取文件，在一些场景下性能会比较高，但是偶尔会有一些难以预料的错误(不稳定)
-m num, --max-count=num  匹配到N行后退出程序
-n, --line-number  在匹配行前输出行号
--null  在匹配文件名后带上空字节
-O  是否要递归处理软链接，默认是不处理
-o, --only-matching  只输出匹配到的内容,比如echo "hel2lo" | grep -o "l[1-9]l" 会输出 l2l
-p 和-O参数相反，默认是不处理软连接
-q, --quiet, --silent  是否输出匹配到的行 (在这个模式下性能更高)。后面可以根据exitCode来判断是否匹配到了内容
-R, -r, --recursive  遇到目录时递归处理
-S  和-O类似，递归处理软连接
-s, --no-messages  不输出错误信息(比如文件不可读或者不存在)
-U, --binary 从二进制文件中查找，但是不会打印出来
-V, --version 输出grep版本
-v, --invert-match  取反操作，没匹配到匹配串的行才输出
-w, --word-regexp  用来搜索单词
-x, --line-regexp  用来搜索整行内容，只有整行内容匹配上时才输出
-y 和-i一样
-Z, -z, --decompress  等价于zgrep
--binary-files=value  value有三种，默认值是binary，表示对二进制进行搜索但不输出，without-match 表示不查找二进制文件，text 表示把所有二进制文件都当成文本文件来看
```

## 四、一些demo

### 4.1 在某个目录下搜索字符串 

其中-F主要是为了查询可以更快速，如果要进行正则匹配可以不加

`grep -rF "xxx" ./*`

### 4.2 使用后向引用来匹配

\1 表示匹配到第一个分组，也就是(hello)，以此类推，\2 表示第二个分组

`echo "hello 1 world hello 999" | egrep "(hello) [1-9]? world \1"`

### 4.3 匹配5行后立即退出

`grep -m 5 "xxx" test.txt`

## 五、参考文档

<https://www.cnblogs.com/Dlg-Blog/p/8733908.html>

<https://www.regular-expressions.info/posix.html>