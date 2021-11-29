---
title: "linux相关指令"
date: 2021-11-29T11:14:03+08:00
draft: false
---

# Ubuntu命令大全

### apt-get

> apt-get update:更新安装列表
> apt-get upgrade:升级软件
> apt-get install software_name :安装软件
> apt-get --purge remove  software_name :卸载软件及其配置
> apt-get autoremove software_name:卸载软件及其依赖的安装包

### 罗列已安装软件

> dpkg --list

//安装.debtmpfs           7.8G  133M  7.7G   2% /tmp
/dev/sda        1.8T  155G  1.6T   9% /home/gongqi/MountFiles
/dev/sdb1       300M  280K  300M   1% /boot/efi
/dev/loop1       32M   32M     0 100% /var/lib/snapd/snap/snapd/10707

​        sudo dpkg -i <package.deb>

sudo uname -m

> 查看Ubuntu系统的位数

### 界面切换

> ctrl+alt+f2 //命令行界面
>
> ctrl+alt+f7 // 图形界面

### 查看帮助

- --help
- man 手册：
  - 9卷：man man 
    - 1. shell 命令帮助信息
       2. 系统调用 帮助信息
       3. 库函数 帮助信息
       4. 第5卷， 文件格式： 如，man passwd 

- 命令使用技巧
  1. 快速补齐命令、文件、目录名 ——> Tab 键
  2. history 列出过往执行过的命令。 使用 ! 编号 执行过往命令
  3. ctrl + p 列出上一条命令
  4. ctrl + n 列出下一条命令
  5. ctrl + a 将光标移至行首 
  6. ctrl + e 将光标移至行尾
  7. ctrl + u 清空光标以前的命令

### ls -l 命令

- 7部分内容：

  1. 文件属性

     1. 文件类型：

       - Linux系统，不以后缀名作为区分文件类型的依据。

         1. 普通文件 - 			占用磁盘存储
         2. 目录文件 d			占用磁盘存储
         3. 软链接（win 快捷方式）l			占用磁盘存储
         4. 字符设备 c			不占用磁盘存储（伪文件）
         5. 块设备 b			不占用磁盘存储（伪文件）
         6. 管道文件 p			不占用磁盘存储（伪文件）
         7. 套接字 s			不占用磁盘存储（伪文件）
         8. 未知（unknow）

       9. 文件访问权限

          1. u：所有者： rwx 读写执行
          2. g：所属组： rwx 读写执行
          3. o：其他人： rwx 读写执行

  2. 硬链接计数：硬链接个数。

  3. 文件所有者

  4. 文件的所属组。

  5. 文件大小：单位字节。

     - 普通文件：实际大小
     - 目录文件：占用磁盘存储大小 (4K 整数倍)

  6. 时间：

     创建、或最后修改时间。

  7. 文件名。

### ls 命令的其他参

1. -a ： all，查询所有文件。
   - Linux系统中，以“.”开头的文件是 隐藏。
2. -d：查看目录本身信息。
   - ls -l 查询目录，默认查目录的字内容的详细信息。
3. -F：查询文件信息，带有类型提示符。
4. -i：查询文件的 inode（i）
5. -h：查询文件时，添加单位（M、k、G）

### 文件创建、删除

- 创建空文件： touch 文件名
- 创建空目录：mkdir 目录名
  - mkdir -p  a/b/c/d  一次性创建多级目录
- 删除空目录：rmdir 目录名。  如果目录非空，无法删除。
- rm命令：
  - -r：递归调用。
  - -f：强制执行。如果命令出错，不会终止
  - -i：以交互式方式，删除文件。
  - 删除文件： rm 文件名
  - 删除目录：rm -r 目录名
  - ==**rm删除的文件，不能恢复，慎重使用**==

### 链接文件

- 硬链接：
  - **创建语法**： ln 文件名 硬链接文件名
  - 特性：源文件和硬链接文件具备相同数据信息，同步更新。
  - 原理：每新增一个硬链接，增加一个“目录项”（增加一个访问扇区途径）。硬链接计数+1
  - 删除硬链接：每删除一个硬链接，目录项-1.当减为0时，操作系统可以重写分配。
  - 目录文件，不能创建硬链接。

- 软连接：
  - 相当于 win 快捷方式。
  - **语法：**ln -s 源文件/目录名   软连接文件名
  - 强调：建议使用 “绝对路径法” 创建软连接。可以随意移动。
  - 软连接文件名：是它所指向的文件的访问路径。
  - 可以给目录创建软连接。

### 文件的复制和移动

- **cp 复制文件**
  - 语法： cp 源文件/目录名  目标位置[/新文件或目录名]
  - -r ：拷贝目录时指定。
  - -a：拷贝目录时指定。保留文件原有属性
  - -i：交互式拷贝。
- **mv 移动、改名**：
  - 移动：mv 文件、目录名 	已经存在的目录名
  - 改名：mv 文件、目录名  不存在的 目录名

### 压缩解压

- 压缩：
  - 语法： tar -zcvf 压缩包名.tar.gz  压缩原材料。
  - 语法： tar -jcvf 压缩包名.tar.bz2  压缩原材料。
    - z：gzip压缩格式
    - j： bzip2 压缩给
    - c：创建压缩包
    - v：显示压缩过程
    - f：指定压缩文件名（最后一个参数）
- 解压：
  - 语法： tar -zxvf 压缩包名.tar.gz 
  - 语法： tar -jxvf 压缩包名.tar.bz2 
    - x：解压缩
    - -C：指定解压缩位置。如： tar -zxvf 压缩包名 -C 待解压缩目录位置。
  - file 命令可以查看文件类型。当没有指定.tar.gz 或 .tar.bz2 时，可以使用file查看。

### 其他命令

- \> :  输出重定向。
  - 语法：命令  \> 文件名。 文件不存在创建。存在覆盖。
  - \>> : 追加。
- cat：读取文件内容，显示到屏幕。
- tac ： 倒序显示文件内容。
- more、less：查看大文件
  - Enter键，显示一行
  - 空格键，显示一屏
  - less 使用 ↑↓  more不能使用
  - q、Ctrl-c 终止查看。
- head、tail：查看文件头部、尾部
  - head -N 文件名 		查看文件前 n 行
  - tail -N 文件名 		查看文件后 n 行
- “|” 管道命令：
  - 将前一个命令的输出，作为后一个命令输入。
  - ls -l | more
  - ps aux | grep 
- pwd：打印当前shell进程的工作目录位置
- which：查询命令的可执行文件所在目录。

## 用户管理

### 切换用户

- su 用户名： 切换登录用户，不改变工作路径。
- su - 用户名：切换登录用户，改变工作路径为 用户的 家目录
- sudo ： 临时获得管理员权限。sudo 对应的命令执行结束，权限消失。

### 添加删除用户

- 添加用户：sudo  addruser 用户名。
- 验证：查看 /etc/passwd 多出新用户信息。
- 删除用户：sudo deluser 用户名。

### 添加删除用户组：

- 添加用户组：sudo addgroup 组名

- 验证：查看 /etc/group 多出新组信息。

- 删除用户组：sudo delgroup 组名。

- 修改文件所有者：chown 新用户名 文件名。

  > chown  -R  gongqi:gongqi   xxx

- 修改文件所属组：chgrp 新组名 文件名。

### 修改用户密码

- passwd命令：修改当前登录用户密码。
- 新增用户默认，不能使用 “sudo” , 修改 配置文件 /etc/sudoers , 添加 与 root 用户格式完全一致的配置项。

### chmod修改文件权限

- 文字设定法：
  - u：所有者
  - g：所属组
  - o：其他人
  - a：所有
  - +：添加权限
  - -：删除权限
  - =：赋值权限
  - rwx：读、写、执行
- ==**数字设定法**==
  - 4	2	1
  - r    w    x
  - chmod 777 文件名。
- vim简单使用
  1. vim  文件名。 不存在创建，存在打开
  2. 按 i 。 左下角 -- 插入 -- 提示
  3. 编辑文本。（误按了 Ctrl-s 使用 Ctrl-q 解除）
  4. 按 esc
  5. 按 冒号。
  6. wq —— 保存退出。
  7. q! —— 强制退出。

### 软件版本降级

- sudo apt-get install aptitude 

  > 安装降级工具

-  sudo apt-cache madison  xxx 

  > 查看软件源中openssh-client的所有版本

-  sudo aptitude install 包名=包的版本号   

  > 降级

### Python共享文件

> python3  -m http.server

### ssh

>  sudo apt-get install openssh-server 
>
>  由于xshell远程连接ubuntu是通过ssh协议的，所以，需要给ubuntu安装ssh服务器。

### 查看硬件

```go
free -h    //查看内存
df -h       //查看磁盘的使用情况
sudo swapon --show  //查看是否有交换分区
lsblk
```

### scp

> scp  要传的文件  root@目标ip:路径    //上传到远程服务器
>
> scp root@目标ip:文件路径  本地位置  //下载到本地服务器

### telnet

> telnet  ip  port     //空格隔分

### locate 

 Locate  [选项] [参数]

功能:在mlocate数据库中搜索条目,用来快速查找文件或目录

示例:

> loacte less1    在各个目录下查找名为less1的这个文件或者文件夹
>
> locate newlocate  为了避免新建的文件夹找不到，可以立即更新数据库(updatedb命令)
>
> locate -i /home/sunjimeng/Documents/*Cate        在使用通配符时忽略大小写
>
> locate /home/sunjimeng/Documents/le    寻找以特定字符串开头的文件或文件夹

### netstat

sudo netstat -ap|grep 9108   //查看9108端口的进程

### du-查看文件大小

> du -h  -s xxx     //查看xxx文件夹的大小
>
> -h 以K，M，G为单位，提高信息的可读性
>
> -s 仅显示总计

### ls 查看当前目录下的文件个数

> ls -l | grep "^-" | wc -l    //普通文件的个数  （不包含文件夹）
>
> ls -l | grep "^d" | wc -l  //文件夹的个数

### DNS

>  sudo chattr +i /etc/resolv.conf   //永久修改DNS

### 分区

> fdisk  -l   // 查看硬盘信息
>
> fdisk  [磁盘路径]  //打开`fdisk`系统
>
> df  -h  //查看硬盘和挂载分区等信息

### 格式化&挂载

> sudo umount  [磁盘路径]      //格式化前必须umount
>
> mkfs.ext4  /dev/sdb1              //将`/dev/sdb1`格式化为`ext4`类型
>
> mount  /dev/sdb1  /[目录名]  //挂载
>
> **设置开机自动挂载**：
>
> 1、ls  -al  /dev/disk/by-uuid  //查询磁盘UUID
>
> 2、vim  /etc/fstab //编辑`/etc/fstab`(用来存放文件系统的静态信息的文件)
>
> 例：UID=e2c11cf7-e0ca-45f7-90e8-5e05b48f8917 /home/gongqi/MountFiles ext4 defaults,noatime 0 0
>
> 3、重启

### uname

> uname -a  或  cat /proc/version   //查看内核版本

### sed

```shell
sed -i 's/v3.0.0-8a16951-nolic/v3.0.0-44cba50/g'  *.yml
//替换制定字符串
sed -e "s/startStr//" //删除每行开头的字符串startStr
```

### awk

```shell
awk -F 'track_id":"' '{print $2}' output.json | awk -F '"' '{print "track_id:"$1}' > track_id.txt
//获得output.json中的字段track_id，保存至result.txt
```

### Argument list too long

```shell
find ./ -name "*" | xargs -i rm {}
或
find test/ -name "*.jpg" -exec rm {} \;
```

### set、env、export



set命令显示当前shell的变量（包括的私有变量以及用户变量，不同类的shell有不同的私有变量），包括当前用户的变量;

env命令显示当前用户的变量;

export命令显示当前导出成用户变量的shell变量。

每个shell有自己特有的变量（set）显示的变量，这个和用户变量是不同的，当前用户变量和你用什么shell无关，不管你用什么shell都在，比如HOME,SHELL等这些变量，但shell自己的变量不同shell是不同的，比如BASH_ARGC， BASH等，这些变量只有set才会显示，是bash特有的，export不加参数的时候，显示哪些变量被导出成了用户变量，因为一个shell自己的变量可以通过export “导出”变成一个用户变量。

### 挂载.img文件

> losetup -f     //查找第一个可用的loop设备
>
> sudo losetup /dev/loop6 uefi-ntfs.img    //关联loop设备与img文件
>
> sudo mount /dev/loop6 xxx     //挂载loop设备

### Tail

tail -f test.log

> 持续输出日志

tail -n +5 log2014.log

> 从第五行开始显示文件

tail -n 5 log2014.log

> 显示最后5行日志

### which

which命令的作用是，在PATH变量指定的路径中，搜索某个系统命令的位置，并且返回第一个搜索结果。 也就是说，使用which命令，就可以看到某个系统命令是否存在，以及执行的到底是哪一个位置的命令。 

### CAT EOF追加文字

cat >> /etc/hosts <<EOF 

192.168.8.10 k8s-master-01 

192.168.8.21 k8s-node-01 

192.168.8.22 k8s-node-02 

EOF

### file

file 命令可以查看文件类型

### curl cip.cc

查看电脑的公网Ip

### dig-域名信息搜索

dig gqfun.top +nostats +nocomments +nocmd
