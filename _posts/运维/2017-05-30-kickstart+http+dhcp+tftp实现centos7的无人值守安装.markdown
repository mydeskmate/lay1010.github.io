---
layout: post
category: "other"
title:  "kickstart+http+dhcp+tftp实现centos7的无人值守安装"
tags: [马哥,Linux,kickstart,自动化]
---  

>环境：
>IP:  172.16.0.11 centos 7.2

### 一. tftp
```
安装tftp服务端和客户端
[root@localhost ~]# yum -y install tftp tftp-server
启动tftp
[root@localhost ~]# systemctl start tftp.socket
[root@localhost ~]# systemctl enable tftp.socket

[root@localhost ~]# ss -unl | grep :69
UNCONN     0      0           :::69                      :::*

在tftp工作目录下准备一个文件，进行测试
[root@localhost ~]# ls /var/lib/tftpboot/
[root@localhost ~]# cp /etc/grub2.cfg /var/lib/tftpboot/
先切换到本地/tmp目录， 因为tftp不支持lcd命令
[root@localhost ~]# cd /tmp
下载grub2.cfg文件
[root@localhost tmp]# tftp 172.16.0.11
tftp> get grub2.cfg
tftp> quit

查看本地目录： 
[root@localhost tmp]# ls
fstab  grub2.cfg  nginx.conf

删除测试文件
[root@localhost tmp]# rm /var/lib/tftpboot/grub2.cfg

```

### 二. dhcp
```
[root@localhost ~]# yum -y install dhcp
[root@localhost ~]# cd /etc/dhcp/
[root@localhost dhcp]# \cp /usr/share/doc/dhcp*/dhcpd.conf.example dhcpd.conf
[root@localhost dhcp]# vim dhcpd.conf
注释掉host配置项
#host hostname {       #名称标识  和客户端hostname可以不同
# hardware ethernet 00:0c:29:f9:98:20;   #客户端mac
# fixed-address  172.16.100.10;          #给上面的mac固定ip地址
#}


option domain-name "magedu.com";    #搜索后缀
option routers 172.16.0.254;        #网关, 配置成实际的网关
option domain-name-servers 202.106.0.20;   #dns地址



subnet 172.16.0.0 netmask 255.255.0.0 {            #子网
    range 172.16.100.101 172.16.100.130;           #地址池
    filename "pxelinux.0";
    next-server 172.16.0.11;
}


[root@localhost dhcp]# systemctl restart dhcpd.service
[root@localhost dhcp]# ss -unl | grep :67
UNCONN     0      0            *:67                       *:*

```

### 三. yum 仓库
```
使用光盘做yum仓库, 最好拷贝到某个目录,而不是直接挂载光盘, 此处直接挂载
[root@localhost dhcp]# mkdir -pv /var/www/html/centos/7/x86_64
[root@localhost dhcp]# mount -r /dev/cdrom /var/www/html/centos/7/x86_64
[root@localhost dhcp]# ls /var/www/html/centos/7/x86_64/
CentOS_BuildTag  EULA  images    LiveOS    repodata              RPM-GPG-KEY-CentOS-Testing-7
EFI              GPL   isolinux  Packages  RPM-GPG-KEY-CentOS-7  TRANS.TBL

[root@localhost dhcp]# systemctl start httpd.service

```
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170531/234438881.png?imageslim)


### 四. kickstart
制作kickstart文件
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011157521.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011222774.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011330745.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011709789.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011754720.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/011920667.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012012282.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012207170.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012321815.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012432226.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012531460.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/012652208.png?imageslim)

```
[root@localhost ~]# ksvalidator centos7.cfg

[root@localhost dhcp]# mkdir /var/www/html/kickstarts
[root@localhost dhcp]# vim /var/www/html/kickstarts/centos7.cfg
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
# old format: keyboard us
# new format:
keyboard --vckeymap=us --xlayouts='us'
# Root password
rootpw --iscrypted $1$hgfvQffN$tXNj5mQldgQt4ziW1QhNF0
# Use network installation
url --url="http://172.16.0.11/centos/7/x86_64"
# System language
lang en_US
# Firewall configuration
firewall --disabled
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
# Run the Setup Agent on first boot
firstboot --enable
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx

# System services
services --disabled="chronyd"
ignoredisk --only-use=sda
# Network information
network  --bootproto=dhcp --device=eno16777984
# Reboot after installation
reboot
# System timezone
timezone Asia/Shanghai --ntpservers=3.centos.pool.ntp.org,0.centos.pool.ntp.org,2.centos.pool.ntp.org,1.centos.pool.ntp.org
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --asprimary --fstype="xfs" --size=1000
part swap --fstype="swap" --size=8000
part / --fstype="xfs" --grow --size=1

%packages
@^minimal
@core

%end

```

五. pxelinux
```
配置yum仓库, 使用公网yum仓库亦可
[root@localhost ~]# vim /etc/yum.repos.d/CentOS-Base.repo
[base]
name=CentOS-$releasever - Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=file:///var/www/html/centos/7/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[root@localhost Packages]# yum -y install syslinux

[root@localhost Packages]# rpm -ql syslinux | grep "pxelinux.0"
/usr/share/syslinux/gpxelinux.0
/usr/share/syslinux/pxelinux.0
拷贝pxelinux到tftp工作目录
[root@localhost Packages]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
拷贝内核和initrd文件
[root@localhost ~]# cp /var/www/html/centos/7/x86_64/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/

准备字符界面需要的文件
[root@localhost ~]# cp /usr/share/syslinux/{chain.c32,menu.c32,memdisk,mboot.c32} /var/lib/tftpboot/
准备引导界面目录
[root@localhost tftpboot]# cd
[root@localhost ~]# cd /var/lib/tftpboot/
[root@localhost tftpboot]# ls
chain.c32  initrd.img  mboot.c32  memdisk  menu.c32  pxelinux.0  vmlinuz
[root@localhost tftpboot]# mkdir pxelinux.cfg
[root@localhost tftpboot]# cd pxelinux.cfg/
[root@localhost pxelinux.cfg]# vim default
default menu.c32
        prompt 5
        timeout 30
        MENU TITLE CentOS 7 PXE Menu

        LABEL linux
        MENU LABEL Install CentOS 7 x86_64
        KERNEL vmlinuz
        APPEND initrd=initrd.img inst.repo=http://172.16.0.11/centos/7/x86_64
        LABEL linux_autoinst
        MENU LABEL Install CentOS 7 x86_64 auto
        KERNEL vmlinuz
        APPEND initrd=initrd.img inst.repo=http://172.16.0.11/centos/7/x86_64 ks=http://172.16.0
.11/kickstarts/centos7.cfg



启动httpd服务(之前启动过,此处应不需要重启)
[root@localhost pxelinux.cfg]# systemctl start httpd.service
```


### 六. 验证
创建虚拟机并启动
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/030414388.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/030546054.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/030741372.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/030817154.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/033021995.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170601/033210383.png?imageslim)

注意: 此处网关设置得不太合理, 并不是pxe服务器作为网关,导致无法上网, 签名配置文件中已改成正确的网关
 
**问题: yum -y install syslinux时,报Error downloading packages:**
```  
syslinux-4.05-13.el7.x86_64: [Errno 256] No more mirrors to try.
```
解决方法: 
```
[root@localhost Packages]# rm -fr /var/cache/yum/*
[root@localhost Packages]# yum clean all

```
