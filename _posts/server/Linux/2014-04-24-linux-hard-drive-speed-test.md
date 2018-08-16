---
layout: post
title: Linux硬盘速度测试
category: server
tags: Linux@server
keywords: linux hard disk speed test ssd
description: 
from: 

---

##1. 使用命令
###hdparm 
Usage:  hdparm  [options] [device] ..
Options:
--direct  use O_DIRECT to bypass page cache for timings
-t   perform device read timings
-T   perform cache read timings

参考链接：[linux hdparm命令参数及用法详解][hdparm]

###DD
利用dd测试读取/写入磁盘速度
time有计时作用，dd用于复制，从if读出，写到of。if=/dev/zero不产生输出，因此可以用来测试纯写速度。同理of=/dev/null不产生输入，可以用来测试纯读速度。bs是每次读或写的大小，即一个块的大小，count是读写块的数量。

参考链接：[Linux用DD命令测试磁盘读写速度][dd]

##2. 测试数据
###希捷企业级硬盘4T

Read

```
[root@localhost ~]# hdparm -Tt --direct /dev/sdb
/dev/sdb:
Timing O_DIRECT cached reads:   484 MB in  2.01 seconds = 240.42 MB/sec
Timing O_DIRECT disk reads:  464 MB in  3.01 seconds = 154.36 MB/sec

[root@localhost opt]# dd if=/opt/1Gb.file bs=64k |dd of=/dev/null
15625+0 records in
15625+0 records out
1024000000 bytes (1.0 GB) copied, 2.77624 seconds, 369 MB/s
2000000+0 records in
2000000+0 records out
1024000000 bytes (1.0 GB) copied, 2.7766 seconds, 369 MB/s
```

Write

```
[root@localhost opt]# time dd if=/dev/zero of=/opt/test.dbf bs=8k count=300000  conv=fdatasync
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB) copied, 17.8886 seconds, 137 MB/s

real     0m34.147s
user     0m0.041s
sys     0m4.755s
```
###Intel 520 240G SSD

```
Read
[root@localhost ~]# hdparm -Tt --direct /dev/sdb
/dev/sdb:
Timing O_DIRECT cached reads:   888 MB in  2.01 seconds = 442.52 MB/sec
Timing O_DIRECT disk reads:  658 MB in  3.01 seconds = 218.85 MB/sec

[root@localhost data]# dd if=/data/1Gb.file bs=64k |dd of=/dev/null
15625+0 records in
15625+0 records out
1024000000 bytes (1.0 GB) copied2000000+0 records in
2000000+0 records out
1024000000 bytes (1.0 GB) copied, 1.68645 seconds, 607 MB/s
, 1.68708 seconds, 607 MB/s
```
Write

```
[root@localhost data]# time dd if=/dev/zero of=/data/test.dbf bs=8k count=300000 conv=fdatasync 
300000+0 records in
300000+0 records out
2457600000 bytes (2.5 GB) copied, 13.7534 seconds, 179 MB/s

real     0m14.125s
user     0m0.038s
sys     0m4.190s
```

[hdparm]:http://www.linuxso.com/command/hdparm.html
[dd]:http://zhuzike.com/linux_dd/




















