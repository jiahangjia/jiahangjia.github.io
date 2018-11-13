---
layout: mypost
title: CentOS6.3搭建LNMP环境
categories: [linux,php]
---
CentOS6.3纯净系统

安装的软件`nginx-1.10.1` `mysql-5.6.31` `php-5.6.23`

## 准备

```shell
yum update
yum groupinstall -y "Development Tools"
```

## 安装nginx

```shell
wget http://nginx.org/download/nginx-1.10.1.tar.gz
tar -zxvf nginx-1.10.1.tar.gz
cd nginx-1.10.1
./configure
错误提示
./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.
解决错误
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz
tar -zxvf pcre-8.39.tar.gz
cd pcre-8.39
./configure
make
make install

cd nginx-1.10.1
./configure
又报错
./configure: error: the HTTP gzip module requires the zlib library.
You can either disable the module by using --without-http_gzip_module
option, or install the zlib library into the system, or build the zlib library
statically from the source with nginx by using --with-zlib=<path> option.
解决错误
wget http://zlib.net/zlib-1.2.8.tar.gz
tar -zxvf zlib-1.2.8.tar.gz
cd zlib-1.2.8
./configure
make
make install

cd nginx-1.10.1
./configure
make
make install
安装成功
```

### nginx配置

```shell
vi /usr/local/nginx/conf/nginx.conf

修改为
######################################
user  www-web www-web;
worker_processes  1;

error_log  logs/error.log;

pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
#空的虚拟主机，防止非绑定的域名和ip访问
    server {
        listen 80 default;
        server_name _;
        return 500;
    }
#虚拟主机test.tmaize.net
    server {
        listen       80;
        server_name  test.tmaize.net www.test.tmaize.net;
        charset utf-8;
        location /{
            root   /www.tmaize.net/test.tmaize.net;
            index  index.html index.htm index.php;
        }
    }
}
######################################
		
启动服务
cd /usr/local/nginx/sbin/
./nginx
报错
./nginx: error while loading shared libraries: libpcre.so.1: cannot open shared object file: No such file or directory
解决
cd /lib
ln -s libpcre.so.0.0.1 libpcre.so.1

继续启动服务
cd /usr/local/nginx/sbin/
./nginx
又报错
nginx: [emerg] getpwnam("www-web") failed in /usr/local/nginx/conf/nginx.conf:1
解决 创建www-web用户
groupadd www-web
useradd -r -g www-web www-web
usermod -s /sbin/nologin www-web

继续启动服务
cd /usr/local/nginx/sbin/
./nginx
./nginx -s reload
服务启动成功，最好在/etc/init.d/下写一个脚本方便管理启动
打开test.tmaize.net显示/www.tmaize.net/test.tmaize.net里的页面

更改权限
chown -R www-web /www.tmaize.net/
chgrp -R www-web /www.tmaize.net/

至此mysql配置完成
```

## 安装Mysql

```shell
清除mysql相关
rpm -qa |grep -i mysql  //先查看系统下有哪些包含MySQL字符串的包
rpm -ev --nodeps mysql-libs-5.1.61-4.el6.i686

下载
到Mysql官网http://www.mysql.com/downloads/
选择Community 版本
选择MySQL Community Server (GPL)
Select Platform:Source Code
选择倒数第二个
Generic Linux (Architecture Independent), Compressed TAR Archive
(mysql-5.6.31.tar.gz)
wget http://dev.mysql.com/get/Downloads/MySQL-5.6/mysql-5.6.31.tar.gz

之前的yum groupinstall -y "Development Tools"好像没有cmake工具
yum install cmake bison-devel ncurses-devel autoconf automake zlib* fiex* libxml* libmcrypt* libtool-ltdl-devel*

cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DMYSQL_UNIX_ADDR=/var/lib/mysql/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci

make  //大概需要20分钟
make install
```

### 配置mysql

```shell
配置用户及权限
groupadd mysql
useradd -r -g mysql mysql
usermod -s /sbin/nologin mysql
chown -R mysql:mysql /usr/local/mysql

mysql配置
cd /usr/local/mysql
scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --user=mysql
//最后的显示New default config file was created as /usr/local/mysql/my.cnf

注：在启动MySQL服务时，会按照一定次序搜索my.cnf，先在/etc目录下找，找不到则会搜索"$basedir/my.cnf"
这里使用/usr/local/mysql/my.cnf，这是新版MySQL的配置文件的默认位置！
注意：在CentOS 6.4版操作系统的最小安装完成后，在/etc目录下会存在一个my.cnf，需要将此文件更名为其他的名字
如：/etc/my.cnf.bak，否则，该文件会干扰源码安装的MySQL的正确配置，造成无法启动。
在使用"yum update"更新系统后，需要检查下/etc目录下是否会多出一个my.cnf
如果多出，将它重命名成别的。否则，MySQL将使用这个配置文件启动，可能造成无法正常启动等问题。

添加PATH
vi /etc/profile
在文件最后添加
---
PATH=/usr/local/mysql/bin:$PATH
export PATH
---
source /etc/profile  //修改生效

添加服务
拷贝服务脚本到init.d目录
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
chmod +x /etc/init.d/mysql
service mysql start
chkconfig mysql on  //设置开机启动可选

配置用户
mysql -u root
SET PASSWORD = PASSWORD('设置密码');
exit
mysql -u root -p  //输入刚才的密码登陆
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '密码' WITH GRANT OPTION;
FLUSH privileges; //更新权限
在本地同Navicat登陆试一下，顺便删除多余的用户，留一个root@%就可以了

配置防火墙，前面貌似关掉过了
至此mysql配置完成，更多配置请修改配置文件
```

### mysql优化

```shell
安装之后发现mysql的内存占用大概在400M左右，对1G内存的机器实在是吃力
下面通过修改mysql配置文件达到降低内存占用的目的

vi /usr/local/mysql/my.cnf

# 一些优化
port= 3306
character-set-server=utf8
performance_schema_max_table_instances=200
table_definition_cache=200
table_open_cache=128
innodb=OFF
default-storage-engine=MYISAM
default-tmp-storage-engine=MYISAM
loose-innodb-trx=0
loose-innodb-locks=0
loose-innodb-lock-waits=0
loose-innodb-cmp=0
loose-innodb-cmp-per-index=0
loose-innodb-cmp-per-index-reset=0
loose-innodb-cmp-reset=0
loose-innodb-cmpmem=0
loose-innodb-cmpmem-reset=0
loose-innodb-buffer-page=0
loose-innodb-buffer-page-lru=0
loose-innodb-buffer-pool-stats=0
loose-innodb-metrics=0
loose-innodb-ft-default-stopword=0
loose-innodb-ft-inserted=0
loose-innodb-ft-deleted=0
loose-innodb-ft-being-deleted=0
loose-innodb-ft-config=0
loose-innodb-ft-index-cache=0
loose-innodb-ft-index-table=0
loose-innodb-sys-tables=0
loose-innodb-sys-tablestats=0
loose-innodb-sys-indexes=0
loose-innodb-sys-columns=0
loose-innodb-sys-fields=0
loose-innodb-sys-foreign=0
loose-innodb-sys-foreign-cols=0
join_buffer_size = 64M
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
#结束

新开窗口监视内存变化
watch -n 2 -d free -h  //新窗口
service mysql start  //旧窗口
发现占用70M左右，比之前的400M下降了许多
```


## php安装前准备

```shell
把php相关的卸载干净
yum remove php
到php官网http://php.net/下载php源码
wget http://cn.php.net/distributions/php-5.6.23.tar.gz

添加repoforge源
官网http://repoforge.org/  选择usage EL 6: i686
wget http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.i686.rpm
rpm -ivh rpmforge-release-0.5.3-1.el6.rf.i686.rpm

更新源
yum clean all
yum makecache
yum list

安装需要的库 干脆一次性装个全的
yum install -y apr* autoconf automake bison bison-devel bzip2 bzip2* cloog-ppl compat* cpp 
yum install -y curl curl-devel fontconfig fontconfig-devel freetype freetype* 
yum install -y freetype-devel gcc gcc-c++ gtk+-devel gd gettext gettext-devel 
yum install -y glibc kernel kernel-headers keyutils keyutils-libs-devel krb5-devel 
yum install -y libcom_err-devel libpng libpng* libpng-devel libjpeg* libsepol-devel 
yum install -y libselinux-devel libstdc++-devel libtool* libgomp libxml2 libxml2-devel libXpm* libX* 
yum install -y libtiff libtiff* make mpfr ncurses* ntp openssl nasm nasm* 
yum install -y openssl-devel patch pcre-devel perl php-common php-gd yasm
yum install -y policycoreutils ppl telnet t1lib t1lib* wget zlib-devel readline-devel
yum install -y libxml2-devel libjpeg-devel libpng-devel openssl-devel freetype-devel libmcrypt-devel libcurl-devel
yum install -y zlib-devel libmcrypt-devel  mhash-devel  libcurl-devel bzip2-devel readline-devel libedit-devel sqlite-devel

```

### php安装

```shell
./configure \
--prefix=/usr/local/php5.6 \
--with-config-file-path=/usr/local/php5.6/etc \
--enable-inline-optimization \
--disable-debug \
--disable-rpath \
--enable-shared \
--enable-opcache \
--enable-fpm \
--with-fpm-user=www-web \
--with-fpm-group=www-web \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-gettext \
--enable-mbstring \
--with-iconv \
--with-mcrypt \
--with-mhash \
--with-openssl \
--enable-bcmath \
--enable-soap \
--with-libxml-dir \
--enable-pcntl \
--enable-shmop \
--enable-sysvmsg \
--enable-sysvsem \
--enable-sysvshm \
--enable-sockets \
--with-curl \
--with-zlib \
--enable-zip \
--with-bz2 \
--with-readline

没有报错 如果报错请安装缺少的库文件

make  //大概15分钟
make install

cp php.ini-production /usr/local/php5.6/etc/php.ini  //复制php配置文件到安装目录
rm -rf /etc/php.ini  //删除系统自带配置文件
ln -s /usr/local/php5.6/etc/php.ini /etc/php.ini  //添加软链接到 /etc目录
cp /usr/local/php5.6/etc/php-fpm.conf.default /usr/local/php5.6/etc/php-fpm.conf  //拷贝模板文件为php-fpm配置文件
ln -s /usr/local/php5.6/etc/php-fpm.conf /etc/php-fpm.conf  //添加软连接到 /etc目录

vi /etc/php-fpm.conf  //去掉pid = run/php-fpm.pid的注释

添加php-fpm启动脚本
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm5.6
chmod +x /etc/init.d/php-fpm5.6

vi /usr/local/php5.6/etc/php.ini

date.timezone = PRC #设置时区
short_open_tag = On #支持php短标签
opcache.enable=1 #php支持opcode缓存
opcache.enable_cli=0
在最后一行添加：zend_extension=opcache.so #开启opcode缓存功能

添加 PHP 命令到环境变量
vi /etc/profile
修改最后几行：PATH=$PATH:/usr/local/mysql/bin:/usr/local/php5.6/bin 保存
php -v  //显示PHP版本

配置nginx支持php
#虚拟主机test.tmaize.net
    server {
        listen       80;
        server_name  test.tmaize.net www.test.tmaize.net;
        charset utf-8;
        location /{
            root   /www.tmaize.net/test.tmaize.net;
            index  index.html index.htm index.php;
        }
        location ~ \.php$ {
            root   /www.tmaize.net/test.tmaize.net;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /www.tmaize.net/test.tmaize.net$fastcgi_script_name;
            include        fastcgi_params;
        }

至此php基本配置完成
```

## 总结


以上只是一些简单的安装步骤，并没有做详细的配置
在具体应用中还会遇到好多问题，遇到问题就多Google

