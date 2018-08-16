---
layout: post
title: LIKE查询索引与优化
category: mysql
tags: Optimize@mysql
keywords: mysql like index optimize
description: 
from: http://blogread.cn/it/article/5333

---

1. `like %keyword`索引失效，使用全表扫描。但可以通过翻转函数+like前模糊查询+建立翻转函数索引=走翻转函数索引，不走全表扫描。
2. `like keyword%`索引有效。
3. `like %keyword%`索引失效，也无法使用反向索引。

****
###1. 查询%xx%的记录
**使用[LOCATE(substr,str)][locate] [POSITION(substr IN str)][position]**   
返回子串substr在字符串str第一个出现的位置，如果substr不是在str里面，返回0。

如果出现的位置〉0，表示包含该字符串。查询效率比like要高。

如果： `table.field like ‘%AAA%’` 可以改为 `locate (‘AAA’ , table.field) > 0`


**使用[instr][instr]**

    SELECT COUNT(*) FROM t WHERE INSTR(t.column,’xx’)> 0

这种查询效果很好，速度很快。

###2. 查询%xx的记录

    SELECT COUNT(T2.name) as COUNT FROM T1 , T2  WHERE T1.id = T2.id AND T1.name LIKE ’%245′`

在执行的时候，执行计划显示，消耗值，io值，cpu值均非常大，原因是like后面前模糊查询导致索引失效，进行全表扫描

解决方法：这种只有前模糊的sql可以改造如下写法

    SELECT COUNT(T2.name) as COUNT from T1,T2 where T1.id = T2.id and REVERSE(T1.name) like REVERSE(‘%245′)

使用翻转函数+like前模糊查询+建立翻转函数索引=走翻转函数索引，不走全扫描。有效降低消耗值，io值，cpu值这三个指标，尤其是io值的降低。


[locate]: http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_locate
[position]: http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_position
[instr]: http://dev.mysql.com/doc/refman/5.0/en/string-functions.html#function_instr