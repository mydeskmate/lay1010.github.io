---
layout: post
category: "运维"
title:  "memcache和haproxy和varnish"
tags: [马哥,Linux]
---  

1、为LNMP架构添加memcached支持，并完成对缓存效果的测试报告；  
  
>操作系统:  CentOS 7.2  
>10.0.0.51 nginx+php+mysql  
>10.0.0.52 memcached   

### 一. 环境准备:   
搭建LNMP编译安装环境
#### 1. 配置163的yum源和阿里云的epel源  
```
[root@localhost ~]# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
[root@localhost ~]# wget -O /etc/yum.repos.d/163.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
[root@localhost ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```  
#### 2. 安装Nginx  
```
#yum install nginx           #配置文件处于/etc/nginx
#systemctl start nginx     #启动nginx
#systemctl enable nginx.service   # 设置为开机启动
# rpm -q nginx
nginx-1.10.2-1.el7.x86_64
```  
测试： http://10.0.0.51是否可以打开   
  
#### 3. 安装Mysql  
```
#rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
#yum repolist enabled | grep “mysql.*-community.*”
#yum -y install mysql-community-server
# systemctl start mysqld
# systemctl enable mysqld
# mysql_secure_installation
查看版本： 
# rpm -q mysql-community-server
mysql-community-server-5.6.36-2.el7.x86_64 
```   
#### 4. 安装php  
```
#yum install php-fpm php-mysql
#systemctl start php-fpm               # 启动php-fpm
#systemctl enable php-fpm           # 设置开机启动

修改Nginx的配置文件：
]# vim /etc/nginx/conf.d/wordpress.conf
server {
        listen 8000;
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000

        location ~ \.php$ {
        root /usr/www;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        }
}
#systemctl reload nginx

在/usr/www 目录中创建 phpinfo.php
# mkdir /usr/www
# vim /usr/www/phpinfo.php
<?php
        phpinfo();
?>  

```  
测试访问： 
![mark](http://ohfysad7j.bkt.clouddn.com/blog/170710/530Cm34ABL.png?imageslim)



#### 5. 安装wordpress  
```
# wget https://cn.wordpress.org/wordpress-4.7.4-zh_CN.tar.gz
tar xf wordpress-4.7.4-zh_CN.tar.gz  -C /usr/www

# mysql -uroot -p
mysql> create database wordpress;
mysql> GRANT ALL ON wordpress.* TO wpuser@'10.%.%.%' IDENTIFIED BY 'wppass';
mysql> flush privileges;

# cp wp-config-sample.php wp-config.php
[root@localhost wordpress]# vim wp-config.php
/** WordPress数据库的名称 */
define('DB_NAME', 'wordpress');

/** MySQL数据库用户名 */
define('DB_USER', 'wpuser');

/** MySQL数据库密码 */
define('DB_PASSWORD', 'wppass');

/** MySQL主机 */
define('DB_HOST', '10.0.0.51');

```
通过页面http://10.0.0.51:8000/wordpress/wp-admin/install.php安装wordpress:
![mark](http://ohfysad7j.bkt.clouddn.com/blog/170710/FbBbEeJEl0.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/170710/CF81I7dD8B.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/170710/B4b0fgkmbe.png?imageslim) 
问题： 登录后，提示
![mark](http://ohfysad7j.bkt.clouddn.com/blog/170710/76KECBjb03.png?imageslim)    
需手动加上index.php才能打开， 这个问题有时间再研究。

### 二. 安装memcached  
```
[root@localhost ~]# yum -y install memcached
[root@localhost ~]# systemctl start memcached.service
```
### 三. 查看memcached状态  
```
[root@localhost ~]# telnet 10.0.0.52 11211
stats             #stats命令用来查看memcahced状态
STAT pid 15706    #memcache服务器的进程ID  
STAT uptime 130   #服务器已经运行的秒数
STAT time 1499479581           #服务器当前的unix时间戳
STAT version 1.4.15            #memcache版本
STAT libevent 2.0.21-stable    #libevent版本
STAT pointer_size 64           #当前操作系统的指针大小（32位系统一般是32bit,64就是64位操作系统）
STAT rusage_user 0.002243      #进程的累计用户时间
STAT rusage_system 0.013459    #进程的累计系统时间
STAT curr_connections 10       #服务器当前存储的items数量
STAT total_connections 11      #从服务器启动以后存储的items总数量
STAT connection_structures 11  #服务器分配的连接构造数
STAT reserved_fds 20
STAT cmd_get 0       #get命令（获取）总请求次数,等于 get_hits + get_misses 
STAT cmd_set 0       #set命令（保存）总请求次数 
STAT cmd_flush 0     #flush命令请求次数
STAT cmd_touch 0     #touch命令请求次数
STAT get_hits 0      #总命中次数
STAT get_misses 0    #总未命中次数
STAT delete_misses 0    #delete命令未命中次数
STAT delete_hits 0      #delete命令命中次数
STAT incr_misses 0      #incr命令未命中次数
STAT incr_hits 0        #incr命令命中次数
STAT decr_misses 0      #decr命令未命中次数
STAT decr_hits 0        #decr命令命中次数
STAT cas_misses 0       #cas命令未命中次数
STAT cas_hits 0         #cas命令命中次数
STAT cas_badval 0       #使用擦拭次数
STAT touch_hits 0       #touch命令未命中次数
STAT touch_misses 0      #touch命令命中次数
STAT auth_cmds 0        #认证命令处理的次数
STAT auth_errors 0      #认证失败数目
STAT bytes_read 7       #总读取字节数（请求字节数）
STAT bytes_written 0    #总发送字节数（结果字节数）
STAT limit_maxbytes 67108864    #分配给memcache的内存大小（字节）
STAT accepting_conns 1          #服务器是否达到过最大连接（0/1）
STAT listen_disabled_num 0      #失效的监听数
STAT threads 4                  #当前线程数
STAT conn_yields 0              #连接操作主动放弃数目
STAT hash_power_level 16
STAT hash_bytes 524288
STAT hash_is_expanding 0
STAT bytes 0                   #当前存储占用的字节数
STAT curr_items 0              #当前存储的数据总数
STAT total_items 0             #启动以来存储的数据总数
STAT expired_unfetched 0
STAT evicted_unfetched 0
STAT evictions 0   #为获取空闲内存而删除的items数（分配给memcache的空间用满后需要删除旧的items来得到空间分配给新的items)
STAT reclaimed 0   #已过期的数据条目来存储新数据的数目
END
```    
### 三．安装PHP的Memcached的扩展　　
>php连接memcached服务的模块有两个，php-pecl-memcache和php-pecl-memc>ached.若要安装php-pecl-memcached需要依赖libmemcached程序包，可以提>供相应操作查看memcached的工具。在这里为方便演示就直接使用php-pecl->memcache扩展模块。　　
```
[root@localhost ~]# yum -y install php-pecl-memcache
[root@localhost ~]# php -m|grep memcache
memcache
```  

打开测试页面查看是否已支持memcache
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170711/011046358.png?imageslim)

### 四. 测试memcached缓存  
```
[root@localhost ~]# cd /usr/www
[root@localhost www]# vim index.php
<?php
$memcache = new Memcache;             #创建一个memcache对象
$memcache->connect('10.0.0.52',11211) or die ("Could not connect"); #连接Memcached服务器
$memcache->set('key','test');        #设置一个变量到内存中，名称是key 值是test
$get_value = $memcache->get('key');   #从内存中取出key的值
echo $get_value;
?>
```
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170711/011639983.png?imageslim)

问题: 出现无法缓存的问题, 重启php所在服务器后结局  

### 五. 为wordpress配置memcache  
下载 WordPress Memcached插件(http://wordpress.org/plugins/memcached/)，解压后，将 object-cache.php 上传到 wp-content 目录（不是 wp-content/plugins/），这样 WordPress 会自动检查在 wp-content 目录下是否有 object-cache.php 文件，如果有，直接调用它作为 WordPress 对象缓存机制。
```
[root@localhost ~]# cp object-cache.php /usr/www/wordpress/wp-content/
[root@localhost ~]# vim object-cache.php +418
修改地址:
$buckets = array('10.0.0.52:11211');
```
登录wordpress进行一些操作看, 查看缓存情况
```
[root@localhost wp-content]# telnet 10.0.0.52 11211
Trying 10.0.0.52...
Connected to 10.0.0.52.
Escape character is '^]'.
stats
STAT pid 11262
STAT uptime 99763
STAT time 1499708968
STAT version 1.4.15
STAT libevent 2.0.21-stable
STAT pointer_size 64
STAT rusage_user 2.200638
STAT rusage_system 1.379317
STAT curr_connections 22
STAT total_connections 58
STAT connection_structures 23
STAT reserved_fds 20
STAT cmd_get 712       #总共获取数据的次数（等于 get_hits + get_misses ）
STAT cmd_set 117       #总共设置数据的次数
STAT cmd_flush 0
STAT cmd_touch 0
STAT get_hits 614  #命中了多少次数据，也就是从 Memcached 缓存中成功获取数据的次数
STAT get_misses 98 #没有命中的次数
STAT delete_misses 0
STAT delete_hits 20
STAT incr_misses 0
STAT incr_hits 2
STAT decr_misses 0
STAT decr_hits 0
STAT cas_misses 0
STAT cas_hits 0
STAT cas_badval 0
STAT touch_hits 0
STAT touch_misses 0
STAT auth_cmds 0
STAT auth_errors 0
STAT bytes_read 159540
STAT bytes_written 483317
STAT limit_maxbytes 67108864  #总的存储大小，默认为 64M
STAT accepting_conns 1
STAT listen_disabled_num 0
STAT threads 4
STAT conn_yields 0
STAT hash_power_level 16
STAT hash_bytes 524288
STAT hash_is_expanding 0
STAT bytes 23219       #当前所用存储大小
STAT curr_items 43
STAT total_items 116
STAT expired_unfetched 0
STAT evicted_unfetched 0
STAT evictions 0
STAT reclaimed 1
END

数据命中率: 614/712=86.2% 
```
2、部署配置haproxy，能够实现将来自用户的80端口的http请求转发至后端8000上的server服务，写出其配置过程。   
环境准备：  
增加一台haproxy服务,CentOS 7.2系统
10.0.0.50   haproxy    
10.0.0.51 nginx+php+mysql  (借用第一题的环境)

### 一. 安装haproxy  
```
]# yum -y install haproxy
```
### 二. 配置haporxy  
```
]# vim /etc/haproxy/haproxy.cfg
...
listen websrvs        #定义一个代理服务器websrvs
    bind        *:80  #指定代理监听端口为80
    server      websrv 10.0.0.51:8000 check    #定义一个后端websrv，注意：后端服务端口如果与监听端口不一致，需要在地址后指明端口号
...
检测配置文件
]# haproxy -f /etc/haproxy/haproxy.cfg -c
Configuration file is valid

```
### 三. 启动服务 
```
]# systemctl start haproxy
]# systemctl enable haproxy
```
### 四. 访问测试  
直接访问后端服务器  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170711/221404799.png?imageslim)   
通过haproxy访问  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170711/221448878.png?imageslim)  

3、阐述varnish的功能及其应用场景，并通过实际的应用案例来描述配置、测试、调试过程。

>系统: CentOS 7.2   
>varnish: 10.0.0.50 
>nginx+php-fpm+mariadb : 10.0.0.51   
>nginx : 10.0.0.52    

### 一. varnish介绍  
Web缓存是指一个Web资源(html,js,css,images…）存在于Web服务器和客户端(浏览器）,缓存会根据进来的请求报文做出响应，后缓存一份到本地的缓存中；当下一个请求到来的时候，如果是相同的URL，缓存会根据缓存机制决定是直接使用从缓存中响应访问请求还是向后端服务器再次发送请求，取决于缓存是否过期及其请求的内容是否发生改变。有效的缓存能减少后端主机的压力，实现快速响应用户的请求，提高用户体验。

varnish就是一种实现上述web缓存功能（通常针对于静态资源提供页面缓存）的一款开源工具，通常它也被称为http或web加速器，同时它也可以做为http反向代理工具，实现负载均衡和动静分离的功能。此外，据官网介绍，Varnish的设计不仅仅是定位于反向代理服务器，根据使用方式的不同，Varnish可扮演的角色也丰富多样，其它可实现的功能如下：

1）WEB应用防火墙；
2）DDoS攻击防护；
3）网站防盗链；
4）负载均衡；
5）integration point；
6）单点登录网关；
7）认证和认证授权；
8）后端主机快速修复；
9）HTTP路由

但在实际生产环境中varnish更多是在HAProxy、Nginx等七层负载均衡器后充当静态资源缓存的反 向代理,本例是一个简化的varnish应用场景，主要用来实现静态资源缓存和动静分离。   
  
### 二. 部署web服务  
```
####在10.0.0.51部署动态web服务  
]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
~]# yum install nginx php-fpm php-mysql php-mbstring php-gd php-xml -y
~]# mkdir -p /data/www
~]# vim /etc/nginx/nginx.conf
location / {
                root /data/www;
                index   index.php index.html index.htm;
        }

        location ~ \.php$ {
            root           /data/www;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

[root@localhost ~]# systemctl start nginx.service
[root@localhost ~]# systemctl enable  nginx.service
[root@localhost ~]# systemctl start php-fpm
[root@localhost ~]# systemctl enable php-fpm  

部署mariadb
~]# yum install mariadb-server -y
~]# vim /etc/my.cnf
[mysqld]
...
innodb_file_per_table = ON
skip_name_resolve = ON

~]# systemctl start mariadb.service
~]# mysql
> grant all on *.* to root@'10.%.%.%' identified by 'magedu';
> flush privileges;

####在10.0.0.52上配置静态服务
]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
~]# yum install nginx -y
~]# mkdir -p /data/www
~]# vim /etc/nginx/nginx.conf
location / {
                root    /data/www;
                index   index.html index.htm;
        }
~]# systemctl start nginx.service
~]# systemctl enable nginx

```
### 三. 部署varnish  
```
~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
~]# yum install varnish -y
######将varnish的监听端口设置为80######
~]# vim /etc/varnish/varnish.params
VARNISH_LISTEN_PORT=80            
######配置varnish参数文件######
~]# vim /etc/varnish/default.vcl
######设置默认的后端静态服务器ip和端口######
backend default {
    .host = "10.0.0.51";
    .port = "80";
######配置健康状态监测######
    .probe = {
        .url = "/";
        .interval = 2s;
        .window = 5;
        .threshold = 4;
    }

}
######配置后端动态web服务器######
backend appsrv {
    .host = "10.0.0.52";
    .port = "80";
######配置健康状态监测######
    .probe = {
        .url = "/";
        .interval = 2s;
        .window = 5;
        .threshold = 4;
    }

}
######定义Purge-ACL控制######
acl purgers {
    "127.0.0.1";
    "10.0.0.0"/24;
}
######定义purge操作######
sub vcl_purge {
    return(synth(200,"Purged"));
}

sub vcl_recv {
######动静分离######
        if (req.url ~ "(?i)\.php$") {
             set req.backend_hint = appsrv;
         } else {
             set req.backend_hint = default;
         }
######如果请求方法为PURGE，且客户端IP满足acl，则执行purge操作，否则返回405页面并提示######
        if (req.method == "PURGE") {                
             if (!client.ip ~ purgers) {
                 return(synth(405,"Purging not allowed for" + client.ip));
                 return(purge);
             }
         }
}
######记录缓存命中状态######
sub vcl_deliver {
        if (obj.hits>0) {
              set resp.http.X-Cache="HIT";
    } else {
              set resp.http.X-Cache="MISS";
    }
}
######启动服务使配置生效######
~]# systemctl start varnish.service
~]# systemctl enable varnish.service
```
### 四.创建检测页  
```
######在10.0.0.51上创建动态探测页######
~]# vim /data/www/index.php <?php
        $conn = mysql_connect('10.0.0.51','root','magedu');
        if ($conn)
                echo "Dynamic webserver to Mariadb is OK!";
        else
                echo "Failure";
?>
######在10.0.0.52上创建静态探测页######
~]# vim /data/www/index.html
<h1>I'm Static Server!</h1>
```
### 五.缓存效果测试  
```
######在varnish CLI命令接口下创建并启用vcl######
~]# varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082
######加载默认vcl配置文件，并命名为test1######
varnish> vcl.load test1 default.vcl 
200        
VCL compiled.
######激活test1######
varnish> vcl.use test1
200        
VCL 'test1' now active
######通过健康状态监测可以看到后端服务器都正常######
varnish> backend.list
200
Backend name                   Refs   Admin      Probe
default(10.0.0.52,,80)         2      probe      Sick 0/5
appsrv(10.0.0.51,,80)          2      probe      Sick 0/5

######第一次访问静态页面，MISS######
~]# curl -I 10.0.0.50/index.html
HTTP/1.1 200 OK
Server: nginx/1.10.2
Date: Wed, 05 Jul 2017 14:38:28 GMT
Content-Type: text/html
Content-Length: 28
Last-Modified: Wed, 05 Jul 2017 14:21:54 GMT
ETag: "595cf602-1c"
X-Varnish: 8
Age: 0
Via: 1.1 varnish-v4
X-Cache: MISS
Connection: keep-alive

######第二次访问静态页面，HIT!######
~]# curl -I 10.0.0.50/index.html
HTTP/1.1 200 OK
Server: nginx/1.10.2
Date: Wed, 05 Jul 2017 14:38:28 GMT
Content-Type: text/html
Content-Length: 28
Last-Modified: Wed, 05 Jul 2017 14:21:54 GMT
ETag: "595cf602-1c"
X-Varnish: 32770 9
Age: 2
Via: 1.1 varnish-v4
X-Cache: HIT
Connection: keep-alive

######第一次访问动态页面，MISS######
~]# curl -I 10.0.0.50/index.php
HTTP/1.1 200 OK
Server: nginx/1.10.2
Date: Wed, 05 Jul 2017 14:53:11 GMT
Content-Type: text/html
X-Powered-By: PHP/5.4.16
X-Varnish: 11
Age: 0
Via: 1.1 varnish-v4
X-Cache: MISS
Content-Length: 35
Connection: keep-alive

######第二次访问动态页面,HIT!######
~]# curl -I 10.0.0.50/index.php
HTTP/1.1 200 OK
Server: nginx/1.10.2
Date: Wed, 05 Jul 2017 14:53:11 GMT
Content-Type: text/html
X-Powered-By: PHP/5.4.16
X-Varnish: 32772 12
Age: 23
Via: 1.1 varnish-v4
X-Cache: HIT
Content-Length: 35
Connection: keep-alive

######分别访问动静资源均正常，说明动静分离实现成功######
[root@localhost ~]# curl  10.0.0.50/index.html
<h1>I'm Static Server!</h1>
[root@localhost ~]# curl  10.0.0.50/index.php
Dynamic webserver to Mariadb is OK!


```


