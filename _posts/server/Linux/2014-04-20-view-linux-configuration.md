---
layout: post
title: 查看Linux资源、配置信息
category: server
tags: Linux@server
keywords: linux configuration status
description: 
from: 

---

##Linux
###查看Linux内核
{% highlight sh %}
[root@localhost ~]# uname -a
Linux localhost 2.6.18-348.el5 #1 SMP Tue Jan 8 17:53:53 EST 2013 x86_64 x86_64 x86_64 GNU/Linux
{% endhighlight %}

###查看Linux发布版本
{% highlight sh %}
[root@localhost ~]# head -n 1 /etc/issue
CentOS release 5.9 (Final)
{% endhighlight %}

##CPU 
###查看CPU型号、进程数
{% highlight sh %}
[root@localhost ~]#  cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
     16  Intel(R) Xeon(R) CPU           E5620  @ 2.40GHz
{% endhighlight %}

###查看CPU位数
{% highlight sh %}
[root@localhost ~]# getconf LONG_BIT
64
{% endhighlight %}

##硬盘
###查看挂载
{% highlight sh %}
[root@localhost ~]# fdisk -l

Disk /dev/sda: 500.1 GB, 500107862016 bytes
255 heads, 63 sectors/track, 60801 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          33      265041   83  Linux
/dev/sda2              34       32668   262140637+  83  Linux
/dev/sda3           32669       36845    33551752+  82  Linux swap / Solaris

Disk /dev/sdb: 240.0 GB, 240057409536 bytes
255 heads, 63 sectors/track, 29185 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1       29186   234431063+  83  Linux
{% endhighlight %}

{% highlight sh %}
[root@localhost ~]# cat /proc/partitions
major minor  #blocks  name

   8     0  488386584 sda
   8     1     265041 sda1
   8     2  262140637 sda2
   8     3   33551752 sda3
   8    16 2930266584 sdb
   8    17 2930266550 sdb1
{% endhighlight %}

###查看硬盘信息
{% highlight sh %}
[root@localhost ~]#  smartctl -a /dev/sdb
smartctl 5.42 2011-10-20 r3458 [x86_64-linux-2.6.18-348.el5] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF INFORMATION SECTION ===
Device Model:     INTEL SSDSC2BW240A4
Serial Number:    BTDA33130BMJ2403GN
LU WWN Device Id: 5 5cd2e4 04b41d516
Firmware Version: DC12
User Capacity:    240,057,409,536 bytes [240 GB]
Sector Size:      512 bytes logical/physical
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   8
ATA Standard is:  ACS-2 (unknown minor revision code: 0xffff)
Local Time is:    Sat Apr 19 17:48:32 2014 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
....
{% endhighlight %}

###查看分区使用情况
{% highlight sh %}
[root@localhost ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda2             243G  5.1G  225G   3% /
/dev/sda1             251M   17M  222M   7% /boot
tmpfs                  16G     0   16G   0% /dev/shm
/dev/sdb1             2.7T  202M  2.6T   1% /opt
{% endhighlight %}

##内存
###概要查看内存情况
{% highlight sh %}
[root@localhost ~]# free -m
             total       used       free     shared    buffers     cached
Mem:         32177      29218       2959          0         60      12811
-/+ buffers/cache:      16346      15830
Swap:        32765          0      32765
{% endhighlight %}
    参数解释
    total 内存总数
    used 已经使用的内存数
    free 空闲的内存数
    shared 多个进程共享的内存总额
    buffers Buffer Cache和cached Page Cache 磁盘缓存的大小
    \-buffers/cache (已用)的内存数:used - buffers - cached
    \+buffers/cache (可用)的内存数:free + buffers + cached

###查看内存详细使用
{% highlight sh %}
[root@localhost ~]# cat /proc/meminfo |more
MemTotal:     32949612 kB
MemFree:       3028672 kB
Buffers:         64292 kB
Cached:       12489420 kB
SwapCached:          0 kB
Active:       21410240 kB
.....
{% endhighlight %}

###查看内存型号
{% highlight sh %}
[root@localhost ~]# dmidecode -t memory
{% endhighlight %}

\# dmidecode 2.11
{% highlight sh %}
SMBIOS 2.6 present.

Handle 0x0014, DMI type 16, 15 bytes
Physical Memory Array
     Location: System Board Or Motherboard
     Use: System Memory
     Error Correction Type: Multi-bit ECC
     Maximum Capacity: 192 GB
     Error Information Handle: Not Provided
     Number Of Devices: 6

Handle 0x0016, DMI type 17, 28 bytes
Memory Device
     Array Handle: 0x0014
     Error Information Handle: Not Provided
     Total Width: 72 bits
     Data Width: 64 bits
     Size: 8192 MB
     Form Factor: DIMM
     Set: None
     Locator: P1-1A
     Bank Locator: BANK0
     Type: DDR3
     Type Detail: Other
     Speed: 1066 MHz
     Manufacturer: Samsung      
     Serial Number: 1802FE33
     Asset Tag: AssetTagNum0
     Part Number: M393B1K70DH0-YH9 
     Rank: Unknown
{% endhighlight %}

##网卡
###查看网卡配置
{% highlight sh %}
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
# Intel Corporation 82576 Gigabit Network Connection
DEVICE=eth0
ONBOOT=yes
BOOTPROTO=static
IPADDR=117.121.19.202
NETMASK=255.255.255.0
GATEWAY=117.121.19.1
DNS1=202.106.0.20
{% endhighlight %}

###查看网卡信息
{% highlight sh %}
[root@localhost ~]#  ethtool eth0
Settings for eth0:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Supports auto-negotiation: Yes
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Advertised auto-negotiation: Yes
	Speed: 1000Mb/s
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 1
	Transceiver: internal
	Auto-negotiation: on
	Supports Wake-on: pumbg
	Wake-on: g
	Current message level: 0x00000007 (7)
	Link detected: yes
{% endhighlight %}

##其他
###查看机器所有信息
{% highlight sh %}
[root@localhost ~]# dmidecode |more
\# dmidecode 2.11
SMBIOS 2.6 present.
51 structures occupying 2686 bytes.
....
{% endhighlight %}

{% highlight sh %}
[root@localhost ~]# dmesg |more
Linux version 2.6.18-348.el5 (mockbuild@builder10.centos.org) (gcc version 4.1.2
 20080704 (Red Hat 4.1.2-54)) #1 SMP Tue Jan 8 17:53:53 EST 2013
Command line: ro root=LABEL=/
BIOS-provided physical RAM map:
 BIOS-e820: 0000000000010000 - 000000000009cc00 (usable)
....
{% endhighlight %}

###查看USB信息
{% highlight sh %}
[root@localhost ~]# lsusb
Bus 003 Device 001: ID 0000:0000  
Bus 005 Device 002: ID 046b:ff10 American Megatrends, Inc. Virtual Keyboard and Mouse
Bus 005 Device 001: ID 0000:0000  
Bus 002 Device 001: ID 0000:0000  
Bus 001 Device 001: ID 0000:0000  
Bus 004 Device 001: ID 0000:0000  
Bus 006 Device 001: ID 0000:0000  
Bus 008 Device 001: ID 0000:0000  
Bus 007 Device 001: ID 0000:0000
{% endhighlight %}

###查看环境变量
{% highlight sh %}
[root@localhost ~]# env
HOSTNAME=localhost
TERM=xterm
SHELL=/bin/bash
HISTSIZE=1000
SSH_CLIENT=182.18.33.15 51148 60000
SSH_TTY=/dev/pts/0
{% endhighlight %}

###查看系统服务
{% highlight sh %}
[root@localhost ~]# chkconfig --list   
NetworkManager 	0:off	1:off	2:off	3:off	4:off	5:off	6:off
acpid          	0:off	1:off	2:on	3:off	4:on	5:off	6:off
anacron        	0:off	1:off	2:on	3:off	4:on	5:off	6:off
atd            	0:off	1:off	2:off	3:off	4:on	5:off	6:off
auditd         	0:off	1:off	2:on	3:off	4:on	5:off	6:off
autofs         	0:off	1:off	2:off	3:on	4:on	5:on	6:off
avahi-daemon   	0:off	1:off	2:off	3:off	4:on	5:off	6:off
{% endhighlight %}

###查看安装的所有软件包
{% highlight sh %}
[root@localhost ~]# rpm -qa 
basesystem-8.0-5.1.1.el5.centos
rmt-0.4b41-6.el5
glibc-2.5-107
mktemp-1.5-24.el5
libstdc++-4.1.2-54.el5
libusb-0.1.12-6.el5
ncurses-5.5-24.20060715
{% endhighlight %}