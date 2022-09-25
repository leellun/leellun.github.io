---
title: linux基础
date: 2020-02-09 20:14:02
categories:
  - 服务器
tags:
  - centos 
  - linux
---

# 一 系统文件夹介绍

- /    根目录分区

- /boot启动Linux的核心文件
- /bin：存放最常用命令； 
- /dev：设备文件；
- /etc：存放各种配置文件；
- /home：用户主目录；
- /lib：系统最基本的动态链接共享库；
- /mnt：一般是空的，用来临时挂载别的文件系统；
- /proc：虚拟目录，是内存的映射；
- /sbin：系统管理员命令存放目录；
- /usr：最大的目录，存许应用程序和文件；
- /usr/X11R6：X-Window目录；
- /usr/src：Linux源代码；
- /usr/include：系统头文件；
- /usr/lib：存放常用动态链接共享库、静态档案库；
- /usr/bin、/usr/sbin：这是对/bin、/sbin的一个补充；

# **二 常用命令**

## 基础命令

**ls**

指定路径或者当前路径下，文件夹详细信息。

参数：-a 获取详细信息

​      -l  列表形似展示

```
ls -l /usr/local
```

**su**

用户切换

su 用户名、su切换成root 、su -进入root

**whoami**

查看当前用户。

**exit**

退出当前用户登录

**which**

查看当前命令的路径

如which ls获取ls命令的文件路径

**du**

du -h 文件：大小查看

## **文件操作命令**

**mkdir**

单级文件夹创建，例如：mkdir test 创建test文件夹。

多级文件夹创建，例如：mkdir -p a/b/c 创建多级文件夹。

**mv**

文件/文件夹移动

```
mv file folder
```

**cp**

文件复制

文件复制：cp file dir

递归文件复制：cp -r dir1 dir2

**rm**

文件删除

参数：

-r 递归删除

-f 强制删除

rm dir

rm -r -f dir

**touch**

创建文件

touch file1

## **文件查看命令**

**cat**

查看文件内容

如 cat a.txt

**more**

一页一页的查看

**less**

一页查看可以回滚

**head**

通过-n指定查看文件的开始部分内容

例如：head -100 file 查看文件的开头100行

**tail**

通过-n指定查看文件的末尾部分内容

例如：tail -100 file 查看文件的末尾100行.

## **权限操作**

**chmod**

u:当前用户 g:当前组 o:其它用户

chmod u+rwx file 为当前用户添加读写执行权限。

chmod g-rx file 为同组用户删除读和执行权限

 

chmod 777 为所有用户(当前用户、同组用户、其它用户)添加读、写、执行权限

7的二进制对应 111，分别表示 读 写 执行，即：r=4，w=2，x=1 

递归设置需要加入参数 -R

 

**chown**

文文件设置用户、用户组。

chown 用户 file

chown 用户.组 file

chown  .组 file

递归设置：

chown -R 用户.组 dir

## **系统命令**

**grep**

将文本中指定的信息进行匹配

grep 关键字 路径名

**ps** 

查看系统进程

ps  -A  或者 ps -ef

**date** 

查看系统时间

date -s “2019-09-13 11：11：11”设置系统时间

**df**

查看系统分区情况

df -lh

**kill**

进程杀死

kill -9 pid

**find**

文件查找

-name 查找文件名

-maxdepth最大深度

-mindepth最小深度

-size 大小筛选，+/-数字

+表示大于指定范围

-表示小于指定范围

单位：

未指定：512字节

c:字节

K和M和文件系统一样

**ln**

链接创建

ln -s 文件 软连接文件

ln  [-d] 文件 硬链接文件

 

注：1 硬链接不需要使用绝对路径 2 普通文件才能创建硬链接，目录不可3硬链接与源文件在同一个硬盘同一个分区里。

**crontab**

查看调度信息crontab -l

编辑调度信息crontab -e

分钟	小时	日期	月份	星期	执行命令

 

**mount**

挂载：mount 硬件  挂载目录

卸载：unmount 硬件    

**eject**

弹出光盘

# **三 vi操作**

进入文件，进行内容查看、编辑

vi test.txt

**进入编辑模式**

- a:光标向后移动一位
- i:光标和所在字符不发生任何变化
- o:给新起一行
- s:删除光标所在字符

 

**行号设置**

显示行号

:set number或者:set nu

取消行号显示

:set nonumber或者:set nonu

 

**内容查找**

查找指定内容，小写n下一个、大写N上一个。

:/内容/ 或者 :/内容

 

跳转到第n行

:n

 

**内容替换**

替换光标所在行的第一个content1

```
:s/content1/content2/
```

替换光标所在行的全部content1

```
:s/content1/content2/g
```

全局替换

```
:%s/content1/content2/g
```

**编辑操作**

dd：删除当前光标行

ndd：包含当前行在内，向后删除n行内容

x：删除光标所在字符。

c+w：从光标所在位置删除至单词结尾，并进入编辑模式

 yy：复制光标当前行

nyy：包含当前行在内，向后复制n行内容

p：粘贴

 u：撤销

J：合并上下行

r：当个字符替换

.：重复执行上次执行的命令。

# 四、 管道

对于grep、head、tail、wc、ls等都可以当作管道符号使用。

ls -l|wc    //计算当前目录共有多少文件

grep sbin passwd | wc  //passwd文件出现sbin的行数统计

cat file|grep ssss   //内容查找过滤

# 五、系统信息

**查看版本当前操作系统内核信息**

```
uname -a
```

**查看当前操作系统版本信息**

```
 cat /proc/version
```

**查看版本当前操作系统发行版信息**

```
cat /etc/issue  或cat /etc/redhat-release
```

**查看cpu相关信息，包括型号、主频、内核信息**

```
cat /proc/cpuinfo
```

**查看位数**

```
getconf LONG_BIT
```

vmstat

```
[root@k8s-master01 redis]# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 405900      0 1099844    0    0    17    20   47   66  4  3 93  0  0
```

结果解释如下：

```
procs
r 列表示运行和等待cpu时间片的进程数，如果长期大于1，说明cpu不足，需要增加cpu。
b 列表示在等待资源的进程数，比如正在等待I/O、或者内存交换等。
cpu 表示cpu的使用状态
us 列显示了用户方式下所花费 CPU 时间的百分比。us的值比较高时，说明用户进程消耗的cpu时间多，但是如果长期大于50%，需要考虑优化用户的程序。
sy 列显示了内核进程所花费的cpu时间的百分比。这里us + sy的参考值为80%，如果us+sy 大于 80%说明可能存在CPU不足。
wa 列显示了IO等待所占用的CPU时间的百分比。这里wa的参考值为30%，如果wa超过30%，说明IO等待严重，这可能是磁盘大量随机访问造成的，也可能磁盘或者磁盘访问控制器的带宽瓶颈造成的(主要是块操作)。 
id 列显示了cpu处在空闲状态的时间百分比 
system 显示采集间隔内发生的中断数
in 列表示在某一时间间隔中观测到的每秒设备中断数。
cs列表示每秒产生的上下文切换次数，如当 cs 比磁盘 I/O 和网络信息包速率高得多，都应进行进一步调查。
memory
swpd 切换到内存交换区的内存数量(k表示)。如果swpd的值不为0，或者比较大，比如超过了100m，只要si、so的值长期为0，系统性能还是正常 
free 当前的空闲页面列表中内存数量(k表示) 
buff 作为buffer cache的内存数量，一般对块设备的读写才需要缓冲。 
cache: 作为page cache的内存数量，一般作为文件系统的cache，如果cache较大，说明用到cache的文件较多，如果此时IO中bi比较小，说明文件系统效率比较好。 
swap
si 由内存进入内存交换区数量。
so由内存交换区进入内存数量。 
IO
bi 从块设备读入数据的总量（读磁盘）（每秒kb）。
bo 块设备写入数据的总量（写磁盘）（每秒kb）
```

# 六、创建分区

列出系统分区 

```
fdisk -l
```

给硬盘分区 

```
fdisk 设备文件名
```

重新加载 /etc/fstab

```
umount -a & mount -a
```

# 七、将任务转到前后台执行

前台执行按ctrl + z 可以放到后台并暂停，jobs 可以查看后台执行的任务，后台进程。 

bg 将一个在后台暂停的命令，变成继续执行 （在后台执行）

 fg 放在前台

ctrl + c 即刻停止。 

 