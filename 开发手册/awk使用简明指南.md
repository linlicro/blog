# awk 简明使用指南

**`awk`**是处理文本文件的一个应用程序，几乎所有 Linux 系统都自带这个程序。

本篇使用指南会介绍该命令行的用法，对于大多数场景，用过足够使用了:

- 基本用法
- 变量
- 函数
- 条件

指南中使用的`demo.txt`内容是:

    $ cat demo.txt
    root:x:0:0:root:/root:/bin/bash
    bin:x:1:1:bin:/bin:/sbin/nologin
    daemon:x:2:2:daemon:/sbin:/sbin/nologin
    adm:x:3:4:adm:/var/adm:/sbin/nologin
    lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
    sync:x:5:0:sync:/sbin:/bin/sync

## 一、基本用法

**`awk`**的基本用法就是下面的形式。

    # 格式
    $ awk 选项 动作 文件名
    # 常用的选项: -F ':' => 指定分隔符为冒号
    # 其他选项还有 
    # * -f file => 指定需执行的外部awk脚本文件
    # * -v var-value => 定义一个变量
    
    # 示例
    $ awk -F ':' '{print $0}' demo.txt

demo.txt是**`awk`**所要处理的文本文件。前面单引号内部是文本文件里面每一行的处理动作`print $0`，其中，$0代表当前行。因此上面的命令就是把每一行原样的打印出来。

下面，用标准输入(stdin)演示下:  *把标准输入的文本，重新打印一遍。*

    $ echo 'this is a txt' | awk '{print $0}'
    this is a txt

`awk`还可以根据空格和制表符，将每一行分成若干行字段，依次用$1, $2, $3代表第一个字段、第二个字段、第三个字段等。

举例: 使用`$3`来表示this is a txt的第三个字段a。

    $ echo 'this is a txt' | awk '{print $3}'
    a

`awk`还可以用`-F`参数指定分隔符。下面，我们把/etc/passwd文件保存为demo.txt

    root:x:0:0:root:/root:/bin/bash
    bin:x:1:1:bin:/bin:/sbin/nologin
    daemon:x:2:2:daemon:/sbin:/sbin/nologin
    adm:x:3:4:adm:/var/adm:/sbin/nologin
    lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
    sync:x:5:0:sync:/sbin:/bin/sync

这个文件的字段分隔符是冒号（ : ），所以要用-F参数指定分隔符为冒号，然后才能得到它的第一个字段。

    $ awk -F ':' '{print $1}' demo.txt
    root
    bin
    daemon
    adm
    lp
    sync

## 二、变量

上面提到了`$+`数字表示某个字段，`awk`还提供其他一些变量。

变量`NF`表示当前行有多少个字段，因此`$NF`就表示最后一个字段。

    $ echo 'this is a text' | awk '{print $NF}'
    test

`$(NF-1)`标识倒数第二个字段。

    $ awk -F ':' '{print $1, $(NF-1)}' demo.txt
    root /root
    bin /bin
    daemon /sbin
    adm /var/adm
    lp /var/spool/lpd
    sync /sbin

变量`NR`表示当前处理的是第几行。

    $ awk -F ':' '{print NR ")", $1}' demo.txt
    1) root
    2) bin
    3) daemon
    4) adm
    5) lp
    6) sync

`awk`的其他内置变量如下: 

    * FILENAME：当前文件名
    * FS：字段分隔符，默认是空格和制表符
    * RS：行分隔符，用于分割每一行，默认是换行符。
    * OFS：输出字段的分隔符，用于打印时分隔字段，默认为空格。
    * ORS：输出记录的分隔符，用于打印时分隔记录，默认为换行符。
    * OFMT：数字输出的格式，默认为％.6g。

## 三、函数

`awk`还提供了一些内置函数，方便对原始数据的处理。

函数`toupper()`用于将字符转为大写。

    $ awk -F ':' '{print toupper($1)}' demo.txt
    ROOT
    BIN
    DAEMON
    ADM
    LP
    SYNC

其他常用函数如下:

    * tolower(): 字符转为小写
    * length(): 返回字符串长度
    * substr(): 返回子字符串
    * sin(): 正弦
    * cos(): 余弦
    * sqrt(): 平方根
    * rand(): 随机数

`awk`完整的函数列表，可以查看[手册](https://www.gnu.org/software/gawk/manual/html_node/Built_002din.html#Built_002din)

## 四、条件

`awk`允许指定输出条件，只输出符合条件的行。

    $ awk '条件 动作' 文件名

下面举个示例:  *输出包含root的行*

    $ awk -F ':' '/root/ {print $1}' demo.txt
    root

下面的例子，只输出奇数行，以及输出第三行以后的行

    $ awk -F ':' 'NR % 2 == 1 {print $1}' demo.txt
    root
    daemon
    lp
    $ awk -F ':' 'NR > 3 {print $1}' demo.txt
    adm
    lp
    sync

下面的例子，输出第一个字段等于指定值的行

    $ awk -F ':' '$1 == "root" {print $1}' demo.txt
    root
    $ awk -F ':' '$1 == "root" || $1 == "bin" {print $1}' demo.txt
    root
    bin

## 五、if语句

`awk`提供了`if`结构，用于编写复杂的条件。

    $ awk -F ':' '{ if($1 > "m") print $1}' demo.txt
    root
    sync

`if`结构还可以指定`else`部分

    $ awk -F ':' '{ if($1 > "m") print $1; else print "---"}' demo.txt
    root
    ---
    ---
    ---
    ---
    sync

## 六、参考资料

- [30 Examples for Awk Command in Text Processing](https://likegeeks.com/awk-command/), Mokhtar Ebrahim