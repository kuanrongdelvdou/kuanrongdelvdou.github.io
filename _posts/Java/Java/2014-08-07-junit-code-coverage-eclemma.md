---
layout: post
title: Junit测试覆盖率检测工具EclEmma
category: java
tags: Java@java
keywords: java junit 覆盖率
description: 
from: 

---
最近在写一个项目，需要Junit覆盖尽量多的逻辑，于是想到找一个覆盖率检测工具，通过搜索找到了2个工具：EclEmma和clover。对比发现clover用起来太麻烦了，需要引入jar包，还要改ant的build文件，太繁琐。

EclEmma基于Java覆盖测试工具 Emma。Emma 是一个在SourceForge上进行的开源项目。从某种程度上说，EclEmma 可以看作是 Emma 的一个图形界面。EclEmma安装、使用十分方便，在Eclipse Marketplace搜索EclEmma即可：    
![eclipse marketplace中安装EclEmma](/public/upload/java/EclEmma_Marketplace.jpg)

安装后，增加了一个按钮：    
![EclEmma按钮](/public/upload/java/EclEmma_Button.jpg)

使用时，选中你的项目，点击EclEmma右侧下下拉按钮，'Coverage As'-'Junit Test'
junit测试就会运行，运行结束后，Console未知会出现以下覆盖率检测结果：    
![EclEmma类覆盖率](/public/upload/java/EclEmma_Console.png)

点击进入代码，会用三种颜色表示代码是否被覆盖，绿色的行表示该行代码被完整的执行，红色部分表示该行代码根本没有被执行，而黄色的行表明该行代码部分被执行。黄色的行通常出现在单行代码包含分支的情况。    
![EclEmma代码覆盖演示](/public/upload/java/EclEmma_Code_Coverage.png)