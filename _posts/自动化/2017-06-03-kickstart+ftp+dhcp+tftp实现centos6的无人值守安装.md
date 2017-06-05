---
layout: post
category: "自动化"
title:  "kickstart+http+dhcp+tftp实现centos6的无人值守安装"
tags: [马哥,Linux,kickstart,自动化]
---  

>环境
>IP :  172.16.0.32   centos 6.6

### 一.安装相关包
```
安装图形界面
[root@localhost ~]# yum groupinstall "Desktop" "X Window System" "Chinese Support"
[root@localhost ~]# yum -y install dhcp tftp-server tftp syslinux vsftpd

```

### 二. 配置dhcp服务
```
[root@localhost ~]# cd /etc/dhcp/
[root@localhost dhcp]# cp /usr/share/doc/dhcp-4.1.1/dhcpd.conf.sample ./dhcpd.conf
cp: overwrite `./dhcpd.conf'? y
[root@localhost dhcp]# vim dhcpd.conf
option domain-name "magedu.com";
option routers 172.16.0.1;   
option domain-name-servers 202.106.0.20;

subnet 172.16.0.0 netmask 255.255.0.0 {
    range 172.16.200.100 172.16.200.200;
    filename "pxelinux.0";
    next-server 172.16.0.32;
}
[root@localhost dhcp]# service dhcpd configtest
Syntax: OK

[root@localhost dhcp]# chkconfig dhcpd on
[root@localhost dhcp]# service dhcpd start

[root@localhost dhcp]# ss -unl | grep :67
UNCONN     0      0                         *:67                       *:*

```

### 三.tftp服务
```
[root@localhost dhcp]# vim /etc/xinetd.d/tftp
disable                 = no

[root@localhost dhcp]# service xinetd restart
[root@localhost dhcp]# chkconfig xinetd on
[root@localhost dhcp]# ss -unl | grep :69
UNCONN     0      0                         *:69                       *:*


[root@localhost dhcp]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/

测试tftp能否下载
[root@localhost dhcp]# cd /tmp
[root@localhost tmp]# tftp 172.16.0.32
tftp> get pxelinux.0
tftp> quit
[root@localhost tmp]# ls
gconfd-gdm      orbit-gdm           pulse-w7f0tLvBt2CE  tmpubqNCd
gconfd-root     orbit-root          pulse-XLd7li5gCIxS  yum.log
keyring-jxLAwk  pulse-1PbAudlT4Zed  pxelinux.0
```

### 四. 准备pxelinux
```
挂载光盘镜像

[root@localhost ~]# mkdir /media/cdrom
[root@localhost ~]# mount -r /dev/cdrom /media/cdrom

[root@localhost ~]# cp /media/cdrom/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/
[root@localhost ~]# cp /media/cdrom/isolinux/{boot.msg,vesamenu.c32,splash.jpg} /var/lib/tftpboot/
[root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux.cfg
[root@localhost ~]# cp /media/cdrom/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
[root@localhost ~]# chmod +w /var/lib/tftpboot/pxelinux.cfg/default
[root@localhost ~]# vim /var/lib/tftpboot/pxelinux.cfg/default
添加一个label
label autoinst
  menu label ^Auto Install CenOS
  menu default
  kernel vmlinuz
  append initrd=initrd.img ks=ftp://172.16.0.32/pub/centos6.cfg


注意: 去掉原来label的menu default

[root@localhost ~]# cat  /var/lib/tftpboot/pxelinux.cfg/default
default vesamenu.c32
#prompt 1
timeout 600

display boot.msg

menu background splash.jpg
menu title Welcome to CentOS 6.6!
menu color border 0 #ffffffff #00000000
menu color sel 7 #ffffffff #ff000000
menu color title 0 #ffffffff #00000000
menu color tabmsg 0 #ffffffff #00000000
menu color unsel 0 #ffffffff #00000000
menu color hotsel 0 #ff000000 #ffffffff
menu color hotkey 7 #ffffffff #ff000000
menu color scrollbar 0 #ffffffff #00000000

label autoinst
  menu label ^Auto Install CenOS
  menu default
  kernel vmlinuz
  append initrd=initrd.img ks=ftp://172.16.0.32/pub/centos6.cfg
label linux
  menu label ^Install or upgrade an existing system
  kernel vmlinuz
  append initrd=initrd.img
label vesa
  menu label Install system with ^basic video driver
  kernel vmlinuz
  append initrd=initrd.img xdriver=vesa nomodeset
label rescue
  menu label ^Rescue installed system
  kernel vmlinuz
  append initrd=initrd.img rescue
label local
  menu label Boot from ^local drive
  localboot 0xffff
label memtest86
  menu label ^Memory test
  kernel memtest
  append -



```

### 五.准备yum仓库
```
[root@localhost ~]# mkdir /var/ftp/pub/centos
[root@localhost ~]# \cp -rf /media/cdrom/* /var/ftp/pub/centos
或者是挂载目录
[root@localhost ~]# mount --bind /media/cdrom /var/ftp/pub/centos/

```

### 六. 准备kickstart文件
```
进入系统图形化界面
[root@localhost yum.repos.d]# yum install  system-config-kickstart
[root@localhost ~]# system-config-kickstart

```
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/095955903.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/100048649.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/101051505.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/113443287.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114153072.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114254222.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114405088.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114422176.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114508253.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114620411.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114729588.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/114804279.png?imageslim)

微调cfg文件
```
[root@localhost ~]# vim centos6.cfg
#logging --level=info
#repo --name="CentOS" --baseurl=cdrom:sr0 --cost=100
#network  --bootproto=dhcp --device=eth1 --onboot=on  不注释导致安装失败

[root@localhost ~]# cat centos6.cfg
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Firewall configuration
firewall --disabled
# Install OS instead of upgrade
install
# Use network installation
url --url="ftp://172.16.0.32/pub/centos"
#repo --name="CentOS" --baseurl=cdrom:sr0 --cost=100
# Root password
rootpw --iscrypted $1$cVdlAuSw$5tYu9Bx6iwKTgGvxTlarJ1
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
# System keyboard
keyboard us
# System language
lang en_US
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# Installation logging level
#logging --level=info
# Reboot after installation
reboot
# System timezone
timezone  Asia/Shanghai
# Network information
network  --bootproto=dhcp --device=eth0 --onboot=on
#network  --bootproto=dhcp --device=eth1 --onboot=on
# System bootloader configuration
bootloader --append="crashkernel=auto rhgb quiet" --location=mbr --driveorder="sda"
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --asprimary --fstype="ext4" --size=100
part swap --fstype="swap" --size=8000
part / --fstype="ext4" --grow --size=1

%packages --nobase
@core

%end



cfg文件拷贝都ftp目录
[root@localhost ~]# cp centos6.cfg /var/ftp/pub/
```
### 七.启动vsftpd
```
[root@localhost ~]# ss -tnl | grep 21
LISTEN     0      32                        *:21                       *:*

```

### 八. 验证
新建虚拟机, 过程略  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/122707165.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/124315210.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/125100199.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170603/125805601.png?imageslim)

**问题: 安装时无法识别eth1**  
解决方法: 将kickstart文件中的eth1项注释

