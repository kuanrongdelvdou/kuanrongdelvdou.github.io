---
layout: post
title: linux下定时备份数据库
category: mysql
tags: Admin@mysql
keywords: database backup cron
description: 
from: 

---
主要是应用linux下crontab定时执行小本实现数据库定时备份的功能。

##1.数据库备份脚本everyday\_db\_backup.sh

    #!/bin/sh
    #dump数据并打包
    mysqldump -uroot -p123456 traffic --quick --single-transaction --ignore-table=traffic.no_need_backup >  /opt/db-backup/db-`date +%Y-%m-%d`.backup
    tar -zcf /opt/db-backup/db-`date +%Y-%m-%d`.backup.tar.gz /opt/db-backup/db-`date +%Y-%m-%d`.backup
    rm -f /opt/db-backup/db-`date +%Y-%m-%d`.backup
    #传送到备份服务器
    scp /opt/db-backup/db-`date +%Y-%m-%d`.backup.tar.gz root@192.168.1.3:/opt/online-backup/database

##2.crontab设置
修改cron配置`crontab -e`

    \#每天2点定时备份数据库
    0 2 * * * /opt/scripts/everyday_db_backup.sh