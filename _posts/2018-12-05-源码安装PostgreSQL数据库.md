---
layout: post
title: "源码安装PostgreSQL数据库的小坑"
date: 2018-12-05 17:33
categories: [postgres]
tags: [postgres]
---

最近打算学习下PG数据库,源码方式安装.操作步骤没啥好说的,按照官方文档来就行.这里主要是记录`configure`过程中的一些小坑.

# configure
---
PG的configure提供了非常丰富的选项,考虑到以后可能会需要`SSL`加密，在实际的编译过程中,用到了`--with-openssl`选项.在configure
的过程中,操作系统会检查系统是否安装了openssl的库,是否包含相关的头文件等.若要查询是否安装相关库文件,以CentOS为例,可以使用如下
命令:`rpm -qa | grep libname`来查询,**但是,有了这个库不一定就万事大吉了,还需要有相关库的头文件.**在实际编译过程中主要遇到以
下两类问题.

# 找不到库文件
---
通过rpm命令查询库是安装成功了的,在`/usr/lib/`目录下也有相关的库,怎么还是报错呢?这里要注意,库名要精确匹配.例如我的机器上ssl相关
的库名是`libssl.so.1.0.2k`,但是configure在链接时用的参数是`-lssl`,这意味着库的名字必须是`libssl.so`.解决方法也很简单,创建一个
软连接指向我们要的库即可:`ln -s libssl.so.1.0.2k libssl.so`.

# 找不到头文件
---
还一种错误是找不到头文件,这些找不到的头文件往往都是第三方库的,比如`readline.h`头文件.这个时候先在系统相关的`include`目录下查找是否
确实没有该文件,如果没有,那就必须安装.
1. 查找该库的相关信息:`yum search readline`,这时可能会看到如下输出:
```bash
[root@localhost lib64]# yum search readline
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.cn99.com
============================= N/S matched: readline =======================================
readline-devel.i686 : Files needed to develop programs which use the readline library
readline-devel.x86_64 : Files needed to develop programs which use the readline library
readline-static.i686 : Static libraries for the readline library
readline-static.x86_64 : Static libraries for the readline library
perl-Term-UI.noarch : Term::ReadLine user interface made easy
readline.i686 : A library for editing typed command lines
readline.x86_64 : A library for editing typed command lines

  Name and summary matches only, use "search all" for everything.
```
这里会发现有很多相关的,有些是跟操作系统架构相关的,比如`x86_64`是`64位`操作系统的库,另外有些是带有`devel`标志的,还一种是不带的,
比如`readline.x86_64`.**带有devel标志的在安装过程中会将相关头文件一起安装,而不带这个标志的则不会.**我的机器是64位,所以选择
`readline-devel.x86_64`进行安装.
