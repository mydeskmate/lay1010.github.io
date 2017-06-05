---
layout: post
category: "自动化"
title:  "cobbler实现centos7和centos6的无人值守安装"
tags: [马哥,Linux,kickstart,自动化]
---  


>环境: 
>系统: CentOS 7.2
>ip: 172.16.0.11


### 一. Cobbler安装准备
Cobbler是一个Linux服务器安装的服务，可以通过网络启动(PXE)的方式来快速安装、重装物理服务器和虚拟机，同时还可以管理DHCP，DNS等。  

Cobbler可以使用命令行方式管理，也提供了基于Web的界面管理工具(cobbler-web)，还提供了API接口，可以方便二次开发使用。  

Cobbler是较早前的kickstart的升级版，优点是比较容易配置，还自带web界面比较易于管理。  

Cobbler内置了一个轻量级配置管理系统，但它也支持和其它配置管理系统集成，如Puppet，暂时不支持SaltStack。   

#### 1.1 系统环境准备
```
[root@localhost ~]# getenforce
Disabled
[root@localhost ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)

```
#### 1.2 配置163的yum源和阿里云的epel源
```
[root@localhost ~]# mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
[root@localhost ~]# wget -O /etc/yum.repos.d/163.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
[root@localhost ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

```

### 二. 安装配置cobbler
#### 2.1 安装Cobbler
```
[root@localhost ~]# yum -y install dhcp tftp tftp-server cobbler cobbler-web pykickstart httpd
[root@localhost ~]# rpm -ql cobbler  # 查看安装的文件，下面列出部分。
/etc/cobbler                  # 配置文件目录
/etc/cobbler/settings         # cobbler主配置文件，这个文件是YAML格式，Cobbler是python写的程序。
/etc/cobbler/dhcp.template    # DHCP服务的配置模板
/etc/cobbler/tftpd.template   # tftp服务的配置模板
/etc/cobbler/rsync.template   # rsync服务的配置模板
/etc/cobbler/iso              # iso模板配置文件目录
/etc/cobbler/pxe              # pxe模板文件目录
/etc/cobbler/power            # 电源的配置文件目录
/etc/cobbler/users.conf       # Web服务授权配置文件
/etc/cobbler/users.digest     # 用于web访问的用户名密码配置文件
/etc/cobbler/dnsmasq.template # DNS服务的配置模板
/etc/cobbler/modules.conf     # Cobbler模块配置文件
/var/lib/cobbler              # Cobbler数据目录
/var/lib/cobbler/config       # 配置文件
/var/lib/cobbler/kickstarts   # 默认存放kickstart文件
/var/lib/cobbler/loaders      # 存放的各种引导程序
/var/www/cobbler              # 系统安装镜像目录
/var/www/cobbler/ks_mirror    # 导入的系统镜像列表
/var/www/cobbler/images       # 导入的系统镜像启动文件
/var/www/cobbler/repo_mirror  # yum源存储目录
/var/log/cobbler              # 日志目录
/var/log/cobbler/install.log  # 客户端系统安装日志
/var/log/cobbler/cobbler.log  # cobbler日志
``` 

#### 2.2 配置Cobbler
```
启动httpd和Cobbler并设置为自启动
[root@localhost ~]# systemctl start httpd
[root@localhost ~]# systemctl enable httpd.service
[root@localhost ~]# systemctl start cobblerd.service
[root@localhost ~]# systemctl enable cobblerd.service

检查cobbler环境: 
[root@localhost cobbler]# cobbler check
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolva         ble hostname or IP for the boot server as reachable by all machines that will us         e it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boo         t server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/tftp
4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/         x86_64 netbooting, you may ensure that you have installed a *recent* version of          the syslinux package installed and can ignore this message entirely.  Files in t         his directory, should you want to support all architectures, should include pxel         inux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is th         e easiest way to resolve these requirements.
5 : enable and start rsyncd.service with systemctl
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler'          and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your         -password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.


逐个解决以上问题: 
[root@localhost ~]# cd /etc/cobbler/
[root@localhost cobbler]# vim settings

问题1: ip修改为cobber本机ip
server: 172.16.0.11
 
问题2: next-server修改为tftp-server
next_server: 172.16.0.11

问题3: 
[root@localhost cobbler]# vim /etc/xinetd.d/tftp
disable                 = no

问题4: 准备bootloader
[root@localhost cobbler]# cp /usr/share/syslinux/{pxelinux.0,menu.c32} /var/lib/cobbler/loaders/

问题5: 启动rsyncd
[root@localhost cobbler]# systemctl start rsyncd.socket
[root@localhost cobbler]# systemctl enable rsyncd.socket

问题6: 可忽略

问题7: 为系统设置复杂密码
[root@localhost cobbler]# openssl passwd -1 -salt 'han' '123456'
$1$han$BtNvGZePxwQMW5gC6IUep1
[root@localhost cobbler]# vim /etc/cobbler/settings
default_password_crypted: "$1$han$BtNvGZePxwQMW5gC6IUep1"

问题8: 可忽略

重新检查: 
[root@localhost ~]# systemctl restart cobblerd.service
[root@localhost ~]# cobbler check
[root@localhost ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
2 : enable and start rsyncd.service with systemctl
3 : debmirror package is not installed, it will be required to manage debian deployments and repositories
4 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.

以上均已处理或可以忽略

cobbler配置同步
[root@localhost ~]# cobbler sync

```
  

>cobbler的运行依赖于dhcp、tftp、rsync及dns服务。其中dhcp可由dhcpd(isc)提供，也可由dnsmasq提供；tftp可由tftp-server程序包提供，也可由cobbler自带的tftp功能提供；rsync由rsync程序包提供；dns可由bind提供，也可由dnsmasq提供。

>cobbler可自行管理这些服务中的部分甚至是全部，但需要配置/etc/cobbler/settings文件中的“manage_dhcp”、“manage_tftpd”、“manage_rsync”和“manage_dns”分别进行定义。另外，由于每种服务都有着不同的实现方式，如若需要进行自定义，需要通过修改/etc/cobbler/modules.conf配置文件中各服务的模块参数的值来实现。

>本文采用了独立管理的方式，即不通过cobbler来管理这些服务。


### 三. 配置dhcp
```
[root@localhost ~]# cd /etc/dhcp/
[root@localhost dhcp]# \cp /usr/share/doc/dhcp*/dhcpd.conf.example dhcpd.conf
[root@localhost dhcp]# grep -v '^#'  dhcpd.conf

option domain-name "magedu.com";
option routers 172.16.0.1;
option domain-name-servers 202.106.0.20, 114.114.114.114;

default-lease-time 600;
max-lease-time 7200;

log-facility local7;

subnet 172.16.0.0 netmask 255.255.0.0 {
    range 172.16.100.200 172.16.100.230;
    filename "pxelinux.0";
    next-server 172.16.0.11;
}


[root@localhost dhcp]# systemctl start dhcpd.service
[root@localhost dhcp]# systemctl enable dhcpd.service
```

### 四. 配置tftp
```
[root@localhost dhcp]# systemctl start tftp.socket
[root@localhost dhcp]# systemctl enable tftp.socket
[root@localhost dhcp]# ss -unl | grep 69
UNCONN     0      0           :::69                      :::*

```

### 五. Cobbler的命令行管理
#### 5.1 查看命令帮助
```
[root@localhost ~]# cobbler
usage
=====
cobbler <distro|profile|system|repo|image|mgmtclass|package|file> ...
        [add|edit|copy|getks*|list|remove|rename|report] [options|--help]
cobbler <aclsetup|buildiso|import|list|replicate|report|reposync|sync|validateks|version|signature|get-loaders|hardlink> [options|--help]

[root@localhost ~]# cobbler import --help  #导入镜像定义distro
Usage: cobbler import [options]

Options:
  -h, --help            show this help message and exit
  --arch=ARCH           OS architecture being imported
  --breed=BREED         the breed being imported
  --os-version=OS_VERSION
                        the version being imported
  --path=PATH           local path or rsync location
  --name=NAME           name, ex 'RHEL-5'
  --available-as=AVAILABLE_AS
                        tree is here, don't mirror
  --kickstart=KICKSTART_FILE
                        assign this kickstart file
  --rsync-flags=RSYNC_FLAGS
                        pass additional flags to rsync

cobbler check    核对当前设置是否有问题
cobbler list     列出所有的cobbler元素
cobbler report   列出元素的详细信息
cobbler sync     同步配置到数据目录,更改配置最好都要执行下
cobbler reposync 同步yum仓库
cobbler distro   查看导入的发行版系统信息
cobbler system   查看添加的系统信息
cobbler profile  查看配置信息

```

#### 5.2 导入镜像定义distro

```
挂载系统镜像
[root@localhost ~]# mkdir /media/cdrom
[root@localhost ~]# mount -r /dev/cdrom /media/cdrom
从光盘导入文件定义distro
[root@localhost ~]# cobbler import --name="CentOS-7.2-x86_64" --path=/media/cdrom
# --path 镜像路径
# --name 为安装源定义一个名字,distro名字
# --arch 指定安装源是32位、64位、ia64, 目前支持的选项有: x86│x86_64│ia64
# 安装源的唯一标示就是根据name参数来定义，本例导入成功后，安装源的唯一标示就是：CentOS-7.1-x86_64，如果重复，系统会提示导入失败。

注意： import自动为导入的distro自动生成一个同名的profile, 并同时提供了一个最小化安装的kickstart文件，可以实现自动化安装,但可能并不符合需求

列出当前的distro
[root@localhost ~]# cobbler distro list
   CentOS-7.2-x86_64

# 镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7.2-x86_64目录下。因此/var/www/cobbler目录必须具有足够容纳安装文件的空间。
[root@localhost ~]# cd /var/www/cobbler/ks_mirror/
[root@localhost ks_mirror]# ls
CentOS-7.2-x86_64  config
[root@localhost ks_mirror]# ls CentOS-7.2-x86_64/
CentOS_BuildTag  GPL       LiveOS    RPM-GPG-KEY-CentOS-7
EFI              images    Packages  RPM-GPG-KEY-CentOS-Testing-7
EULA             isolinux  repodata  TRANS.TBL

列出当前的profile
[root@localhost ~]# cobbler profile list
   CentOS-7.2-x86_64
```

#### 5.3 自定义ks.cfg
```
# Cobbler的ks.cfg文件存放位置
[root@localhost ks_mirror]# cd /var/lib/cobbler/kickstarts/
[root@localhost kickstarts]# ls
default.ks    install_profiles  sample_autoyast.xml  sample_esxi4.ks  sample_old.seed
esxi4-ks.cfg  legacy.ks         sample_end.ks(默认使用的ks文件)        sample_esxi5.ks  sample.seed
esxi5-ks.cfg  pxerescue.ks      sample_esx4.ks       sample.ks

使用pxe的kickstart文件,并修改
[root@localhost kickstarts]# vim centos7.cfg
url --url="http://172.16.0.11/cobbler/ks_mirror/CentOS-7.2-x86_64/"

注意: 自定义的ks文件和模版的ks文件稍有不同, 某些变量无法从配置文件中获取,如 url --url=$tree, 
rootpw --iscrypted $default_password_crypted

[root@localhost kickstarts]# cat centos7.cfg
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
url --url="http://172.16.0.11/cobbler/ks_mirror/CentOS-7.2-x86_64/"
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

# 查看指定的profile设置
[root@localhost kickstarts]# cobbler profile report --name=CentOS-7.2-x86_64
Name                           : CentOS-7.2-x86_64
TFTP Boot Files                : {}
Comment                        :
DHCP Tag                       : default
Distribution                   : CentOS-7.2-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks   默认ks文件
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 :
Internal proxy                 :
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      :
Virt RAM (MB)                  : 512
Virt Type                      : kvm

# 编辑profile，修改关联的ks文件
[root@localhost kickstarts]# cobbler profile edit --name=CentOS-7.2-x86_64 --kickstart=/var/lib/cobbler/kickstarts/centos7.cfg

[root@localhost kickstarts]# cobbler profile report --name=CentOS-7.2-x86_64             Name                           : CentOS-7.2-x86_64
TFTP Boot Files                : {}
Comment                        :
DHCP Tag                       : default
Distribution                   : CentOS-7.2-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/centos7.cfg
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 :
Internal proxy                 :
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      :
Virt RAM (MB)                  : 512
Virt Type                      : kvm


# 每次修改完都要同步一次
[root@localhost kickstarts]# cobbler sync

```
#### 5.4  安装CentOS 7系统  
新建一台虚拟机，启动  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/152340417.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/153718574.png?imageslim)

local： 本地硬盘启动  
CentOS-7-x86_64  :  profile名字  

### 六. 使用cobbler-web管理cobbler

#### 6.1 使用authn_configfile模块认证cobbler_web用户
```
[root@localhost ~]# cd /etc/cobbler/
[root@localhost cobbler]# cp modules.conf{,.bak}
[root@localhost cobbler]# vim modules.conf
module = authn_configfile

创建认证文件
[root@localhost cobbler]# htdigest -c /etc/cobbler/users.digest Cobbler cblradmin
Adding password for cblradmin in realm Cobbler.
New password:
Re-type new password:
输入密码： 123456
账号： cdlradmin

[root@localhost cobbler]# systemctl restart cobblerd.service

```

![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/155004492.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/155334049.png?imageslim)
导入镜像定义distro
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/013910976.png?imageslim)
备注: Prefix 即 name


**问题: 访问http://172.16.0.12/cobbler_web提示没有权限**
```
Forbidden

You don't have permission to access /cobbler_web on this server.
```
分析日志
```
[Sun Jun 04 01:26:31.669590 2017] [ssl:error] [pid 5536] [client 172.16.0.10:52276] AH02219: access to /usr/share/cobbler/web/cobbler.wsgi failed, reason: SSL connection required

```
解决方法: 
```
使用https地址访问: https://172.16.0.12/cobbler_web
在浏览器上添加例外
```


### 七. 配置CentOS 6.6镜像
#### 7.1挂载镜像
```
卸载centos7镜像
[root@localhost ~]# umount /media/cdrom
[root@localhost ~]# mount -r /dev/cdrom /media/cdrom
```

#### 7.2 导入镜像
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/164322531.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/165855452.png?imageslim)

#### 7.3 添加kickstart文件
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/163212247.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/170119138.png?imageslim)

#### 7.4 指定kickstart文件
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/170439120.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/170523180.png?imageslim)

centos6.cfg
```
[root@localhost ~]# cat centos6.cfg
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Firewall configuration
firewall --disabled
# Install OS instead of upgrade
install
# Use network installation
url --url="http://172.16.0.11/cobbler/ks_mirror/CentOS-6.6-x86_64/"
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

```

#### 7.5 配置同步
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/171010762.png?imageslim)

#### 7.6 安装centos 6系统
新建虚拟机,启动  
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/171244944.png?imageslim)
![mark](http://ohfysad7j.bkt.clouddn.com/blog/20170604/172129822.png?imageslim)


### 八. 附加： ks.cfg文件(供参考)
文件大部分参数含义见kickstart文章，此处只讲一些不同的地方。同时可以参考模板文件。    
```
[root@linux-node1 kickstarts]# cat CentOS-7.1-x86_64.cfg
# Cobbler for Kickstart Configurator for CentOS 7.1 by yao zhang
install
url --url=$tree  # 这些$开头的变量都是调用配置文件里的值。
text
lang en_US.UTF-8
keyboard us
zerombr
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb quiet"
# Network information
$SNIPPET('network_config')
timezone --utc Asia/Shanghai
authconfig --enableshadow --passalgo=sha512
rootpw  --iscrypted $default_password_crypted
clearpart --all --initlabel
part /boot --fstype xfs --size 1024  # CentOS7系统磁盘默认格式xfs
part swap --size 1024
part / --fstype xfs --size 1 --grow
firstboot --disable
selinux --disabled
firewall --disabled
logging --level=info
reboot
%pre
$SNIPPET('log_ks_pre')
$SNIPPET('kickstart_start')
$SNIPPET('pre_install_network_config')
# Enable installation monitoring
$SNIPPET('pre_anamon')
%end
%packages
@base
@compat-libraries
@debugging
@development
tree
nmap
sysstat
lrzsz
dos2unix
telnet
iptraf
ncurses-devel
openssl-devel
zlib-devel
OpenIPMI-tools
screen
%end
%post
systemctl disable postfix.service
%end
```