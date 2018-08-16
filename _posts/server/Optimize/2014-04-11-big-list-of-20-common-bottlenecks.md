---
layout: post
title: 影响系统性能的20个瓶颈
category: server
tags: Optimize@server
keywords: common bottlenecks
description: 
from: http://highscalability.com/blog/2012/5/16/big-list-of-20-common-bottlenecks.html

---

在现实中，众所周知，瓶颈是无穷无尽的而且涉及方方面面。

###数据库:
* 工作中数据大小超过可用内存 RAM
* 长短查询混合
* 写-写 冲突
* 大的联合查询占光内存
* 虚拟化:
* 共享 HDD 存储，磁盘寻道挂起
* 云平台中的网络 I/O 波动

###编程:
* 线程：死锁、相对于事件驱动来说过于重量级、调试、线程数与性能比非线性
* 事件驱动编程：回调的复杂性、函数调用中如何保存状态（how-to-store-state-in-function-calls）
* 缺少profile工具、缺少trace工具、缺少日志工具
* 单点故障、横向不可扩展
* 有状态的应用
* 搓设计：一台机器上能跑，几个用户也能跑，几个月后，几年后，尼玛，发现扛不住了，整个架构需要重写。
* 算法复杂度
* 依赖于诸如DNS查找等比较搞人的外部组件
* 栈空间

###磁盘:
* 本地磁盘存取
* 随机磁盘读写 -> 磁盘寻道
* 磁盘碎片化
* 写入超过SSD容量的数据导致SSD硬盘性能降低
* 操作系统：
* 内核缓冲刷入磁盘，填充linux缓冲区缓存
* TCP缓冲区过小
* 文件描述符限制
* 功率分配

###缓存:
* 不使用memcached
* HTTP中，header，etags，不压缩（headers, etags, not gzipping）
* 没有充分使用浏览器缓存功能
* 字节码缓存（如PHP）
* L1/L2缓存. 这是个很大的瓶颈. 把频繁使用的数据保持在L1/L2中. 设计到的方面很多：网络数据压缩后再发送，基于列压缩的DB中不解压直接计算等等。有TLB友好的算法。最重要的是牢固掌握以下基础知识：多核CPU、 L1/L2，共享L3，NUMA内存，CPU、内存之间的数据传输带宽延迟，磁盘页缓存，脏页，TCP从CPU到DRAM到网卡的流程。

###CPU:
* CPU 过载
* 上下文切换 -> 一个内核上跑了太多的线程，linux调度对于应用来说很不友好, 太多的系统调用, 等等...
* IO 等待 -> 所有的CPU都挂起等待比较慢的IO
* CPU 缓存: 缓存数据是一个为了平衡不同实例有不同的值和繁重的同步缓存数据保持一致，而精心设计的一个进程。
* 背板吞吐量
* 
###网络:
* 网卡的最大输出带宽，IRQ达到饱和状态，软件中断占用了100%的CPU
* DNS查找
* 丢包
* 网络路由瞎指挥
* 网络磁盘访问
* 共享SAN（Storage Area Network)
* 服务器失败 -> 服务器无响应

###过程:
* 测试时间 Testing time
* 开发时间 Development time
* 团队人数 Team size
* 预算 Budget
* 代码缺陷 Code debt

###内存:
* 内存溢出 -> 杀进程，进入 swap ，越来越慢
* 内存溢出导致磁盘频繁读写（swap相关）
* 内存库开销
* 内存碎片
* Java 需要垃圾收集导致程序暂停
* C 语言的 malloc 无法分配 