---
layout: post
category: ['Linx', 'Nginx','PHP','Redis','MySQL']
title: Centos6.8 LNMP 编译安装
---

##1. Nginx 安装配置：

- 安装依赖扩展

```
yum install apr* curl curl-devel gcc gcc-c++ openssl openssl-devel pcre-devel gd gd-devel kernel keyutils patch perl libXpm* libvpx libjpeg libpng libXpm libXpm-devel libxml2 libxml2-devel freetype freetype-devel libpng* libpng-devel  ncurses* ncurses-devel libtool* libtool-libs glibc glibc-devel libevent libevent-devel libidn libidn-devel net-snmp* net-snmp-devel -y
```
- 安装工具包

```
yum groupinstall "Development Tools"
```

- 建立ngnix运行的用户组与用户名

```
groupadd nginx
useradd -g nginx nginx
#下载最新nginx版本
wget http://nginx.org/download/nginx-1.15.2.tar.gz
tar xf nginx-1.15.2.tar.gz && cd nginx-1.15.2

./configure --prefix=/usr/local/nginx --sbin-path=/usr/local/nginx/sbin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx/nginx.pid --lock-path=/var/lock/nginx.lock --user=nginx --group=nginx --with-http_ssl_module --with-http_flv_module --with-http_stub_status_module --with-http_gzip_static_module --http-client-body-temp-path=/var/tmp/nginx/client/ --http-proxy-temp-path=/var/tmp/nginx/proxy/ --http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ --http-uwsgi-temp-path=/var/tmp/nginx/uwsgi --http-scgi-temp-path=/var/tmp/nginx/scgi --with-pcre --with-file-aio --with-http_image_filter_module

make && make install
```
- 安装完创建启动文件

```
vim /etc/init.d/nginx

```
```
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15 
# description:  Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid
   
# Source function library.
. /etc/rc.d/init.d/functions
   
# Source networking configuration.
. /etc/sysconfig/network
   
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
   
nginx="/usr/local/nginx/sbin/nginx"
prog=$(basename $nginx)
   
NGINX_CONF_FILE="/etc/nginx/nginx.conf"
   
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
   
lockfile=/var/lock/subsys/nginx
   
make_dirs() {
   # make required directories
   user=`nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
	   if [ `echo $opt | grep '.*-temp-path'` ]; then
		   value=`echo $opt | cut -d "=" -f 2`
		   if [ ! -d "$value" ]; then
			   # echo "creating" $value
			   mkdir -p $value && chown -R $user $value
		   fi
	   fi
   done
}
   
start() {
	[ -x $nginx ] || exit 5
	[ -f $NGINX_CONF_FILE ] || exit 6
	make_dirs
	echo -n $"Starting $prog: "
	daemon $nginx -c $NGINX_CONF_FILE
	retval=$?
	echo
	[ $retval -eq 0 ] && touch $lockfile
	return $retval
}
   
stop() {
	echo -n $"Stopping $prog: "
	killproc $prog -QUIT
	retval=$?
	echo
	[ $retval -eq 0 ] && rm -f $lockfile
	return $retval
}
   
restart() {
	configtest || return $?
	stop
	sleep 1
	start
}
   
reload() {
	configtest || return $?
	echo -n $"Reloading $prog: "
	killproc $nginx -HUP
	RETVAL=$?
	echo
}
   
force_reload() {
	restart
}
   
configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}
   
rh_status() {
	status $prog
}
   
rh_status_q() {
	rh_status >/dev/null 2>&1
}
   
case "$1" in
	start)
		rh_status_q && exit 0
		$1
		;;
	stop)
		rh_status_q || exit 0
		$1
		;;
	restart|configtest)
		$1
		;;
	reload)
		rh_status_q || exit 7
		$1
		;;
	force-reload)
		force_reload
		;;
	status)
		rh_status
		;;
	condrestart|try-restart)
		rh_status_q || exit 0
			;;
	*)
		echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
		exit 2
esac
```

- 修改访问权限

```
chmod +x /etc/init.d/nginx
```
- 添加开机启动

```
chkconfig --add nginx
```
- 开启nginx

```
service nginx start
```

- 修改配置文件

```
vim /etc/nginx/nginx.conf
```
```
user  nginx nginx;
worker_processes  1; #查看cpu信息  /proc/cpuinfo “cpu id” 确定CPU个数

error_log  /var/log/nginx/error.log crit;
events {
	use epoll;
	multi_accept on;
	worker_connections  65535;
}

http {
	include       mime.types;
	default_type  application/octet-stream;

	sendfile        on;
	tcp_nopush      on;
	tcp_nodelay     on;

	keepalive_timeout  20;

	gzip  on;
	gzip_disable "MSIE [1-6]\.(?!.*SV1)";

	client_header_buffer_size    1k;
	large_client_header_buffers  4 4k;

	fastcgi_connect_timeout 300;
	fastcgi_send_timeout 300;
	fastcgi_read_timeout 300;
	fastcgi_buffer_size 64k;
	fastcgi_buffers 16 64k;
	fastcgi_busy_buffers_size 128k;
	fastcgi_temp_file_write_size 128k;
	fastcgi_intercept_errors on;

	open_file_cache          max=5000  inactive=20s;
	open_file_cache_valid    30s;
	open_file_cache_min_uses 2;
	open_file_cache_errors   off;


	server {
		listen       80;
		server_name  localhost;
		root   /www/;
		index  index.php index.html index.htm;

		if (!-e $request_filename){
			rewrite ^/(.*) /index.php last;
		}

		location ~ \.php$ {
			fastcgi_pass   127.0.0.1:9000;
			fastcgi_index  index.php;
			fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
			include        fastcgi_params;
		}
	}
}	
```

**到此Nginx安装完成**
##2. PHP 编译安装
- 安装 libmcrypt 扩展库

```
yum install libmcrypt libmcrypt-devel mcrypt mhash -y
```

 - 下载PHP文件
 
```
wget http://cn2.php.net/get/php-7.2.8.tar.gz/from/this/mirror
tar zxvf mirror && cd php-7.2.8
```
- 开始编译源文件

```
./configure --prefix=/usr/local/php --with-config-file-path=/usr/local/php/etc --with-mysqli --with-pdo-mysql --with-gd --with-png-dir=/usr/local/libpng --with-jpeg-dir=/usr/local/jpeg --with-freetype-dir=/usr/local/freetype --with-xpm-dir=/usr/ --with-zlib-dir=/usr/local/zlib --with-iconv --enable-libxml --enable-xml --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --enable-opcache --enable-mbregex --enable-fpm --enable-mbstring --enable-ftp  --with-openssl --enable-pcntl --enable-sockets --with-xmlrpc --enable-zip --enable-soap --without-pear --with-gettext --enable-session --with-curl

make

make install
```
- 复制php配置文件到安装目录

```
cp php.ini-production /usr/local/php/etc/php.ini
```

- 删除系统自带配置文件

```
rm -rf /etc/php.ini 
```

- 添加软链接到 /etc目录

```
ln -s /usr/local/php/etc/php.ini /etc/php.ini
```

- 拷贝模板文件为php-fpm配置文件

```
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
ln -s /usr/local/php/etc/php-fpm.conf /etc/php-fpm.conf
```

- 拷贝php-fpm到启动目录

```
cp sapi/fpm/init.d.php-fpm /etc/rc.d/init.d/php-fpm
cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
```

- 添加执行权限 

```
chmod +x /etc/rc.d/init.d/php-fpm
```

- 设置开机启动

```
chkconfig php-fpm on
```

- 启动

``` 
service php-fpm start
```

##3. 编译安装redis扩展
- 下载redis最新版本

```
wget http://pecl.php.net/get/redis-4.1.0.tgz
tar zxvf redis-4.1.0.tgz && cd redis-4.1.0
```

- 生成配置文件

```
/usr/local/php/bin/phpize
```
- 编译redis

```
./configure --with-php-config=/usr/local/php/bin/php-config

make && make install
```
- 修改php.ini

```
extension = redis.so
```
##4. YUM 安装 MySql（编译的太累）
 - 使用yum命令安装

```
yum install mysql mysql-devel mysql-server
```

- 修改MySql配置

```
vim /etc/my.cnf 
#设置编码
default-character-set=utf8
```

- 添加开机启动

```
chkconfig --add mysqld
```
- 启动MySql

```
service mysqld start
```
- 配置MySql

```
mysqladmin -u root password 123456

mysql -uroot -p
```
- 远程访问

```
update user set host = '%' where user = 'root';
```
如果报错不用管，切换到mysql数据库，执行：

![](/res/img/in_posts/2018/sql_install.png)

##5. Redis服务器搭建

- 下载、安装redis server版本

```
wget http://download.redis.io/releases/redis-stable.tar.gz
tar xf redis-stable.tar.gz && cd redis-stable
make
```

- 配置

```
cp src/redis-server /usr/local/bin/
cp src/redis-cli /usr/local/bin/

mkdir /etc/redis
mkdir -p /var/redis/log
mkdir /var/redis/run
mkdir /var/redis/redis

cp redis.conf /etc/redis/redis.conf

```
- 修改配置

```
vim /etc/redis/redis.conf
```

```
daemonize yes
pidfile /var/redis/run/redis.pid
logfile /var/redis/log/redis.log
dir /var/redis/redis
```

- 运行redis

```
redis-server /etc/redis/redis.conf
```
- 查看运行状态

```
ps -ef | grep redis
```

##6. 废话两句
- 为什么不用**`centos7.x`**
 - 因为莫名其妙的BUG多啊。
- 为什么要写这篇博客
 - 新买了一台云服务器，配置的时候顺手记录一下。
- 为什么...
 - PHP是世界上最好的语言。 

















