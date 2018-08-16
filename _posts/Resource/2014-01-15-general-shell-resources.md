---
layout: post
title: Shell 常用资源
category: resource
tags: Shell
keywords: Shell
description: 
---


## 常用指令

### 文件夹操作
```
查看文件夹大小   du -h --max-depth=1 /home/ys
查看驱动器空间   df -h 
```

### 压缩解压命令
```
tar zxvf aaa.tar.gz
tar zcvf aaa.tar.gz aaa
```

### 查看端口的占用
```
lsof -i:8087  查看8087端口的使用
```

### 一句话实现一个HTTP服务，把当前文件夹作为根目录
```
python -m SimpleHTTPServer
```

### 查看本地网络服务活动状态
```
lsof -i
```

### 查看自己的外网ip
```
curl ifconfig.me
```

### 下载整个网站
```
wget --random-wait -r -p -e robots=off -U mozilla http://www.example.com
```

### 后台运行一段不中止的程序，并可随时查看它的状态
```
screen -d -m -S some_ name ping my_router
```

### 创建守护进程
```
nohup python /var/www/a.py &
```

### 查看当前文件夹下文件（文件夹）大小
```
du -h --max-depth=1 .
```

### 查看所有磁盘大小
```
df -h
```

### 查看硬盘性能
```
iostat -x 1 10
```

### 查看系统整体性能
```
vmstat 1 10
```
### 诊断网络
```
mtr 
ping
traceroute
dig
```

### 列出本机监听的端口号
```
netstat –tlnp
netstat -anop
```

### 在远程机器上运行一段脚本
```
ssh user@server bash < /path/to/local/script.sh
```

### 批量重命名

```
# ls
1.txt  2.txt  3.txt  4.txt  5.txt
# rename 's/([0-9]).txt/file_$1.txt/' *.txt
# ls
file_1.txt  file_2.txt  file_3.txt  file_4.txt  file_5.txt
```

###截取部分文件
```
得到行号
#grep -n '18:00:00' a.log
142825:2014-07-02 18:00:00.047  INFO XXXX
142826:2014-07-02 18:00:00.117  INFO XXXX
截取文件（指定行号到结尾）
#sed '142825,$p' a.log > b.log
```