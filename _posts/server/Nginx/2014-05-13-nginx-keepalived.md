---
layout: post
title: 分享我做的nginx+keepalived做的主主架构
category: server
tags: Nginx@server
keywords: nginx keepalived 负载均衡 mysql高可用
description: 
from: http://dl528888.blog.51cto.com/2382721/830807

---

最近帮朋友设计他们公司的系统架构，这是第2次进行帮他设计了，第一个是把他的lanmp架构（所有的应用与服务、数据库都在一个服务器里）改成1+1模式（nginx+mysql），最近由于他公司的名气上升，每天的在线数在4000-6000左右，并发最多能到9000左右，现有的架构有一些支撑不了，所有我又重新的帮他设计了一下。

我新设计的架构为dns（轮询）+nginx+keepalived（主主模式，2台服务器）+web（应用服务器2台服务器）+mysql（采用drbd+hearbeat+mysql，2台服务器），总共6台服务器，系统均为rhel 5.4 x86_64。

此篇文章既可以是一个完整性的架构，也可以是nginx+keepalived单主、双主模式负载均衡，nginx的反向代理，msyql的drbd+heartbeat+msyql的高可用，以及通过nfs来实现web服务器间的数据共享，分开与结合都是一篇很有深度的文章（个人认为），所以我没有把这些用到的技术分开来写，通过一篇文章写能够给大家更好的理解，否则单个的描述可能会造成大家对单个技术能更好的理解，但对整体的不会有太好的了解（虽然我也想分开写，这样能多几个推荐）。

##下面是我做的架构图：

![架构图](/public/upload/server/mysql-dbrd-heartbeat-structure.jpg)

网络通信的过程如下：

1. 用户想进行浏览www.netnvo.com网站的内容，先进行本地dns查看，是否有此网站的ip，如果没有则去上级dns进行查找；
2. 如果找到，则跳转到相应的ip上；
3. 请求到达负载均衡层时，根据相应的策略把请求分发到web服务器里；
4. web服务器接收到请求，进行处理并发送读写操作请求给数据库；
5. 数据库收到读写操作请求，完成此项操作并反馈给web服务器；
6. web服务器收到反馈并发送处理完成的数据到负载均衡里；
7. 负载均衡收到数据并发送给用户；
8. 用户收到了数据，以一定的方式进行查看（不限于浏览器）；

##下面是我具体的安装过程

###一. 负载均衡层

1. 在test1与test2里安装nginx

可以使用我之前发的文章[“模块化的安装lnmp脚本”][lnmp-install]来进行自动化安装nginx。

 * 先进行下载lnmp脚本

{% highlight sh %}
[root@test1 tmp]# wget http://202.96.42.117/soft/install_lnmp.tar.gz
{% endhighlight %} 

 * 先解压

{% highlight sh %}
[root@nginx1 tmp]# tar zxf install_lnmp.tar.gz  
[root@nginx1 tmp]# ll  
total 64456  
-rwxr-xr-x 1 root root    13767 Mar 25 01:23 install_lnmp.sh  
-rw-r--r-- 1 root root 65910919 Mar 25 01:24 install_lnmp.tar.gz  
drwxr-xr-x 5 root root     4096 Mar 23 09:54 soft  
{% endhighlight %}

 * 然后再检测是否安装lnmp

{% highlight sh %}
[root@test1 tmp]# sh install_lnmp.sh install_check  
Sat Apr  7 03:29:46 MDT 2012 Start install!  
========================== Check install ================================  
Error: /usr/local/nginx not found!!!  
Error: /usr/local/php not found!!!  
Error: /usr/local/mysql not found!!!  
========================== Check install ================================  
Sorry,Failed to install LNMP!  
Please check errors and logs.  
Sat Apr  7 03:29:46 MDT 2012 Finish install!  
Total runtime: 0 Seconds  
{% endhighlight %}

 * 从输出看到没有安装lnmp软件
那现在我们进行nginx安装，在安装之前，先进行依赖库的安装
{% highlight sh %}
[root@test1 tmp]# sh install_lnmp.sh init 
{% endhighlight %}
 * 然后再安装nginx
{% highlight sh %}
[root@test1 tmp]# sh install_lnmp.sh install_nginx 
{% endhighlight %}
 * 完成之后进行检测
{% highlight sh %}
[root@test1 tmp]#  sh install_lnmp.sh install_check  
Sat Apr  7 04:48:05 MDT 2012 Start install!  
========================== Check install ================================  
/usr/local/nginx [found]  
Error: /usr/local/php not found!!!  
Error: /usr/local/mysql not found!!!  
========================== Check install ================================  
Sorry,Failed to install LNMP!  
Please check errors and logs.  
Sat Apr  7 04:48:05 MDT 2012 Finish install!  
Total runtime: 0 Seconds  
{% endhighlight %}
 * 可以发现nginx是安装完成
{% highlight sh %}
[root@test1 sbin]# curl -i 127.0.0.1  
HTTP/1.1 200 OK  
Server: YWS/1.0  
Date: Sat, 07 Apr 2012 10:08:39 GMT  
Content-Type: text/html  
Content-Length: 151  
Last-Modified: Sat, 07 Apr 2012 09:38:53 GMT  
Connection: keep-alive  
Accept-Ranges: bytes  
 
<html> 
<head> 
<title>Welcome to nginx!</title> 
</head> 
<body bgcolor="white" text="black"> 
<center><h1>Welcome to nginx!</h1></center> 
</body> 
</html> 
{% endhighlight %}
能打开nginx的首页了

 * 下面是我为nginx负载均衡做的配置

nginx的配置为
{% highlight sh %} 
user  www www;  
 
worker_processes 8;  
 
error_log  /usr/local/nginx/logs/nginx_error.log  crit;  
 
pid        /usr/local/nginx/nginx.pid;  
 
#Specifies the value for maximum file descriptors that can be opened by this process.   
worker_rlimit_nofile 65535;  
 
events   
{  
  use epoll;  
  worker_connections 65535;  
}  
 
http   
{  
  include       mime.types;  
  default_type  application/octet-stream;  
 
  #charset  gb2312;  
        
  server_names_hash_bucket_size 128;  
  client_header_buffer_size 32k;  
  large_client_header_buffers 4 32k;  
  client_max_body_size 8m;  
        
  sendfile on;  
  tcp_nopush     on;  
 
  keepalive_timeout 60;  
 
  tcp_nodelay on;  
 
  fastcgi_connect_timeout 300;  
  fastcgi_send_timeout 300;  
  fastcgi_read_timeout 300;  
  fastcgi_buffer_size 64k;  
  fastcgi_buffers 4 64k;  
  fastcgi_busy_buffers_size 128k;  
  fastcgi_temp_file_write_size 128k;  
 
  gzip on;  
  gzip_min_length  1k;  
  gzip_buffers     4 16k;  
  gzip_http_version 1.0;  
  gzip_comp_level 2;  
  gzip_types       text/plain application/x-javascript text/css application/xml;  
  gzip_vary on;  
 
  #limit_zone  crawler  $binary_remote_addr  10m;  
 
    upstream web1       ##定义负载均衡组为web1  
{  
        ip_hash;                  
        server 10.1.88.168:80;  
        server 10.1.88.177:80;  
 
 }  
  server  
  {  
    listen       80;  
    server_name  www.netnov.com;  
    location / {   
    root /data/www;   
    index index.php index.htm index.html;   
    proxy_redirect off;   
    proxy_set_header Host $host;   
    proxy_set_header X-Real-IP $remote_addr;   
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;   
    proxy_pass http://web1;   
    }   
    access_log  off;   
  }                               
    location ~ .*\.(php|php5)?$  
    {        
      #fastcgi_pass  unix:/tmp/php-cgi.sock;  
      fastcgi_pass  127.0.0.1:9000;  
      fastcgi_index index.php;  
      include fastcgi.conf;  
   }  
      
    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$  
    {  
    expires      30d;  
    }  
    location ~ .*\.(js|css)?$  
    {  
      expires      1h;  
    }      
 
    location /                  ###当后端服务器遇到500、502、504、错误与超时，自动将请求转发给web1组的另一台服务器，达到故障转移  
    {  
    proxy_pass  http://web1;  
    proxy_next_upstream http_500 http_502 http_504 error timeout invalid_header;  
    include     /usr/local/nginx/conf/proxy.conf;  
}     
    log_format  access  '$remote_addr - $remote_user [$time_local] "$request" '  
              '$status $body_bytes_sent "$http_referer" '  
              '"$http_user_agent" $http_x_forwarded_for';  
    access_log  /usr/local/nginx/logs/access.log  access;  
}  
  server  
  {  
    listen  80;  
    server_name  status.netnov.com;  
 
    location / {  
    stub_status on;  
    access_log   off;  
    }  
  }  
}  
 
 {% endhighlight %}
test2也跟test1一样安装，就不在进行重复演示了

2.安装keepalived

 * 先下载master端的（也就是test1）
{% highlight sh %} 
[root@test1 src]# wget http://www.keepalived.org/software/keepalived-1.2.2.tar.gz  
[root@test1 src]# tar zxvf keepalived-1.2.2.tar.gz  
[root@test1 src]# cd keepalived-1.2.2  
[root@test1 keepalived-1.2.2]# ./configure --prefix=/usr/local/keepalived  
[root@test1 keepalived-1.2.2]# make  
[root@test1 keepalived-1.2.2]# make install  
[root@test1 keepalived-1.2.2]# cp /usr/local/keepalived/sbin/keepalived /usr/sbin/  
[root@test1 keepalived-1.2.2]# cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/  
[root@test1 keepalived-1.2.2]# cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  
[root@test1 keepalived-1.2.2]# mkdir /etc/keepalived  
[root@test1 tmp]# chkconfig --add keepalived  
[root@test1 tmp]# chmod 755 /etc/init.d/keepalived   
[root@test1 tmp]# chkconfig keepalived on  
[root@test1 tmp]#vim /etc/keepalived/keepalived.conf  
global_defs {  
 notification_email {  
 denglei@ctfo.com  
}  
notification_email_from root@test1.com  
smtp_server 127.0.0.1  
smtp_connect_timeout 30  
router_id LVS_DEVEL  
}  
vrrp_script chk_http_port {   
                script "/tmp/monitor_nginx.sh"  
                interval 2  
                weight 2  
}   
vrrp_instance VI_1 {   
        state MASTER  
        interface eth0  
        virtual_router_id 51  
        priority 101  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.200  
        }   
} 
 {% endhighlight %} 

 * 下面是监控nginx的脚本
{% highlight sh %} 
[root@test1 tmp]# cat monitor_nginx.sh  
#!/bin/bash   
A=`ps -C nginx --no-header |wc -l`  
if [ $A -eq 0 ];then   
                /usr/local/nginx/sbin/nginx  
                sleep 3  
                if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then  
                       killall keepalived  
                fi  
fi  
[root@test1 tmp]#service keepalived restart  
BACKUP端的
[root@test2 src]# wget http://www.keepalived.org/software/keepalived-1.2.2.tar.gz  
[root@test2 src]# tar zxvf keepalived-1.2.2.tar.gz  
[root@test2 src]# cd keepalived-1.2.2  
[root@test2 keepalived-1.2.2]# ./configure --prefix=/usr/local/keepalived  
[root@test2 keepalived-1.2.2]# make  
[root@test2 keepalived-1.2.2]# make install  
[root@test2 keepalived-1.2.2]# cp /usr/local/keepalived/sbin/keepalived /usr/sbin/  
[root@test2 keepalived-1.2.2]# cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/  
[root@test2 keepalived-1.2.2]# cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  
[root@test2 keepalived-1.2.2]# mkdir /etc/keepalived  
[root@test2 keepalived-1.2.2]# vim /etc/keepalived/keepalived.conf  
[root@test2 tmp]# chkconfig --add keepalived  
[root@test2 tmp]# chmod 755 /etc/init.d/keepalived   
[root@test2 tmp]# chkconfig keepalived on  
[root@test2 tmp]#vim /etc/keepalived/keepalived.conf  
global_defs {  
 notification_email {  
 denglei@ctfo.com  
}  
notification_email_from root@test2.com  
smtp_server 127.0.0.1  
smtp_connect_timeout 30  
router_id LVS_DEVEL  
}  
vrrp_script chk_http_port {   
                script "/tmp/monitor_nginx.sh"  
                interval 2  
                weight 2  
}   
vrrp_instance VI_1 {   
        stateBACKUP  
        interface eth0  
        virtual_router_id 51  
        priority 100  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.200  
        }   
}  
[root@test2 tmp]# cat monitor_nginx.sh  
#!/bin/bash   
A=`ps -C nginx --no-header |wc -l`  
if [ $A -eq 0 ];then   
                /usr/local/nginx/sbin/nginx  
                sleep 3  
                if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then  
                       killall keepalived  
                fi  
fi  
 
[root@test2 tmp]#service keepalived restart  
 {% endhighlight %}

现在test1与test2的keepalived都安装完成，是单主模式的负载均衡，由于我想把网站做成负载均衡双主模式，所以我在test1与test2的keepalived.conf里做了修改：
 
 * test1里的
{% highlight sh %}
global_defs {  
 notification_email {  
 denglei@ctfo.com  
}  
notification_email_from root@test1.com  
smtp_server 127.0.0.1  
smtp_connect_timeout 30  
router_id LVS_DEVEL  
}  
vrrp_script chk_http_port {   
                script "/tmp/monitor_nginx.sh"  
                interval 2  
                weight 2  
}   
vrrp_instance VI_1 {   
        state MASTER  
        interface eth0  
        virtual_router_id 51  
        priority 101  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.200  
        }   
}  
vrrp_instance VI_2 {   
        state BACKUP  
        interface eth0  
        virtual_router_id 52  
        priority 90  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.201  
        }   
} 
{% endhighlight %} 
 * test2里的
{% highlight sh %}
global_defs {  
 notification_email {  
 denglei@ctfo.com  
}  
notification_email_from root@test1.com  
smtp_server 127.0.0.1  
smtp_connect_timeout 30  
router_id LVS_DEVEL  
}  
vrrp_script chk_http_port {   
                script "/tmp/monitor_nginx.sh"  
                interval 2  
                weight 2  
}   
vrrp_instance VI_1 {   
        state BACKUP  
        interface eth0  
        virtual_router_id 51  
        priority 90  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.200  
        }   
}  
vrrp_instance VI_2 {   
        state MASTER  
        interface eth0  
        virtual_router_id 52  
        priority 101  
        authentication {   
                     auth_type PASS  
                     auth_pass eric  
        }   
        track_script {   
                chk_http_port  
        }   
        virtual_ipaddress {   
             10.1.88.201  
        }   
}
{% endhighlight %}  
这样主主模式的负载均衡就部署完成

###二. web层的部署

1. 安装nginx+php(fastcgi)

可以使用源码安装，也可以使用我的lnmp脚本进行安装

 * 先下载模块化安装lnmp脚本
{% highlight sh %}  
[root@test4 tmp]# wget http://202.96.42.117/soft/install_lnmp.tar.gz
{% endhighlight %}   
 * 然后解压
{% highlight sh %}  
[root@test4 tmp]# tar zxf install_lnmp.tar.gz 
{% endhighlight %}  
 * 然后安装nginx
{% highlight sh %}  
[root@test4 tmp]# sh install_lnmp.sh install_nginx 
{% endhighlight %}   
 * 安装完成再进行php的安装
{% highlight sh %}  
[root@test4 tmp]# sh install_lnmp.sh install_php 
{% endhighlight %}  
 * 都安装完成之后，进行检查
{% highlight sh %}  
 [root@test4 tmp]#  sh install_lnmp.sh install_check  
Sat Apr  7 04:48:05 MDT 2012 Start install!  
========================== Check install ================================  
/usr/local/nginx [found]  
/usr/local/php  [found]  
Error: /usr/local/mysql not found!!!  
========================== Check install ================================  
Sorry,Failed to install LNMP!  
Please check errors and logs.  
Sat Apr  7 04:55:05 MDT 2012 Finish install!  
Total runtime: 0 Seconds  
{% endhighlight %}  
可以看到nginx与php都安装完成

##现在开始安装数据库层

###三. 数据库层

安装drbd

1.修改hosts文件

 * test6的
{% highlight sh %}  
[root@test6 /]# cat /etc/hosts  
# Do not remove the following line, or various programs  
# that require network functionality will fail.  
127.0.0.1   localhost.localdomain localhost  
::1     localhost6.localdomain6 localhost6  
10.1.88.175 test6  
10.1.88.179 test7  
{% endhighlight %} 
 
 * test7的
{% highlight sh %}  
[root@test7 /]# cat /etc/hosts  
# Do not remove the following line, or various programs  
# that require network functionality will fail.  
127.0.0.1   localhost.localdomain localhost  
::1     localhost6.localdomain6 localhost6  
10.1.88.175 test6  
10.1.88.179 test7  

[root@test6 ~]# yum install -y drbd83 kmod-drbd83 heartbeat*  
[root@test6 ~]# rpm -qa|grep heartbeat  
heartbeat-pils-2.1.3-3.el5.centos  
heartbeat-stonith-2.1.3-3.el5.centos  
heartbeat-gui-2.1.3-3.el5.centos  
heartbeat-devel-2.1.3-3.el5.centos  
heartbeat-ldirectord-2.1.3-3.el5.centos  
 {% endhighlight %} 
 
 * 加载模块
{% highlight sh %}  
[root@test6 ~]# modprobe drbd 
{% endhighlight %}  
 * 在test6与test7都查看是否加载成功
{% highlight sh %}  
[root@test6~]# lsmod |grep drbd  
drbd                  298760  3  
[root@test7~]# lsmod |grep drbd  
drbd                  298760  5  
 
[root@test6/]# cat /etc/drbd.conf  
#  
# please have a a look at the example configuration file in  
# /usr/share/doc/drbd83/drbd.conf  
#  
 
#  
# please have a a look at the example configuration file in  
# /usr/share/doc/drbd83/drbd.conf  
global {  
    # minor-count 64;  
    # dialog-refresh 5; # 5 seconds  
    # disable-ip-verification;  
usage-count no;     
}  
 
common {  
  syncer { rate 100M; }  
}  
 
resource db {  
  protocol      C;  
  handlers {  
    pri-on-incon-degr "echo o > /proc/sysrq-trigger ; halt -f";  
    pri-lost-after-sb "echo o > /proc/sysrq-trigger ; halt -f";  
    local-io-error "echo o > /proc/sysrq-trigger ; halt -f";  
    fence-peer "/usr/lib64/heartbeat/drbd-peer-outdater -t 5";  
    pri-lost "echo pri-lost. Have a look at the log files. | mail -s 'DRBD Alert' denglei@ctfo.com";  
    split-brain "/usr/lib/drbd/notify-split-brain.sh denglei@ctfo.com";  
    out-of-sync "/usr/lib/drbd/notify-out-of-sync.sh denglei@ctfo.com";  
  }  
 
  net {  
    # timeout           60;  
    # connect-int       10;  
    # ping-int          10;  
    # max-buffers     2048;  
    # max-epoch-size  2048;  
    cram-hmac-alg "sha1";  
shared-secret "MySQL-HA";  
  }  
 
  disk {  
    on-io-error detach;  
fencing resource-only;  
  }  
 
  startup {  
    wfc-timeout 120;  
    degr-wfc-timeout 120;  
  }  
 
  device        /dev/drbd1;  
   
  on test6 {  
disk        /dev/sda2;  
address     10.1.88.175:7788;  
    meta-disk   internal;  
  }  
  on test7 {  
disk        /dev/sda6;  
address     10.1.88.179:7788;  
    meta-disk   internal;  
  }  
 }
{% endhighlight %}
  
 * 把配置文件另传给test7一份
{% highlight sh %}  
[root@test6 ~]# scp /etc/drbd.conf 10.1.88.179:/etc/  
root@10.1.88.179's password:   
drbd.conf                                                                 100%  687     0.7KB/s   00:00 
{% endhighlight %}     
 * 初始化meta-data area：
{% highlight sh %}    
[root@test6 ~]# drbdadm create-md db  
md_offset 2730786816  
al_offset 2730754048  
bm_offset 2730668032  
 
Found ext3 filesystem  
     2666788 kB data area apparently used  
     2666668 kB left usable by current configuration  
 
Device size would be truncated, which  
would corrupt data and result in  
'access beyond end of device' errors.  
You need to either  
   * use external meta data (recommended)  
   * shrink that filesystem first  
   * zero out the device (destroy the filesystem)  
Operation refused.  
 
Command 'drbdmeta 1 v08 /dev/sda2 internal create-md' terminated with exit code 40  
drbdadm create-md db: exited with code 40 
{% endhighlight %}  
 
 * 出现这个问题使用 dd 指令将一些资料塞到 /dev/sda2后再执行 drbdadm create-md db（db是drbd.conf里设置的资源名称） 指令即可顺利执行
{% highlight sh %}  
[root@test6 ~]# drbdadm create-md db  
Writing meta data...  
initializing activity log  
NOT initialized bitmap  
New drbd meta data block successfully created.  
success  
{% endhighlight %}  
 * 成功的完成了初始化meta-data area，在test7里进行初始化
{% highlight sh %}  
[root@test7 /]# drbdadm create-md db  
  --==  Thank you for participating in the global usage survey  ==--  
The server's response is:  
 
you are the 2845th user to install this version  
md_offset 2730754048  
al_offset 2730721280  
bm_offset 2730635264  
 
Found some data 
 
 ==> This might destroy existing data! <==  
 
Do you want to proceed?  
[need to type 'yes' to confirm] yes  
 
You want me to create a v08 style flexible-size internal meta data block.  
There appears to be a v08 flexible-size internal meta data block  
already in place on /dev/sda6 at byte offset 2730754048  
Do you really want to overwrite the existing v08 meta-data?  
[need to type 'yes' to confirm] yes  
 
Writing meta data...  
initializing activity log  
NOT initialized bitmap  
New drbd meta data block successfully created.  
{% endhighlight %}  

 * 然后在test6与test7里都启动drbd
{% highlight sh %}  
[root@test6 ~]# /etc/init.d/drbd start  
Starting DRBD resources: [ d(db) s(db) n(db) ]........  
[root@test7 /]# /etc/init.d/drbd start  
Starting DRBD resources: [ d(db) s(db) n(db) ].
{% endhighlight %}    
 * 通过端口查看drbd是否启动
{% highlight sh %}  
[root@test6 ~]# netstat -antl|grep 7789  
tcp        0      0 10.1.88.175:7789            10.1.88.179:50990           ESTABLISHED   
tcp        0      0 10.1.88.175:40323           10.1.88.179:7789            ESTABLISHED  
[root@test7 /]# netstat -antl|grep 7789  
tcp        0      0 10.1.88.179:7789            10.1.88.175:40323           ESTABLISHED   
tcp        0      0 10.1.88.179:50990           10.1.88.175:7789            ESTABLISHED 
{% endhighlight %}   
 * 查看test6与test7的drbd状态
{% highlight sh %}  
[root@test6 ~]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----  
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:2666636  
[root@test7 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
1: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----  
    ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:2666636  
{% endhighlight %}  

对输出的含义解释如下：

ro： 表示角色信息，第一次启动drbd时，两个drbd节点默认都处于Secondary状态.

ds： 是磁盘状态信息，“Inconsistent/Inconsisten”，即为“不一致/不一致”状态，表示两个节点的磁盘数据处于不一致状态.

Ns： 表示网络发送的数据包信息.

Dw： 是磁盘写信息.

Dr： 是磁盘读信息.

 * 途中显示了drbd两台主机都是"备机"状态.DRBD无法判断哪一方为主机,
以哪一方的磁盘数据作为标准数据.所以,我们需要初始化，比如我选择test6作为主机
{% highlight sh %}
[root@test6 ~]# drbdsetup /dev/drbd1 primary -o 
{% endhighlight %}

 * 此命令只在主机里输入
然后在查看test6与test7的drbd状态
{% highlight sh %}
[root@test6 ~]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----  
    ns:29696 nr:0 dw:0 dr:29696 al:0 bm:1 lo:0 pe:7 ua:0 ap:0 ep:1 wo:b oos:2637836  
    [>....................] sync'ed:  1.3% (2637836/2666636)K  
    finish: 0:03:00 speed: 14,400 (14,400) K/sec  
[root@test7 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
1: cs:SyncTarget ro:Secondary/Primary ds:Inconsistent/UpToDate C r-----  
    ns:0 nr:1328128 dw:1328128 dr:0 al:0 bm:81 lo:0 pe:8 ua:0 ap:0 ep:1 wo:b oos:1338508  
    [=========>..........] sync'ed: 50.0% (1338508/2666636)K  
    finish: 0:02:10 speed: 10,224 (10,216) want: 10,240 K/sec 
{% endhighlight %} 

从test6的drbd状态看出：“ro状态现在变为“Primary/Secondary”，“ds”状态也变为“UpToDate/Inconsistent”，也就是“实时/不一致”状态，现在数据正在主备两个主机的磁盘间进行同步，且同步进度为1.3%，同步速度每秒14M左右。

 * 等一会在查看drbd的状态：
{% highlight sh %}
[root@test6 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----  
    ns:2666636 nr:0 dw:0 dr:2666636 al:0 bm:163 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0  
[root@test7 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
1: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----  
    ns:0 nr:2666636 dw:2666636 dr:0 al:0 bm:163 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0 
{% endhighlight %} 
从test6与test7的drbd输出可知，磁盘状态都是"实时",表示数据同步完成了.现在把主机的drbd挂载到一个目录上进行使用.备机的DRBD设备无法被挂载,因为它是用来接收主机数据的,由drbd负责操作，也就是说如果test6为主机时，可以用/dev/drbd1挂载到一个目录，然后你可以进入那个目录里进行各种操作，test7为备机，那么test7就不能把/dev/drbd1挂载到一个目录，必须等得test7为主机的时候，才能把/dev/drbd1挂载到一个目录。

 * 在挂载/dev/drbd1的时候，需要先给/dev/drbd1格式化文件系统
{% highlight sh %}
[root@test6 /]# mkfs.ext3 /dev/drbd1  
mke2fs 1.39 (29-May-2006)  
Filesystem label=  
OS type: Linux  
Block size=4096 (log=2)  
Fragment size=4096 (log=2)  
333984 inodes, 666659 blocks  
33332 blocks (5.00%) reserved for the super user  
First data block=0 
Maximum filesystem blocks=683671552 
21 block groups  
32768 blocks per group, 32768 fragments per group  
15904 inodes per group  
Superblock backups stored on blocks:   
    32768, 98304, 163840, 229376, 294912  
 
Writing inode tables: done                              
Creating journal (16384 blocks): done  
Writing superblocks and filesystem accounting information: done  
 
This filesystem will be automatically checked every 35 mounts or  
180 days, whichever comes first.  Use tune2fs -c or -i to override.
{% endhighlight %}
  
 * 然后把/dev/drbd1挂载到/mnt目录下（这个目录可以随便指定）
{% highlight sh %}
[root@test6 /]# mount /dev/drbd1 /mnt 
{% endhighlight %}
 * 然后进入目录查看内容
{% highlight sh %}
[root@test6 /]# cd /mnt/  
[root@test6 mnt]# ll  
total 16  
drwx------ 2 root root 16384 Mar  8 22:28 lost+found 
{% endhighlight %} 
 * 然后我们可以在mnt里创建个文件，看看是否能保存到备机test7的drbd里
{% highlight sh %}
[root@test6 mnt]# touch create_by_test6  
[root@test6 mnt]# ll  
total 16  
-rw-r--r-- 1 root root     0 Mar  8 22:32 create_by_test6  
drwx------ 2 root root 16384 Mar  8 22:28 lost+found  
{% endhighlight %}
 * 然后到备机test7里查看是否有这个文件，这个操作也是drbd的主备机切换,你需要将drbd的主备机互换一下.如果不执行这个操作，你到备机test7里查看会发现/mnt目录里什么都没有
{% highlight sh %}
[root@test7 /]# cd /mnt/  
[root@test7 mnt]# ll  
total 0 
{% endhighlight %} 
 * 在主机test6上,先要卸载掉drbd设备，或者卸载/dev/drbd1挂载的/mnt目录

 * 需要先退出mnt目录，然后在执行卸载，否则会出现umount: /mnt: 
{% highlight sh %}
device is busy
[root@test6 mnt]# cd ..  
[root@test6 /]# umount /mnt 
{% endhighlight %} 
 * 然后使用drbdadm secondary db把主机变为备机
{% highlight sh %}
[root@test6 /]# drbdadm secondary db  
[root@test6 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:Connected ro:Secondary/Secondary ds:UpToDate/UpToDate C r-----  
    ns:2775056 nr:0 dw:108420 dr:2666773 al:38 bm:163 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0  
{% endhighlight %}

 * 现在test6变成了备机，我们接下来到test7里操作，使用drbdadm primary db，使test7变成主机
{% highlight sh %}
[root@test7 /]# drbdadm primary db  
[root@test7 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
1: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----  
    ns:0 nr:2775056 dw:2775056 dr:0 al:0 bm:163 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0  
{% endhighlight %}
 * 从test7的drbd输出可以看到，test7成为了主机
然后把drbd挂载到mnt目录下，查看是否有test6里建立的文件
{% highlight sh %}
[root@test7 /]# mount /dev/drbd1 /mnt  
[root@test7 /]# cd /mnt  
[root@test7 mnt]# ll  
total 16  
-rw-r--r-- 1 root root     0 Mar  8 22:32 create_by_test6  
drwx------ 2 root root 16384 Mar  8 22:28 lost+found 
{% endhighlight %} 
可以看出，test7成为主机时，收到了之前主机test6建立的文件夹。

现在drbd安装完成，我们继续安装heartbeat与mysql
 
2．安装heartbeat与mysql

前文已经yum安装了heartbeat所以现在就不安装了

 * 查看heartbeat的配置文件ha.cf
{% highlight sh %}
[root@test6 ~]# grep -v "^#" /etc/ha.d/ha.cf   
debugfile /var/log/ha-debug                  #错误的日志  
logfile /var/log/ha-log                      #日志  
logfacility local0                           #这个是设置heartbeat的日志，这里是用的系统日志  
keepalive 2                                  #心跳的频率  
deadtime 10                                  #死亡时间，如果其他节点10s回应，则认为死亡  
warntime 5                                   #如果死亡之后，5s还没有连接则把警告信息写入日志里  
initdead 120                                 #在其他节点死掉之后，系统启动前需要等待的时间，一般为deadtime的两倍  
udpport 694                                  #用udp协议的694端口通信  
ucast eth0 10.1.88.179                       #另外一个节点的ip  
auto_failback off                            #设置当死亡节点
恢复正常之后是否重新启用；容易发生数据不一致的情况，必须项，不然后面hb_standby命令无法使用；  
node    test6                                #节点名（通过uname -n查询）  
node    test7                                #节点名（通过uname -n查询）  
ping 10.1.88.254                             #ping网关查看网络情况（当网络或者heartbeat失效是使用）  
respawn hacluster /usr/lib/heartbeat/ipfail  #这里是配置ip绑定和切换的功能， ipfail就是控制ip切换的程序  
 
apiauth ipfail gid=haclient uid=hacluster    #控制ip切换的时候所使用的用户 
{% endhighlight %}
 
 * 设置节点之间的通信密钥
{% highlight sh %}
[root@test6 ~]# grep -v "^#" /etc/ha.d/authkeys   
auth 1  
1 crc 
{% endhighlight %} 
 * 使用heartbeat的haresources来定义资源
{% highlight sh %}
[root@test6 ~]# grep -v "^#" /etc/ha.d/haresources   
 
test6 IPaddr::10.1.88.199/24/eth0:1 mysqld_umount mysqld
{% endhighlight %}  
解释：

1）test6做为默认的drbd的primary机器（如果想指定哪个主机为drbd的primary则这里输入那台主机的uname -n的名字）

2）Ipaddr为/etc/ha.d/resource.d目录里面的一个可执行文件。resource.d里面的所有文件都是可执行的，如果不行，执行下面的指令
{% highlight sh %}
#chmod 755 -R /etc/ha.d/resource.d
{% endhighlight %}
3) 10.1.88.199/24/eth0:1是指定的heartbeat的VIP

4) mysqld_umount 定义的脚本

5）mysqld:这个也是/etc/ha.d/resource.d目录下的内容，就是mysql的启动程序。如果没有，可以执行下面指令
{% highlight sh %}
#ln -s /etc/init.d/mysqld /etc/ha.d/resource.d/mysqld
 {% endhighlight %}

###三．安装mysql
可以使用源码安装，也可以使用yum安装，下面我使用的是源码安装（生产环境一定要是有最新稳定版的源码安装）

 * 先下载模块化安装lnmp脚本
{% highlight sh %}
[root@test6 tmp]# wget http://202.96.42.117/soft/install_lnmp.tar.gz 
{% endhighlight %}
 * 然后解压
{% highlight sh %}
[root@test6 tmp]# tar zxf install_lnmp.tar.gz
{% endhighlight %} 
 * 然后安装nginx
{% highlight sh %}
[root@test6 tmp]# sh install_lnmp.sh install_mysql 
{% endhighlight %}
 * test7也是一样
在启动heartbeat
{% highlight sh %}
[root@test6 /]# service heartbeat start  
logd is already running  
Starting High-Availability services:   
2012/03/14_21:36:50 INFO:  Resource is stopped  
                                                           [FAILED]  
heartbeat[6579]: 2012/03/14_21:36:50 ERROR: Client child command [/usr/lib/heartbeat/ipfail] is not executable  
heartbeat[6579]: 2012/03/14_21:36:50 ERROR: Heartbeat not started: configuration error.  
heartbeat[6579]: 2012/03/14_21:36:50 ERROR: Configuration error, heartbeat not started.
{% endhighlight %}  
 * 如果发生这个问题，先查看你的系统是32还是64位的，如果是64位的，则在ha.cf
里respawn hacluster /usr/lib/heartbeat/ipfail吧这个lib改成lib64；32位的不变。
启动之后，进入mysql，建立数据库db，然后建立表t，插入数据
{% highlight sh %}
[root@test6 resource.d]# mysql  
Welcome to the MySQL monitor.  Commands end with ; or \g.  
Your MySQL connection id is 2  
Server version: 5.0.95 Source distribution  
 
Copyright (c) 2000, 2011, Oracle and/or its affiliates. All rights reserved.  
 
Oracle is a registered trademark of Oracle Corporation and/or its  
affiliates. Other names may be trademarks of their respective  
owners.  
 
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.  
 
mysql> show databases;  
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |   
| 2051               |   
| lost+found         |   
| mysql              |   
| test               |   
+--------------------+  
5 rows in set (0.04 sec)  
 
mysql> create database db;  
Query OK, 1 row affected (0.01 sec)  
 
mysql> use db  
Database changed  
mysql> create table t (id int(10),name char(10));  
Query OK, 0 rows affected (0.05 sec)  
 
mysql> insert into t values(001,"test1"),(002,"test2");  
Query OK, 2 rows affected (0.00 sec)  
Records: 2  Duplicates: 0  Warnings: 0  
 
mysql> select * from t;  
+------+----------+  
| id   | name     |  
+------+----------+  
|    1 | test1|   
|    2 | test2 |   
+------+----------+  
2 rows in set (0.00 sec)  
 
mysql> exit  
Bye 
{% endhighlight %} 

 * 之后停止heartbeat，查看其它节点（test7）里是否有mysql的数据
{% highlight sh %}
[root@test6 /]# service heartbeat stop 
{% endhighlight %}
 * drbd已经变为从了，drbd1已经从database里卸载了
{% highlight sh %}
[root@test6 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----  
    ns:712 nr:316 dw:972 dr:6142 al:8 bm:3 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0  
[root@test6 /]# df -h  
Filesystem            Size  Used Avail Use% Mounted on  
/dev/sda3             375G  2.9G  353G   1% /  
/dev/sda1             122M   18M   98M  16% /boot  
tmpfs                 2.0G     0  2.0G   0% /dev/shm 
{% endhighlight %}
 
 * 在test7里查看drbd是否为主，drbd1是否装载了database
{% highlight sh %}
[root@test7 /]# cat /proc/drbd   
version: 8.3.12 (api:88/proto:86-96)  
GIT-hash: e2a8ef4656be026bbae540305fcb998a5991090f build by mockbuild@builder10.centos.org, 2012-01-28 13:52:25  
 
 1: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----  
    ns:316 nr:712 dw:1028 dr:3174 al:5 bm:3 lo:0 pe:0 ua:0 ap:0 ep:1 wo:b oos:0  
[root@test7 /]# df -h  
Filesystem            Size  Used Avail Use% Mounted on  
/dev/sda2             349G  3.9G  327G   2% /  
/dev/sda8             122M   18M   98M  16% /boot  
tmpfs                 2.0G     0  2.0G   0% /dev/shm  
/dev/drbd1            2.6G   89M  2.3G   4% /database 
{% endhighlight %}
 
从上面可以看到，test7已经变为主，drbd1已经挂载到了database了

 * 进入mysql里查看db数据库、t表是否已交传过来
{% highlight sh %}
[root@test7 /]# mysql  
Welcome to the MySQL monitor.  Commands end with ; or \g.  
Your MySQL connection id is 2  
Server version: 5.0.95 Source distribution  
 
Copyright (c) 2000, 2011, Oracle and/or its affiliates. All rights reserved.  
 
Oracle is a registered trademark of Oracle Corporation and/or its  
affiliates. Other names may be trademarks of their respective  
owners.  
 
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.  
 
mysql> show databases;  
+--------------------+  
| Database           |  
+--------------------+  
| information_schema |   
| 2051               |   
| db                 |   
| lost+found         |   
| mysql              |   
| test               |   
+--------------------+  
6 rows in set (0.04 sec)  
 
mysql> use db;  
Reading table information for completion of table and column names  
You can turn off this feature to get a quicker startup with -A  
 
Database changed  
mysql> select * from t;  
+------+----------+  
| id   | name     |  
+------+----------+  
|    1 | test1  |   
|    2 | test2 |   
+------+----------+  
2 rows in set (0.00 sec)  
{% endhighlight %}
可以看到mysql已经收到了数据。这样我们的drbd+heartbeat+mysql已经实现了高可用的mysql数据库了。








[lnmp-install]: http://dl528888.blog.51cto.com/2382721/816542 "悬停显示"