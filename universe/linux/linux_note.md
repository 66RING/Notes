---
title: Linux学习
date: 2020-1-20
tags: linux
---

## Linux哲学

- Linux基本原则
    * 1. 由目的单一的小程序组成，组合小程序完整复杂任务
    * 2. 一切皆文件
    * 3. 避免捕获用户接口，避免与用户交互
    * 4. 配置文件保存为纯文本格式
- 命令格式`命令 选项 参数`
    * 短选项：`-`加某个字母，如`-h`查看help
        + 短选项可以多个选项写一起`-a -b = -ab`
    * 长选项：`--<option>`
- 命令模板
    * `[]`中表示可选的内容
    * `<>`中表示必须给出的内容
    * `...`表示可以出现多个，可以使用多次
    * 用`|`分割表示多选一，如`[-a|-b]`
- 魔数Magic Number：标记文件的执行格式，如`#!/bin/sh`，也称为shebang


### FHS(文件层级系统)

- /boot: 系统启动相关文件
- /dev: 设备文件
- /etc: 配置文件，存文本
- /home: 用户家目录
- /root: 管理员家目录
- /lib: 库文件
    * /lib/modules: 内核模块文件
    * 静态库
        + linux下表现为.a
    * 动态库:载入到内存中可以复用
        + win下表现为.dll文件，linux下表现为.so(shared object)
- /opt: 可选目录，第三方程序的安装目录
- /proc: 伪文件系统，内核映射
- /sys: 伪文件系统，跟硬件相关的属性映射
- /tmp: 临时文件，会定时清理
- /var: 可变化文件
- /bin: 可执行文件，用户命令
- /sbin: 管理命令
- /usr: universal shared read-only
    * usr下之所以有/usr/bin和/usr/sbin是因为/bin跟启动系统相关，而/usr下跟启动系统后提供正常功能相关
- /usr/local: 又是一个独立的文件系统，也有bin，sbin。是第三方软件的安装路径


## 常用工具

- man：查看手册
    * `whatis name`查看name的目录，使用`man [n] name`打开name第n章
- tr：字符串转换或删除
    * `tr 'a-z' 'A-Z'`把输入中的字符转换成大写
    * `tr -d 'abc'`删除出现的字符，这里是a，b，c
- shell中可以开子shell，使用`pstree`可以查看父子关系，使用exit退出一个
- 环境变量
    * `PATH`命令搜索路径
    * `HISTSIZE`命令历史缓冲区
        + 使用`history`查看命令历史
        + 使用`!n`执行命令历史中第n条历史命令
        + 使用`!-n`执行命令历史中倒数第n条历史命令
        + 使用`!!`执行刚才执行的命令
        + 使用`!string`执行命令历史中最近的以指定字符串开头的命令
        + 使用`!$`引用上一个命令的最后一个参数
- 命令替换：`$(COMMAND)`或使用反引号\`COMMAND\`
    * 把命令的结果放入一个命令中
    * `echo $(ls)`
- **shell中的引号**：
    * 反引号\`\`：命令替换
    * 双引号""：弱引用。可以实现变量替换
    * 单引号''：强引用。不能实现变量替换
- 文件名通配，globbing
    * `*`：任意长度的任意字符
    * `?`：任意单个字符
    * `[]`：指定范围内的任意单个字符
    * `[^]`：指定范围外的任意单个字符
    * 特殊字符，形如`[:punct:]`。`[:punct:]`所有的标点符号
- 生成列表
    * `seq [OPTION] [FIRST] [INCREMENT] LAST`
- `xargs`
    * 从标准输入接收命令并执行
- `w`
    * 显示已登录的用户
- `last`
    * 显示登录日志，即显示`/var/log/wtmp`文件
- `lastb`
    * 显示错误的登录日志，即显示`/var/log/btmp`文件
- `lastlog`
    * 显示每一个用户最近一次的成功登录信息
- `basename`
    * 显示文件的基路径名


## 管道及IO重定向

- 系统设定
    * 默认输入设备，标准输入，stdin, 0
    * 默认输出设备，标准输出，stdout, 1
    * 标准错误输出，stderr, 2
- 重定向，默认都是重定向标准输入输出
    * 输出重定向
        + `>`覆盖输出
            + 强制覆盖：当设置了不允许覆盖已存在文件时使用`>|`
        + `>>`追加输出
        + `2>`, `2>>`重定向错误输出
            + 可以使用`ls ./ > a.out 2> b.out`这样的形式
        + `&>`重定向标准输出或错误输出到同一个文件
    * 输入重定向
        + `<`输入重定向
        + `<<`Here Document
            ```
            echo << END_SIGN
            > hello
            > END_SIGN
            ```
- 管道
    * `cmd1 | cmd2 `前一个命令的输出作为后一个命令的输入
- tee
    * `man tee`一个输入两个输出


## Shell编程

### 变量

`VARNAME=VALUE`

- 引用：`$VAENAME` or `${VARNAME}`，注意`$()`是引用执行结果，而`{}`才是引用内容
- 环境变量：作用域为当前shell或器子进程
    * 用`export`导出，`export var=value`
- 位置变量
    * `$0, $1, ...`，所有参数，第一个参数...
    * `shift [n]`，把第一个变量弹出。因此有很多变量时可以使用shift
- 特殊变量
    * `$?`，上一个命令的**执行状态**返回值，0正确，1~255执行错误
    * `$#`，参数个数
    * `$*`，参数列表
    * `$@`，参数列表
- 声明`declare [OPTION] VAR=VALUE`
    * `declare -i SUM=0`，声明整形变量SUM
    * `declare -x A=0`，声明export变量A


### 条件分支

``` shell
if expression; then
    statement
elif
    statement
else
    statement
fi
```

- 条件测试的表达式
    * `[ expression ]`， **注意** 中扩号两端的空格是必须的
    * `[[ expression ]]`，两种方法用法相同，但是两个中括号是bash的关键字
    * `test expression`
    * 因为if语句取的是表达式执行状态的结果，所以如果条件判断是命令则不需要中括号，如`if test expression; then`
- shell中进行算数运算
    * **shell把所有的变量都当作字符**，所以要进行算数运算需要额外操作
    * 1. `let`，见`help let`。如`let c=$a+$b`
    * 2. `$[expression]`。如`c=$[$a+$b]`
    * 3. `$(())`。如`c=$(($a+$b))`
    * 4. `expr expression`，表达式中，各操作数和操作符直接要有空格，且要使用命令引用。因为本质是调用`expr`程序。如`c=$(expr $a + $b)`
- 退出脚本：`exit`
    * 如果没有指定退出状态，则返回上一条命令的执行状态
    * 0正确，1～255错误
- 文件测试
    * `-e FILE`，是否存在
    * `-f FILE`，是否普通文件
    * `-d FILE`，是否为目录
    * `-r/w/x`，是否可读写执行
- 组合测试条件
    * `-a`，逻辑与
    * `-o`，逻辑或
    * `!`，非
    * 也可以使用`[ expression ] || [ expression ]`


#### switch case

``` bash
case SWITCH in
    value1)
        statement
        ...
        ;;
    value2|value3)
        statement
        ...
        ;;
    [0-9])
        statement
        ...
        ;;
    *)
        statement
        ...
        ;;
esac
```


### 文本处理

#### sed

流编辑器(Stream EDitor, sed)。行编辑器。默认将输出到屏幕

`sed [option] 'AddressCommand' file...`

- option
    * `-n`静默模式，不再显示模式空间中的内容
    * `-i`修改原文件
    * `-r`使用扩展正则表达式
- Address:
    * 1. Startline,Endline
        + \$ 表示最后一行
    * 2. /RegExp/
    * 3. /pattern1/,/pattern2/
        + 从被1匹配到的行开始到被2匹配的行结束
    * 4. Startline, +N
        + 从起始行开始向后N行
- Command
    * `d`删除
        + 如`sed 1,2d`
    * `p`显示符合条件行
    * `a string`在指定行后追加
    * `i string`在指定行前面加
    * `r FILE`将指定文件的内容添加都符号条件的行
    * `w FILE`将地址指定的范围的行保存到指定文件中
    * `s/pattern/string/[opt]`查找并替换
        + `g`全局替换
        + `i`忽略大小写
        + 分隔符号不一定是`/`也可是`@, #, ~...`等
        + `%`全文查找
    * 引用匹配串
        + `&`整个串
        + `\1`第1组


### 特殊权限

- SUID：文件运行时，相应进程的属主是程序文件自身的属主，而非启动者
    * `chmod u+s FILE`
    * `chmod u-s FILE`
    * 如果FILE本身就有执行权限，则SUID显示为s，否则为S
- GUID：文件运行时，相应进程的属组是程序文件自身的属组，而非启动者所属的基本组
    * `chmod g+s FILE`
    * `chmod g-s FILE`
- Sticky：在一个公共目录，每个用户都可以创建文件，删除自己的文件，但不能删除别人的文件
    * `chmod o+t DIR`
    * `chmod o-t DIR`
- 也可用`111`，二进制表示

- FACL, Filesystem Access Control List
    * `setfacl OPTION FILE`
        + `-m`，设定
            + `u:UID:perm`
            + `g:GID:perm`
        + `-x`，取消设定
    * `getfacl FILE`


## 文件系统

### 创建设备节点

`mknod`

``` shell
                         主设备号  次设备号             
crw-rw----+ 1 root video      81,   2   Aug 31 17:04 video2
```


### 文件系统

文件系统类型众多，接口可能各异，软件开发不可能要为每种文件系统的不同接口编写代码，那就本末倒置了。所以 **Linux使用VFS(虚拟文件系统)作为翻译官** 以支持众多文件系统。

linux常用的文件系统有ext3、ext4、xfs...

创建文件系统：`mkfs`

- 格式化
    * 高级格式化：创建文件系统`mkfs`


### 内存

linux使用线性虚拟内存做中介，从虚拟内存映射到物理内存。当物理内存满时，将闲置的程序内存放到交换空间。

查看内存以及交换空间`free`，默认单位是字节，`free -m`以兆为单位显示。

- 交换空间扩容
    * 1. 新建分区
    * 2. 创建交换分区文件系统`mkswap /path/dev`
    * 3. 启用交换空间`swapon /path/dev`，关闭交换分区`swapoff /path/dev`


### dd命令

`man dd` conver and copy a file，转换并复制文件。取别于cp，cp是以文件为单位复制，而dd复制的是底层是数据流。cp复制文件需要将文件从文件系统读取到内存，然后再从内存复制到文件系统的其他位置。而dd复制相当于直接复制文件的二进制代码。

dd的好处是可以只复制文件的某一部分。

- `dd if=<src> of=<tar>`
    * dd备份
        + `dd if=/dev/sda of=/mnt/usb/mdr.bak bs=512 count=1`
        + `dd if=/mnt/usb/mdr.bak of=/dev/sda bs=512 count=1`
        + 每次复制512byte，共复制1次
    * 用dd做磁盘镜像
        + `dd if=/path/dev of=/path/xxx.iso bs=1M count=1024`
        + 还可以用来做磁盘性能测试


### fstab文件

文件系统配置文件`/etc/fstab`文件系统表。初始化时会自动挂载此文件中定义的每个文件系统

```
要挂载的设备 挂载点 文件系统类型 挂载选项 转储频率 文件系统检测次序
    转储频率 表示多长时间(天)做次备份
    文件系统检测次序 只有根可以为1
```


### mount命令

- `mount DEVICE MOUNT_POINT`
    * `-o loop`挂载本地回环设备。**如iso镜像**
    * `-a`挂载定义在fstab中的所有文件系统 

当`umount`busy时，可以使用`fuser -v /mount/point`查看谁在访问。使用`fuser -km /mount/point`杀死所有进程






