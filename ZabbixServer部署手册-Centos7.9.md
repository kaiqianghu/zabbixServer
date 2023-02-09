> 环境描述
> 系统镜像：CentOS-7-x86_64-Everything-2009.iso
> zabbix：zabbix-5.0.31.tar.gz
> nginx：nginx-1.22.1.tar.gz
> php(版本必须是7.2.0以上)：php-7.4.32.tar.gz
> mysql：mysql-boost-5.7.39.tar.gz


相关源码包都提前上传到了/root/下
# 一、部署LNMP
## 1.1 安装nginx依赖包及其服务
```shell
# 1、安装依赖
yum -y install gcc  gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel lua-devel

# 2、解压源码包至root
tar xf /root/nginx-1.22.1.tar.gz -C /root/

tar zxf /root/zlib-1.2.11.tar.gz -C /usr/local/src/

tar zxf /root/openssl-1.0.2k.tar.gz -C /usr/local/src/

tar zxf /root/ngx_devel_kit-0.3.1rc1.tar.gz -C /usr/local/src/

tar zxf /root/lua-nginx-module-0.10.13.tar.gz -C /usr/local/src/


# 3、创建服务管理用户
useradd nginx -s /sbin/nologin

# 4、创建zabbix数据目录（所有服务都安装此目录下）
mkdir -p /usr/local/zabbix

# 5、编译安装nginx
cd /root/nginx-1.22.1

./configure --prefix=/usr/local/zabbix/nginx_zabbix     --user=nginx     --group=nginx     --with-http_stub_status_module     --without-http_memcached_module     --with-http_ssl_module     --with-http_gzip_static_module     --with-openssl=/usr/local/src/openssl-1.0.2k     --with-zlib=/usr/local/src/zlib-1.2.11      --with-pcre     --add-module=/usr/local/src/ngx_devel_kit-0.3.1rc1     --add-module=/usr/local/src/lua-nginx-module-0.10.13

make && make install
```
## 1.2 配置nginx.conf
```shell
# 1、备份原始conf
cp -a /usr/local/zabbix/nginx_zabbix/conf/nginx.conf /usr/local/zabbix/nginx_zabbix/conf/nginx.conf.bak

# 2、修改nginx.conf（设置用户为deploy,并将php的location取消注释）
vi /usr/local/zabbix/nginx_zabbix/conf/nginx.conf

user nginx nginx;
worker_processes 32;
......
        location / {
            root   html;
            index  index.html index.htm index.php;
        }

......
        location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
......

# 3、验证配置文件
/usr/local/zabbix/nginx_zabbix/sbin/nginx -t

# 4、正常输出情况如下图👇
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1675865700858-34f6314a-3c2a-409c-8e80-aaa162f3e832.png#averageHue=%23120d09&clientId=uf31a22a6-dae2-4&from=paste&height=68&id=Sj7uV&name=image.png&originHeight=68&originWidth=680&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7099&status=done&style=none&taskId=u822beb7c-8bcd-4e83-af11-11f20e44a8e&title=&width=680)
## 1.3 配置systemctl管理nginx
vi /usr/lib/systemd/system/nginx-zabbix.service
```shell
[Unit]
Description=The NGINX HTTP server
After=syslog.target network-online.targetnss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/zabbix/nginx_zabbix/logs/nginx.pid
ExecStartPre=/usr/local/zabbix/nginx_zabbix/sbin/nginx -t
ExecStart=/usr/local/zabbix/nginx_zabbix/sbin/nginx
ExecReload=/usr/local/zabbix/nginx_zabbix/sbin/ -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
## 1.4 启动nginx
```shell
systemctl daemon-reload

# 设置服务开机自启并启动
systemctl enable --now nginx-zabbix

# 查看服务启动状态以及端口存活状态
systemctl status nginx-zabbix && ss -ntulp | grep 80

# 查看可否正常访问nginx，返回结果如下图👇
curl http://192.168.101.201
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1675949399022-34838173-9e22-42e4-bf29-f1ee70cf2e40.png#averageHue=%23040202&clientId=u1f5e3776-1b2d-4&from=paste&height=492&id=u7b633750&name=image.png&originHeight=492&originWidth=750&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28248&status=done&style=none&taskId=uc920c6f4-8110-42ee-ab5f-a8d9aeedbc6&title=&width=750)

# 二、部署PHP解释器
## 1.1 安装php依赖包及其服务
```shell
# 1、安装依赖包
yum install -y gcc gcc-c++ make zlib zlib-devel pcre pcre-devel libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel ncurses ncurses-devel curl curl-devel e2fsprogs e2fsprogs-devel openssl openssl-devel sqlite-devel libxslt-devel openldap openldap-devel

# 2、解压源码包至root
tar xf /root/php-7.2.26.tar.gz -C /root

# 3、源码编译安装php
cd /root/php-7.2.26

./configure --prefix=/usr/local/zabbix/php_zabbix --with-config-file-path=/usr/local/zabbix/php_zabbix/etc --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-fpm-user=nginx --with-fpm-group=nginx --with-gd --with-jpeg-dir --with-png-dir --with-freetype-dir --with-iconv-dir --with-libxml-dir --with-zlib-dir --with-curl --with-gettext --with-openssl --with-mhash --with-xmlrpc --with-xsl --without-pear --with-xpm-dir=no --enable-libxml --enable-xml --enable-bcmath --enable-mbstring --enable-mbregex --enable-sockets --enable-ctype --enable-session --enable-shmop --enable-sysvsem --enable-inline-optimization --enable-fpm --enable-pcntl --enable-soap --enable-short-tags --enable-static --enable-ftp --disable-ipv6 --enable-exif --enable-opcache --enable-zip


# 4、判断是否编译成功，输出0为成功
echo $?

# 5、安装
make && make install
```
## 1.2 配置php
### 1.2.1 启动脚本
```shell
cp /root/php-7.2.26/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm

chmod +x /etc/init.d/php-fpm
```
### 1.2.2 配置文件
```shell
# 1、配置php-fpm.conf
cp -a /usr/local/zabbix/php_zabbix/etc/php-fpm.conf.default /usr/local/zabbix/php_zabbix/etc/php-fpm.conf

sed -e 's#;pid = run/php-fpm.pid#pid = run/php-fpm.pid#g' -i /usr/local/zabbix/php_zabbix/etc/php-fpm.conf

# 2、配置www.conf
cp -a /usr/local/zabbix/php_zabbix/etc/php-fpm.d/www.conf.default /usr/local/zabbix/php_zabbix/etc/php-fpm.d/www.conf

sed -e 's/pm.max_children = 5/pm.max_children = 50/g' -e 's/pm.start_servers = 2/pm.start_servers = 20/g' -e 's/pm.min_spare_servers = 1/pm.min_spare_servers = 5/g' -e 's/pm.max_spare_servers = 3/pm.max_spare_servers = 35/g' -i /usr/local/zabbix/php_zabbix/etc/php-fpm.d/www.conf

# 3、配置php.ini
cp -a /root/php-7.2.26/php.ini-production /usr/local/zabbix/php_zabbix/etc/php.ini

sed -r -e 's/^(short_open_tag =).*/\1 On/g' -e 's/^(disable_functions =).*/\1 passthru,exec,system,chroot,chgrp,chown,shell_exec,proc_open,proc_get_status,ini_alter,ini_alter,ini_restore,dl,openlog,syslog,readlink,symlink,popepassthru,stream_socket_server,escapeshellcmd,dll,popen,disk_free_space,checkdnsrr,checkdnsrr,getservbyname,getservbyport,disk_total_space,posix_ctermid,posix_get_last_error,posix_getcwd, posix_getegid,posix_geteuid,posix_getgid, posix_getgrgid,posix_getgrnam,posix_getgroups,posix_getlogin,posix_getpgid,posix_getpgrp,posix_getpid, posix_getppid,posix_getpwnam,posix_getpwuid, posix_getrlimit, posix_getsid,posix_getuid,posix_isatty, posix_kill,posix_mkfifo,posix_setegid,posix_seteuid,posix_setgid, posix_setpgid,posix_setsid,posix_setuid,posix_strerror,posix_times,posix_ttyname,posix_uname/g' -e 's/^(expose_php =).*/\1 Off/g' -e 's/^(max_execution_time =).*/\1 600/g' -e 's/^(max_input_time =).*/\1 600/g' -e 's/^(memory_limit =).*/\1 1024M/g' -e 's/^(post_max_size =).*/\1 32M/g' -e 's/^;(always_populate_raw_post_data =).*/\1 -1/' -e 's/^(upload_max_filesize =).*/\1 2M/g' -e 's/^(default_socket_timeout =).*/\1 120/g' -e 's/^;(date.timezone =).*/\1 Asia\/Shanghai/' -e 's/^(mysql.connect_timeout =).*/\1 120/g' -e 's/^;(opcache.enable=).*/\11/' -e 's/^;(opcache.enable_cli=).*/\10/' -e '/;curl.cainfo =/azend_extension=opcache.so' -e 's/^(memory_limit =).*/\1 1024M/g' -i /usr/local/zabbix/php_zabbix/etc/php.ini
```
## 1.3 配置systemctl管理php
vim /usr/lib/systemd/system/php-fpm.service
```shell
[Unit]
Description=php-fpm
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/zabbix/php_zabbix/sbin/php-fpm

[Install]
WantedBy=multi-user.target
```
## 1.4 启动php-fpm
```shell
systemctl daemon-reload

# 设置服务开机自启并启动
systemctl enable --now php-fpm

# 查看服务启动状态以及端口存活状态
systemctl status php-fpm && ss -ntulp | grep 9000
```
## 1.5 建立测试文件，测试能否正常访问php
```shell
cat << EOF >/usr/local/zabbix/nginx_zabbix/html/phptest.php
<?php
phpinfo();
?>
EOF


# 查看可否正常访问php，返回结果如下图👇
curl http://192.168.101.201/phptest.php
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1675868825938-514c0e79-397c-4900-8f84-7353e3e77797.png#averageHue=%23c9a770&clientId=uf31a22a6-dae2-4&from=paste&height=511&id=u632c3b5e&name=image.png&originHeight=511&originWidth=1135&originalType=binary&ratio=1&rotation=0&showTitle=false&size=65404&status=done&style=none&taskId=u2e4ebd67-39a0-4c71-a43b-3cfe964dd03&title=&width=1135)
# 三、安装数据库并导入数据
## 3.1 安装数据库
```shell
# 安装启动数据库
yum -y install mariadb mariadb-server

systemctl enable --now mariadb

mysqladmin -u root  password 'Root123!@#'


tar xf /root/zabbix-5.0.17.tar.gz -C /root
```
## 3.2 创建数据库和用户
```shell
mysql -uroot  -p'Root123!@#'

#初始化数据
create database vdms character set utf8 collate utf8_bin;

create user servername@localhost identified by 'serverpassword';

grant all privileges on vdms.* to servername@localhost;

use vdms;

source /root/zabbix-5.0.17/database/mysql/schema.sql;

source /root/zabbix-5.0.17/database/mysql/data.sql;

source /root/zabbix-5.0.17/database/mysql/images.sql;
```
# 四、安装zabbix server
## 5.1 安装zabbix依赖包及其服务
```shell
# 1、安装依赖
yum install -y OpenIPMI-devel libssh2-devel net-snmp-devel unixODBC-devel mysql-devel libxml2-devel libcurl-devel libevent-devel

# 2、创建zabbix用户
useradd zabbix -s /sbin/nologin

# 3、编译zabbix
cd /root/zabbix-5.0.17

./configure --prefix=/usr/local/zabbix/zabbixServer --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2 --with-openipmi --with-unixodbc

make install
```
## 5.3 配置zabbix_server.conf
```shell
# 1、配置zabbix_server
cp -a /root/zabbix-5.0.17/misc/init.d/fedora/core5/zabbix_server /etc/init.d/

sed -ri 's#ZABBIX_BIN=(.*)$#ZABBIX_BIN="/usr/local/zabbix/zabbixServer/sbin/zabbix_server"#' /etc/init.d/zabbix_server

# 2、配置zabbix_agentd
cp -a /root/zabbix-5.0.17/misc/init.d/fedora/core5/zabbix_agentd /etc/init.d/

sed -ri 's#ZABBIX_BIN=(.*)$#ZABBIX_BIN="/usr/local/zabbix/zabbixServer/sbin/zabbix_agentd"#' /etc/init.d/zabbix_agentd

# 3、修改zabbix_server.conf
sed -i '/# DBHost=/aDBHost=127.0.0.1' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

sed -i 's/^DBName=zabbix/DBName=vdms/' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

sed -i 's/^DBUser=zabbix/DBUser=servername/' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

sed -i '/# DBPassword=/aDBPassword=serverpassword' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

sed -i '/# DBPort=/aDBPort=3306' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

sed -i '/# PidFile=/aPidFile=/tmp/zabbix_server.pid' /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

# 查看配置
grep -Ev "^$|#" /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf

# 4、修改zabbix_agentd.conf
sed -ri "s/^Server=(.*)$/Server=\1,`ifconfig |grep 192.168.101 |awk '{print $2}'`/" /usr/local/zabbix/zabbixServer/etc/zabbix_agentd.conf

sed -ri "s/^ServerActive=(.*)$/ServerActive=`ifconfig |grep 192.168.101 |awk '{print $2}'`/" /usr/local/zabbix/zabbixServer/etc/zabbix_agentd.conf

sed -ri "s/^Hostname=(.*)$/Hostname=`ifconfig |grep 192.168.101 |awk '{print $2}'`/" /usr/local/zabbix/zabbixServer/etc/zabbix_agentd.conf

sed -ri "263aInclude=/opt/zabbix/etc/zabbix_agentd.conf.d/*.conf" /usr/local/zabbix/zabbixServer/etc/zabbix_agentd.conf


# 查看配置
grep -Ev "#|^$" /usr/local/zabbix/zabbixServer/etc/zabbix_agentd.conf


# 4、配置bin
ln -s /usr/local/zabbix/zabbixServer/bin/* /usr/local/bin
```
## 5.4 上线zabbix页面
```shell
cd /root/zabbix-5.0.17/ui

cp -axv ./ /usr/local/zabbix/nginx_zabbix/html/zabbix

chown -R nginx.nginx /usr/local/zabbix/nginx_zabbix/html/
```
## 5.5 启动zabbix_server、zabbix_agentd
```shell
# 启动server
/usr/local/zabbix/zabbixServer/sbin/zabbix_server -c /usr/local/zabbix/zabbixServer/etc/zabbix_server.conf
```
## 5.6 浏览器访问zabbix配置即可
例如我的是：[http://192.168.190.143/zabbix/setup.php](http://192.168.190.143/zabbix/setup.php)
![1675877047469.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1675877049901-9fd86845-92af-4f4b-a0ee-261695a04b0e.png#averageHue=%23ccf1f5&clientId=uaa9caa50-6893-4&from=paste&height=1229&id=udcf799b3&name=1675877047469.png&originHeight=1229&originWidth=1623&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39208&status=done&style=none&taskId=u087f9f8c-52a0-432c-9cfc-faa5dd8edf8&title=&width=1623)
# 五、各问题解决方式
![1675964936793.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1675964938752-2eb10a7a-3647-473f-8cd3-e8f0a48f044b.png#averageHue=%23070302&clientId=ubdff6c77-f572-4&from=paste&height=430&id=ub9870042&name=1675964936793.png&originHeight=430&originWidth=951&originalType=binary&ratio=1&rotation=0&showTitle=false&size=27002&status=done&style=none&taskId=u76bd6cdc-dab0-4f07-bb5c-503607ab9a2&title=&width=951)

