---
path: how-to-install-httpd-on-linux
title: 在linux中使用源码安装httpd服务器
date: 2017-03-08 11:21:10
updated: 2017-03-08 11:21:10
categories:
- Linux
tags:
- linux
- httpd
---

前天在Centos中安装了Apache的httpd，安装的机器在公司内网，于是选择了源码进行安装。俗话说好记性不如烂笔头，现将安装过程进行记录，也希望能帮到各位网友。<!--more-->
####准备工作
因公司机器已经安装c++编译相关，该工作不再赘述，如无法使用make相关命令，请自行安装g++、libc等库。
1. 笔者写这篇博文时选中的版本是[httpd-2.4.23.tar.gz](http://mirrors.tuna.tsinghua.edu.cn/apache/httpd/httpd-2.4.23.tar.gz)

2. Apache Portable Runtime（APR），为上层的应用程序提供一个可以跨平台使用的底层支持接口库。笔者选定的版本仍为官网最新版本[apr-1.5.2.tar.gz](http://mirrors.tuna.tsinghua.edu.cn/apache/apr/apr-1.5.2.tar.gz)

3. [apr-util-1.5.4.tar.gz](http://mirrors.hust.edu.cn/apache/apr/apr-util-1.5.4.tar.gz)

4. pcre，该模块主要为httpd提供了rewrite功能，笔者选定了[pcre-8.38.tar.gz](ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.38.tar.gz)

5. [lynx](http://invisible-mirror.net/archives/lynx/tarballs/lynx2.8.8rel.2.tar.gz)及其依赖[ncurses](http://ftp.gnu.org/pub/gnu/ncurses/ncurses-6.0.tar.gz)，其中lynx是纯文本浏览器，httpd的执行status命令时会访问server-status，lynx用于解析html并输出文本信息，它依赖于ncurses，curses库是可以在Linux终端中写出字符用户界面，新的版本是ncurses库。不安装lynx及ncurses也可以，使用curl访问server-status链接即可。

####安装过程
分别使用tar -xvzf命令解压缩各tar.gz，笔者将各压缩包解压在/opt/downloads下，依次执行以下的安装命令。
1. 安装ncurses(不使用lynx请跳过)
``` sh
cd /opt/downloads/ncurses-6.0/
./configure --prefix=/usr/local/ncurses
make
make install
```
2. 安装lynx
``` sh
cd /opt/downloads/lynx2-8-8/
./configure --prefix=/usr/local/lynx --with-curses-dir=/usr/local/ncurses
make
make install
```
3. 安装apr
``` sh
cd /opt/downloads/apr-1.5.2
./configure --prefix=/usr/local/apr
make
make install
```
4. 安装apr-util
``` sh
cd /opt/downloads/apr-util-1.5.4
./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr
make
make install
```
5. 安装pcre
``` sh
cd /opt/downloads/pcre-8.38
./configure --with-apr=/usr/local/pcre
make
make install
```
6. 安装httpd
``` sh
cd /opt/downloads/httpd-2.4.23
./configure --prefix=/usr/local/apache2 --enable-expires --enable-headers --enable-modules=most --enable-so --with-mpm=worker --enable-rewrite --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util --with-pcre=/usr/local/pcre
make
make install
```
至此，httpd安装完毕。

>上面命令所使用到的`./configure`后的参数说明：（可执行`./configure --help`查看支持的参数，在执行./configure命令后，可使用`echo $?`查看是否有错误，返回0说明没问题，可继续执行make命令）

>`#--prefix=<install_path>` 指定便以后的二进制文件安装目录，若省略使用默认目录

>`#--with-xxx` 一般指定其加载的模块路径

>`--enable-module=so` 指明编译动态加载模块，使得apache的各模块与核心分开编译，运行时动态加载，最新版已缺省编译此模块

>`--enable-deflate` 支持网页压缩

>`--enable-expires` 通过配置文件控制HTTP的`“Expires:”`和`“Cache-Control:”`头内容，即对网站图片、js、css等内容，提供客户端浏览器缓存的设置

>`--enable-rewrite` 支持URL重写

>以下为本次未使用的参数说明：

>`--enable-cache` 支持缓存

>`--enable-file-cache` 支持文件缓存

>`--enable-mem-cache` 支持记忆缓存

>`--enable-disk-cache` 支持磁盘缓存

>`--enable-static-support` 支持静态连接(默认为动态连接)

>`--enable-static-htpasswd` 使用静态连接编译 htpasswd - 管理用于基本认证的用户文件

>`--enable-static-htdigest` 使用静态连接编译 htdigest - 管理用于摘要认证的用户文件 

>`--enable-static-rotatelogs` 使用静态连接编译 rotatelogs - 滚动 Apache 日志的管道日志程序 

>`--enable-static-logresolve` 使用静态连接编译 logresolve - 解析 Apache 日志中的IP地址为主机名

>`--enable-static-htdbm` 使用静态连接编译 htdbm - 操作 DBM 密码数据库 

>`--enable-static-ab` 使用静态连接编译 ab - Apache HTTP 服务器性能测试工具

>`--enable-static-checkgid` 使用静态连接编译 checkgid 

>`--disable-cgid` 禁止用一个外部 CGI 守护进程执行CGI脚本

>`--disable-cgi` 禁止编译 CGI 版本的 PHP

>`--disable-userdir` 禁止用户从自己的主目录中提供页面

####配置httpd
1. 修改httpd.conf文件
> 去除`ServerName`的注释，并修改设置其值，如`localhost:80`
>若要开启server-status监控httpd的运行状态，需在httpd.conf中打开对httpd-info.conf的引用，并修改http-info.conf的相关配置，参照如下
>```
<Location /server-status>
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1        
</Location>
>```
1. 使用一下命令注册httpd为service
>```  sh
cp /usr/local/apache2/bin/apachectl /etc/init.d/httpd
```
编辑/etc/init.d/httpd文件，在注释的顶部添加chkconfig的配置
```
# chkconfig:2345 90 90
# description:Apache
```
并为该文件添加可执行的权限
```  sh
chmod +x /etc/init.d/httpd
```
添加httpd为服务
```  sh
chkconfig --add httpd
```
>现在可以使用`service httpd start|stop|status`等命令操作了

Just enjoy it！