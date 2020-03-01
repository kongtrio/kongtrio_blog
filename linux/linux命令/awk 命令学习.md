[TOC]

## 一、awk 介绍

awk是linux上一款强大的文本分析工具，它可以将文件逐行的读入，然后用分割符分割开来，再对分割的各个部分进行处理。awk分割的各个部分叫做域，默认的分割符是空格和制表符。可以通过**-F**来指定分割符。

awk有3个不同版本: awk、nawk和gawk，未作特别说明，一般指gawk，gawk 是 AWK 的 GNU 版本。

## 二、使用姿势   

### 主要语法 

`awk [参数] 'pattern{action}'`

awk的使用看起来比较复杂，但是掌握好它的语法后，其实也很简单。

和大多数的命令一样，awk可以指定一些参数，也可以不传任何参数。awk具体支持哪些参数读者可以通过`man awk`查看相关帮助文档。这里就不多做介绍。

在参数后面跟着一个单引号括起来的执行语句，其中**pattern**表示过滤规则，支持正则表达式和逻辑表达式。如果pattern涉及多个条件，可以用 **&&** 和 **||** 来关联，分别表示与和或。**pattern可以不填，表示不过滤任何数据。**

pattern后面根据一个花括号括起来的真正的执行语句，比如 `print $1` 表示输出分割后的第一个域。action{}可以有多个语句，以;号隔开。**如果缺失action则表示输出一整行的内容。和{print}以及{print $0}效果一样**

### 使用示例

假设有一段文本**test.txt**，有2个用逗号隔开的列，分别表示姓名、年龄。

```shell
jack,18
nick,24
joe,19
hack,19
```

**输出所有人的名字**

```shell
awk -F , '{print $1}' test.txt
```

`-F ,`表示用逗号作为分割符，之后**pattern**为空，**action**则为`{print $1}`。awk执行后，每一行都会分割成两列，之后输出第一列的数据。其中$1则表示第一列的数据, \$2则表示第二列的数据,以此类推，\$n则表示第n列的数据。**要注意的是，\$0表示的是这一整行的数据**。

**输出年龄小于20岁，并且名字中带有ck内容的人（整行正则匹配）**

```shell
awk -F , '/ck/ && $2 < 20{print}' test.txt
```

上面用两个斜杠`//`围起来的就是正则表达式，awk会对每行进行正则匹配，匹配不上的就不会进行处理。

由于直接使用`//`匹配的是整行数据，所以如果我们的需求是要找名字中带有ck内容的人的话，语句就不是很正确。因为如果这一行中刚好有其他字段也包含了ck，那么就可能造成误匹配。那么，怎么就对名字这个字段进行正则匹配呢？

**输出年龄小于20岁，并且名字中带有ck内容的人（就对名字的字段进行匹配）**

```shell
awk -F , '$1 ~ /ck/ && $2 < 20{print}' test.txt
```

`$1 ~ //`则表示对第一列数据进行正则匹配。

**指定多个分隔符**

```shell
# 如果想指定多个分隔符，可以这样
echo "1;2,3.4" | awk -F '[;,.]+' '{print $1,$2,$3,$4}' 
```

### BEGIN和END关键字     

action里面的语句会对每一行过滤后的数据进行输出，那么，如果我们想在输出的头部和尾部增加一些内容，应该怎么做呢？答案就是使用BEGIN和END关键字。**记得一定要大写**

BEGIN后面跟一个语句块`{}`，表示在awk扫描文本前输出一些内容。END用法也一样，在awk扫描文本后输出一些内容。

```shell
awk -F , 'BEGIN {print "name,age"} {print} END {print "end"}' test.txt
//输出 
name,age
jack,18
nick,24
joe,19
hack,19
end
```

上面的语句我们可以看到有3个`{}`语句块，分别表示**扫描前语句块**，**扫描文本时使用的语句块**，**扫描后语句块**。

### 内置变量  

```shell
ARGC               命令行参数个数
ARGV               命令行参数排列
ENVIRON            支持队列中系统环境变量的使用
FILENAME           awk浏览的文件名
FNR                浏览文件的记录数,也就是记录所在的行数
FS                 设置输入域分隔符，等价于命令行 -F选项
NF                 浏览记录的域的个数
NR                 已读的记录数
OFS                输出域分隔符
ORS                输出记录分隔符
RS                 控制记录分隔符
```

awk内置了一些变量，我们可以在语句块直接使用 。

**直接输出第2行的数据**

```shell
awk -F , 'FNR==2 {print}' test.txt
```

### print和printf的区别  

awk中同时提供了print和printf两种打印输出的函数。

其中print函数的参数可以是变量、数值或者字符串。字符串必须用双引号引用括起来，参数必须用逗号分隔，不然多个参数之间连在一起会造成混淆。

printf和c语言中的printf基本一样，可以格式化字符串。

**print输出例子**

```shell
awk -F , '{print "name="$1",""age="$2}' test.txt
```

**printf输出例子**

```shell
awk -F , '{printf "name=%s,age=%0.2f\n",$1,$2}' test.txt

//输出
name=jack,age=18.00
name=nick,age=24.00
name=joe,age=19.00
name=hack,age=19.00
```

通过printf，我们可以将年龄转化成小数点

## 三、awk 脚本  

通过 ` -f scriptfile ` 来将awk执行语句放到脚本中。我们可以编写一个awk脚本`test.awk`。

```shell
BEGIN {print "name,age"}
$2 < 20 && /ck/ {print $1}
END {print "end"}
```

之后执行

```shell
awk -F , -f test.awk test.txt
```

等同于执行

```shell
awk -F , 'BEGIN {print "name,age"}  $2 < 20 && /ck/ {print $1} END {print "end"}'
```

awk脚本也可以这么写  

```shell
#!/usr/bin/awk -f
BEGIN {print "name,age"}
$2 < 20 && /ck/ {print $1}
END {print "end"}
```

之后直接执行该脚本即可

```shell
./test.awk -F , test.txt
```

## 四、awk 编程

### 定义变量  

awk语句中可以直接定义变量然后使用

```shell
# 设置count变量，统计一共有多少行
awk -F , '{count++;print $0} END {print count}' test.txt
# 设置count变量的初始值为1
awk -F , 'BEGIN{count=1} {count++;print $0} END {print count}' test.txt
```

我们可以在BEGIN语句块中设置变量的初始值，如果print要输出一个没有定义过的变量，awk也不会报错，而是输出空字符串。

### 条件语句  

awk的条件语句也和C语言基本一样。

```shell
# 如果岁数小于20岁并且名字字段带有ck，输出的岁数就+5，否则就+1
awk -F , '{if($2<20 && $1 ~ /ck/){print $1,$2+5}else{print $1,$2+1}}' test.txt
```

上面的语句就是我们常见的if…else，很好理解。

### 循环语句  

awk的循环语句也和C语言基本一样。支持**while、do/while、for、break、continue**这些关键字。

```shell
# 每一行重复输出3次
awk -F, 'BEGIN{i=0} {while(i<3){print $0;i++};i=0}' test.txt
awk -F, '{for(i=0;i<3;i++){print $0}}' test.txt
```

### 数组  

awk的数组的下标可以是数字或者字母，这和js的map比较像。

```Shell
# 输出的过程中，遇到名字是nick的，把它的名字转换成 hello 。最后我们再输出一个world
awk -F, 'BEGIN{nickname["nick"]="hello";nickname[1]="world"} {if(nickname[$1]!=""){print nickname[$1],$2}else{print $1,$2}} END{print nickname[1]}' test.txt
```

上面的语句在BEGIN块中定义了nickname的数组，同时使用了两种下标，分别是"nick"和数字1。之后在后面也用到了。

## 五、写在结尾

awk是linux上一款处理文本的神器，以前看到或者要用的时候总是习惯去百度，然后直接抄过来改一点东西，而没有去较系统的学习一下。主要还是被它那看似复杂的语句块吓住，今天在认真看了几篇awk教程后发现其本质也很简单，学习起来其实很快。个人觉的有编程基础的人，差不多半个小时就足以将awk掌握个大概。

当然，要深入的学习awk肯定不是一件简单的事，本篇博客也只是对awk的语句进行一些简单的介绍，也足以应付我们工作中大部分的需求。如果想深入学习awk的同学可以去官方文档<http://www.gnu.org/software/gawk/manual/gawk.html>学习看看。