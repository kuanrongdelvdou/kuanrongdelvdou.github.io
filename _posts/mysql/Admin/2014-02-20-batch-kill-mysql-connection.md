---
layout: post
title: 批量杀死MySQL连接的几种方法
category: mysql
tags: Admin@mysql
keywords: batch kill mysql connection
description: 
from: http://automaticthoughts.iteye.com/blog/1612388

---
##方法一
通过information_schema.processlist表中的连接信息生成需要处理掉的MySQL连接的语句临时文件，然后执行临时文件中生成的指令。

    mysql> select concat('KILL ',id,';') from information_schema.processlist where user='root';
    +------------------------+
    | concat('KILL ',id,';') |
    +------------------------+
    | KILL 3101;             |
    | KILL 2946;             |
    +------------------------+
    2 rows in set (0.00 sec)
 
    mysql>select concat('KILL ',id,';') from information_schema.processlist where user='root' into outfile '/tmp/a.txt';
    Query OK, 2 rows affected (0.00 sec)
 
    mysql>source /tmp/a.txt;
    Query OK, 0 rows affected (0.00 sec)

##方法二
杀掉当前所有的MySQL连接

    mysqladmin -uroot -p processlist|awk -F "|" '{print $2}'|xargs -n 1 mysqladmin -uroot -p kill
杀掉指定用户运行的连接，这里为Mike

    mysqladmin -uroot -p processlist|awk -F "|" '{if($3 == "Mike")print $2}'|xargs -n 1 mysqladmin -uroot -p kill
##方法三
通过SHEL脚本实现

    #杀掉锁定的MySQL连接
    for id in `mysqladmin processlist|grep -i locked|awk '{print $1}'`
    do
        mysqladmin kill ${id}
    done

##方法四
通过Maatkit工具集中提供的mk-kill命令进行

    #杀掉超过60秒的sql
    mk-kill -busy-time 60 -kill
    #如果你想先不杀，先看看有哪些sql运行超过60秒
    mk-kill -busy-time 60 -print
    #如果你想杀掉，同时输出杀掉了哪些进程
    mk-kill -busy-time 60 -print –kill

mk-kill更多用法可参考：    
[http://www.maatkit.org/doc/mk-kill.html](http://www.maatkit.org/doc/mk-kill.html)    
[http://www.sbear.cn/archives/426](http://www.sbear.cn/archives/426)    
Maatkit工具集的其它用法可参考：    
[http://code.google.com/p/maatkit/wiki/TableOfContents?tm=6](http://code.google.com/p/maatkit/wiki/TableOfContents?tm=6)    
参考文档：    
[kill-mysql-connectio-in-batch](http://www.orczhou.com/index.php/2010/10/kill-mysql-connectio-in-batch/)    
[mass-killing-of-mysql-connections](http://www.mysqlperformanceblog.com/2009/05/21/mass-killing-of-mysql-connections/)