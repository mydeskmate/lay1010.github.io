---
layout: post
category: "运维"
title:  "heartbeartv2实现lamp高可用"
tags: [马哥,Linux,集群]
---  

### 3、基于heartbeat v2 crm实现HA LAMP组合；要求，部署wordpress，用于编辑的文章中的任何数据在节点切换后都能正常访问；

拓扑： 
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/013847906.png?imageslim)
>环境： CentOS6.6
NFS: 172.16.0.34   输出mysql数据目录
ntp: 172.16.0.31    时间服务器
node1: 172.16.0.32   heartbeart+httpd+php+mysql
node2： 172.16.0.33  heartbeart+httpd+php+mysql
fip： 172.16.0.40   浮动ip


### 一. 安装ntp时间服务器
```
1. 安装ntp服务器
#查看是否已经安装
[root@localhost ~]# rpm -qa ntp
#如果没有就安装
[root@localhost ~]# yum -y install ntp

2. 配置ntp服务
[root@localhost ~]# vim /etc/ntp.conf
#设置权限
#restrict default kod nomodify notrap nopeer noquery  默认拒绝所有来源的任何访问
restrict 127.0.0.1    #给于本机所有权限 
restrict 172.16.0.0 mask 255.255.0.0 notrap nomodify   #给于局域网机的机器有同步时间的权限
#设置上层ntp服务器,注释掉原有上层服务器
server ntp1.aliyun.com prefer   #设置上传时间服务器，假prefer表示优先
server time.nist.gov
#内部时钟,用在没有外网ntp服务器时
server  127.127.1.0     # local clock
fudge   127.127.1.0 stratum 10    #时间服务器的层次

[root@localhost ~]# cat /etc/ntp.conf |awk '{if($0 !~ /^$/ && $0 !~ /^#/) {print $0}}'
driftfile /var/lib/ntp/drift
restrict 127.0.0.1
restrict 172.16.0.0 mask 255.255.0.0 notrap nomodify
server ntp1.aliyun.com prefer
server time.nist.gov
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys

3. 启动ntp
[root@localhost ~]# service ntpd start
[root@localhost ~]# chkconfig  ntpd on

4. 查看并测试
查看ntp端口
[root@localhost ~]# netstat -ulnp | grep ntpd
udp        0      0 172.16.0.31:123             0.0.0.0:*                               1485/ntpd
udp        0      0 127.0.0.1:123               0.0.0.0:*                               1485/ntpd
udp        0      0 0.0.0.0:123                 0.0.0.0:*                               1485/ntpd
udp        0      0 fe80::20c:29ff:fe90:d168:123 :::*                                    1485/ntpd
udp        0      0 ::1:123                     :::*                                    1485/ntpd
udp        0      0 :::123                      :::*                                    1485/ntpd

查看ntp服务器有无和上层连通
[root@localhost ~]# ntpstat
synchronised to NTP server (182.92.12.11) at stratum 3
   time correct to within 11 ms
   polling server every 64 s
   
查看ntp服务器与上层ntp服务器的状态
[root@localhost ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*time5.aliyun.co 10.137.38.86     2 u    5   64  377    3.272    1.288   0.844
xnist1-lnk.binar .ACTS.           1 u  139   64  144  318.446  -44.920   5.038

其中，
remote - 本机和上层ntp的ip或主机名，“+”表示优先，“*”表示次优先
refid - 参考上一层ntp主机地址
st - stratum阶层
when - 多少秒前曾经同步过时间
poll - 下次更新在多少秒后
reach - 已经向上层ntp服务器要求更新的次数
delay - 网络延迟
offset - 时间补偿
jitter - 系统时间与bios时间差
```

### 二.heartbeart环境准备
(1) 设置主机名
```
[root@node1 ~]# cat /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node1.magedu.com

[root@node2 ~]# cat /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node2.magedu.com
```
(2) 分别配置hosts文件
```
~]# vim /etc/hosts
172.16.0.32 node1.magedu.com node1
172.16.0.33 node2.magedu.com node2
```
(3) ssh密钥认证
```
[root@node1 ~]# ssh-keygen -t rsa
[root@node1 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@node2

[root@node2 ~]# ssh-keygen -t rsa
[root@node2 ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@node1
```
(4) 同步时间
```
 ~]# yum -y install ntp
 ~]# vim /etc/ntp.conf
 server 172.16.0.31
 ~]# service  ntpd start
 ~]# chkconfig ntpd on
 ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 172.16.0.31     182.92.12.11     3 u   46   64    7    0.219  112.007   0.418
 ~]# ntpstat
synchronised to NTP server (172.16.0.31) at stratum 4
   time correct to within 1072 ms
   polling server every 64 s
   验证: 
   [root@node1 ~]# date; ssh node2 'date'
Sun May 21 15:57:15 CST 2017
Sun May 21 15:57:15 CST 2017
```
### 三.nfs服务器
```
#创建目录并授权
[root@localhost data]# mkdir /mydata
[root@localhost data]# groupadd -r -g 306 mysql
[root@localhost data]# useradd -r -g 306 -u 306 mysql
[root@localhost data]# mkdir /mydata/data
[root@localhost data]# chown -R mysql.mysql /mydata/data
[root@localhost data]# ll /mydata/
total 4
drwxr-xr-x. 2 mysql mysql 4096 May 18 18:23 data
#配置共享
[root@localhost data]# vim /etc/exports
/mydata     172.16.0.0/16(rw,no_root_squash)
[root@localhost data]# exportfs -arv
#启动nfs
[root@localhost ~]# service rpcbind start
[root@localhost ~]# service nfs start
[root@localhost ~]# showmount -e 172.16.0.34
Export list for 172.16.0.34:
/mydata 172.16.0.0/16
```
### 四.安装mysql
在node1和node2分别执行
```
]# yum -y install nfs-utils
]# mkdir /mydata
]# groupadd -r -g 306 mysql
]# useradd -r -g 306 -u 306 mysql
]# mount -t nfs 172.16.0.34:/mydata /mydata

]# tar xf mariadb-5.5.52-linux-x86_64.tar.gz -C /usr/local/
]# cd /usr/local/
]# ln -sv mariadb-5.5.52-linux-x86_64 mysql
]# cd mysql/
]# chown -R root.mysql ./*
#初始化数据库(只执行一次)
]# ./scripts/mysql_install_db  --datadir=/mydata/data/ -user=mysql

配置文件
mysql]# mkdir /etc/mysql
mysql]# cp support-files/my-large.cnf /etc/mysql/my.cnf
mysql]# vim /etc/mysql/my.cnf
[mysqld]
...
datadir =/mydata/data
innodb_file_per_table = on
skip_name_resolve = on

准备启动脚本,并启动
mysql]# cp support-files/mysql.server /etc/rc.d/init.d/mysqld
mysql]# chkconfig --add mysqld
mysql]# service mysqld start

创建数据库(只执行一次)
[root@localhost mysql]# /usr/local/mysql/bin/mysql
MariaDB [(none)]> create database wpdb;
MariaDB [(none)]> grant all on wpdb.* to 'wpuser'@'172.16.%' identified by '123456';
MariaDB [(none)]> flush privileges;
MariaDB [(none)]> exit

关闭mysqld服务,并设置开机不启动
]# service mysqld stop
]# chkconfig mysqld off

卸载nfs共享
]# umount /mydata
```
###　五. httpd安装
在node1和node2分别执行
```
]# yum -y install httpd
]# echo "<h1>node1</h1>" > /var/www/html/index.html    #node1执行
]# echo "<h1>node2</h1>" > /var/www/html/index.html

启动
]# service httpd start;ssh node2 'service httpd start'  #node1执行

访问测试
]# curl node{1,2}
<h1>node1</h1>
<h1>node2</h1>

停止服务及开机启动
]# service httpd stop;ssh node2 'service httpd stop'   #node1执行
]# chkconfig httpd off; ssh node2 'chkconfig httpd off'

查看是否关闭自启动
~]# chkconfig --list httpd; ssh node2 'chkconfig --list httpd'
httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
httpd           0:off   1:off   2:off   3:off   4:off   5:off   6:off
```

### 六.安装php
在node1和node2分别执行
```
]# yum -y install php php-mysql
```
### 七.安装wordpress
在node1和node2分别执行
```
~]# cd /usr/local/src
src]# wget https://cn.wordpress.org/wordpress-4.7.4-zh_CN.zip
src]# unzip wordpress-4.7.4-zh_CN.zip
src]# cp -r -p  wordpress/* /var/www/html
src]# cd /var/www/html/
wordpress]# cp wp-config-sample.php wp-config.php         
wordpress]# vim wp-config.php                             
define('DB_NAME', 'wpdb');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', '123456');
define('DB_HOST', '172.16.0.40');     

注意: 页面安装时必须使用fip,因为ip地址会写入数据库中,造成页面打开失败(访问另一个不可用的ip地址)

测试命令:
手动挂载数据目录并启动mysql                         
]# mount -t nfs 172.16.0.34:/mydata /mydata
]# service mysqld start
]# service httpd start
手动关闭httpd,msyqld服务并取消挂载
[root@node1 wordpress]# service httpd stop
[root@node1 wordpress]# service mysqld stop
[root@node1 wordpress]# umount /mydata/
```

### 八. 安装heartbeat
在node1和node2分别执行
(1) 安装依赖包
```
~]# yum -y install  net-snmp-libs libnet PyXML  gettext-devel  libtool-ltdl pygtk2-libglade
~]# rpm -ivh libnet-1.1.6-7.el6.x86_64.rpm
```
(2) 安装heartbeart及相关包
```
~]# rpm -ivh heartbeat-2.1.4-12.el6.x86_64.rpm heartbeat-pils-2.1.4-12.el6.x86_64.rpm heartbeat-stonith-2.1.4-12.el6.x86_64.rpm heartbeat-gui-2.1.4-12.el6.x86_64.rpm
```
(3) 准备heartbeat配置文件
```
~]# cp /usr/share/doc/heartbeat-2.1.4/{ha.cf,haresources,authkeys} /etc/ha.d/
~]# cd /etc/ha.d/
ha.d]# chmod 600 authkeys
```
(4) 配置authkeys
```
ha.d]# openssl rand -base64 6
7mA2c+82
ha.d]# vim authkeys
auth 2
#1 crc
2 sha1 7mA2c+82
```
(5) 配置ha.cf
```
ha.d]# vim ha.cf
#logfacility    local0
logfile /var/log/heartbeat.log
mcast eth0 225.23.190.1 694 1 0
auto_failback on
node node1.magedu.com
node node2.magedu.com
ping 172.16.0.1
crm on

[root@node1 ha.d]# grep -v "#" ha.cf
logfile /var/log/heartbeat.log
mcast eth0 225.23.190.1 694 1 0
auto_failback on
node node1.magedu.com
node node2.magedu.com
ping 172.16.0.1
crm on  
```
(6) 配置文件拷贝到node2
```
[root@node1 ha.d]# scp -p  authkeys ha.cf root@node2:/etc/ha.d
```
(7) 启动服务
```
[root@node1 ~]# service heartbeat start; ssh node2 'service heartbeat start'
```
(8) 查看集群状态
```
~]# crm_mon
Refresh in 14s...

============
Last updated: Sun May 21 18:26:53 2017
Current DC: node2.magedu.com (2fe0005c-a582-4e46-9e22-353eed7d91cc)
2 Nodes configured.
0 Resources configured.
============

Node: node2.magedu.com (2fe0005c-a582-4e46-9e22-353eed7d91cc): online
Node: node1.magedu.com (81ae3d5f-02ca-4c97-81cc-6d8b933f716c): online
```
(9) 设置hacluster密码
哪个节点使用hb_gui,就在哪个节点创建
```
[root@node1 ~]# echo "mageedu" | passwd --stdin hacluster
[root@node2 ~]# echo "mageedu" | passwd --stdin hacluster
```
(10)hb_gui设置
问题: 解决hb_hui无法正常显示问题
```
[root@node1 ha.d]# hb_gui &
[root@node1 ha.d]# Traceback (most recent call last):
  File "/usr/bin/hb_gui", line 41, in <module>
    import gtk, gtk.glade, gobject
  File "/usr/lib64/python2.6/site-packages/gtk-2.0/gtk/__init__.py", line 64, in <module>
    _init()
  File "/usr/lib64/python2.6/site-packages/gtk-2.0/gtk/__init__.py", line 52, in _init
    _gtk.init_check()
RuntimeError: could not open display

#安装支持xshell xmanager包，否则xshell无法调用x11,安装后重启系统
[root@node1 ha.d]# yum -y install xorg-x11-xauth
#重启后可以弹出窗口,但无法显示文字,
[root@node1 ~]# hb_gui &
[1] 1276
[root@node1 ~]# /usr/bin/hb_gui:2856: PangoWarning: failed to choose a font, expect ugly output. engine-type='PangoRenderFc', script='latin'
  win_widget.show_all()
/usr/bin/hb_gui:2856: PangoWarning: failed to choose a font, expect ugly output. engine-type='PangoRenderFc', script='common'
  win_widget.show_all()
#安装字体
[root@node1 ~]#  yum install dejavu-lgc-sans-fonts
```
配置
```
[root@node1 ~]# hb_gui &
```
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/184141668.png?imageslim)
登录  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/184418522.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/184533074.png?imageslim)
解释：
Node Name ： 节点名字
Online： 是否在线
Is it DC：是不是DC
Type： 是不是集群成员
StandBy: 是不是在备用模式
Expected up: 是否启动为true
Shutdown： 是否shutdown
unclean： 是不是非干净（一致性）状态

添加资源ip,nfs,mysqld,httpd
要求: 
1. mysqld要在nfs启动之后出能启动
2. ip地址和mysqld服务之间没有先后关系
3. httpd服务需要先启动ip,因为服务启动的时候明确需要ip地址的资源
注意: 分组group已经包含了顺序约束和排列约束,要按照顺序创建资源
1) 定义一个组并添加ip
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/212521790.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/212644019.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/212756960.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/212846829.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/212938357.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/213039643.png?imageslim)

2) 添加nfs文件系统
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/213431312.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/213902735.png?imageslim)
3) 添加mysql资源
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/214047501.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/214231236.png?imageslim)
4) 添加web资源
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/214822661.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/215057051.png?imageslim)
5) 启动组
保存配置
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/220709632.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/220749041.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170522/220919834.png?imageslim)
默认在DC上运行,所以所有资源运行在node2上

(11) 查看节点上运行的资源
```
服务:
[root@node2 ~]# ss -tnl | egrep "3306|80"
LISTEN     0      50                        *:3306                     *:*
LISTEN     0      128                      :::80                      :::*
进程:
[root@node2 ~]# ps aux | egrep "mysqld_safe|httpd"
root      2811  0.0  0.3  11432  1512 ?        S    22:08   0:00 /bin/sh /usr/local/mysql/bin/mysqld_safe --datadir=/mydata/data --pid-file=/mydata/data/node2.magedu.com.pid
root      3246  0.0  1.8 252404  9104 ?        Ss   22:08   0:00 /usr/sbin/httpd
apache    3248  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3249  0.0  1.0 252404  5252 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3250  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3251  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3252  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3253  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3254  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
apache    3255  0.0  1.0 252404  5228 ?        S    22:08   0:00 /usr/sbin/httpd
ip地址: 
[root@node2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 00:0c:29:b6:b3:39 brd ff:ff:ff:ff:ff:ff
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:b6:b3:43 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.33/24 brd 172.16.0.255 scope global eth0
    inet 172.16.0.40/16 brd 172.16.255.255 scope global eth0
    inet6 fe80::20c:29ff:feb6:b343/64 scope link
       valid_lft forever preferred_lft forever
nfs挂载: 
[root@node2 ~]# mount | grep mydata
172.16.0.34:/mydata on /mydata type nfs (rw,vers=4,addr=172.16.0.34,clientaddr=172.16.0.33)
```
(12) 访问测试
当前在node2上

![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005114461.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005301950.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005343851.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005421552.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005606891.png?imageslim)
(13) 模拟节点故障,node2切换为备节点standby
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/005807664.png?imageslim)
页面访问写入均正常
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/010022708.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/010132664.png?imageslim)
(14) node2切换为active, node2抢回所有资源
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/010530534.png?imageslim)

**问题: 用fip打开页面时,报连接不到数据库或者是页面上的连接跳转到另一个没有开启服务的ip地址**
解决方法: 将wp-config.php中的ip设置为fip
原因是ip地址会写入mysql数据库
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170523/011214180.png?imageslim)






