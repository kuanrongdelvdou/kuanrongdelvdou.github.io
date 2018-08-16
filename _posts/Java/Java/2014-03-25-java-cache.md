---
layout: post
title: 超高速缓存的最佳实践 
category: java
tags: Java@java
keywords: java cache 缓存
description: 
from: http://www.importnew.com/10278.html

---
定制高速缓存解决方案是一件非常有趣的事情，它似乎是改善应用程序整体性能的最简单的方式。然而，超高速缓存是一项很大的技术难题，在实践之前需要注意几个事项。

最佳范例
##1、key/value集合并不是缓存
几乎我做过的所有项目都用到了一些定制高速缓存解决方案，这些方案都是使用的Java Maps。然而Map并不是缓存的解决方案，因为可能缓存超出了一个key/value的存储容量。缓存还需要满足以下特点：

* `驱逐策略(eviction policies)`
* `最大容量限制(max size limit)`
* `持久性存储(persistent store)`
* `弱引用建(weak references keys)`
* `统计(statistics)`
Java Map并不能提供上述的特点，你也不应该花费你客户的钱去定制缓存方案。你应该选择一个更好的缓存技术，比如 EHCache 或 Guava Cache，这两种缓存技术都是非常强大的，而且用起来也非常的简单。这些工具经常被一些项目用来测试，所以，代码质量相比与其他的定制方案更加优秀。

##2、使用一个缓存抽象层
Spring提供的缓存抽象层是一套非常灵活的方案。@Cacheable注解可以将业务逻辑层的代码从缓存横切关注点分离开来。缓存解决方案是可以通过配置文件进行配置的，所以它不会破坏业务层的方法。

##3、谨防缓存的开销
每一个api接口都是需要计算成本的，而缓存也不例外。如果你缓存一个web服务或者是一个开销比较大的数据库操作，那么这种开销可以忽略不计。如果在一个递归算法中使用本地缓存，那么就需要考虑缓存解决方案的开销了。甚至Spring的缓存抽象层都是有开销的，所以一定要确保收益大于成本。

##4、如果数据库查询操作非常慢，那么缓存可能是最后的解决方案了。
如果使用类似Hibernate的ORM工具，那么它就是首先要考虑进行优化的位置。确保抓取策略(fetching strategy)被正确的设计,这样才不会面临N+1的查询问题(N+1 query problems),你同样需要对SQL语句数进行断言操作(assert the SQL statement count，译者译：包括对增删改查操作的断言)，以验证ORM生成的查询语句是否存在问题。
当你对ORM生成的SQL语句进行优化之后，你需要再一次检查数据库查询速度是否还是那么慢。同时要确保所有的索引都用上了，这样你的SQL查询才会非常高效。索引必须要全部都放在内存中，不然就会浪费更多的SSD或者HDD硬盘空间了。
你的数据库是可以缓存查询结果的，一定要利用好这个特点。
如果数据集是很大的，并且数据增长的速度也是非常快的，那么你就需要按比例将这些数据分配到多个数据库碎片(shards)中。
如果这样都不能够解决你的问题，那么就考虑换一个更加优秀的缓存解决方案吧，比如Memcached。

##5、是否会影响到数据一致性呢？
当你在业务层之前使用缓存，数据一致性的约束是非常难做到的。如果缓存不能同步到数据库，那么ACID的特性就会受到影响。如果一个根实体改变了数据，那么将会影响到很大一部分的缓存。如果你抛弃缓存实体，那么所有由缓存带来的效益将会失去。如果你异步更新了缓存实体，就会影响到数据的一致性，最终一致)的数据模型也就不存在了。

##让我们来看看实例吧
受关于Java 8 computeIfAbsent Map这篇文章的启发，我写了一个Guava Cache缓存方案，有下面几个优点：
有一个固定大小的缓存（2条记录）
在jdk1.6下运行
{% highlight java %}
private LoadingCache<Integer, Integer> fibonacciCache = CacheBuilder.newBuilder()
	.maximumSize(2)
	.build(new CacheLoader<Integer, Integer>() {
		public Integer load(Integer i) {
			if (i == 0)
				return i;
			if (i == 1)
				return 1;
			LOGGER.info("Calculating f(" + i + ")");
			return fibonacciCache.getUnchecked(i - 2) + fibonacciCache.getUnchecked(i - 1);
		}
	});  
@Test
public void test() {
	for (int i = 0; i < 10; i++) {
		LOGGER.info("f(" + i + ") = " + fibonacciCache.getUnchecked(i));
	}
}
{% endhighlight %}
输出为：
{% highlight sh %}
INFO  [main]: FibonacciGuavaCacheTest - f(0) = 0
INFO  [main]: FibonacciGuavaCacheTest - f(1) = 1
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(2)
INFO  [main]: FibonacciGuavaCacheTest - f(2) = 1
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(3)
INFO  [main]: FibonacciGuavaCacheTest - f(3) = 2
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(4)
INFO  [main]: FibonacciGuavaCacheTest - f(4) = 3
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(5)
INFO  [main]: FibonacciGuavaCacheTest - f(5) = 5
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(6)
INFO  [main]: FibonacciGuavaCacheTest - f(6) = 8
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(7)
INFO  [main]: FibonacciGuavaCacheTest - f(7) = 13
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(8)
INFO  [main]: FibonacciGuavaCacheTest - f(8) = 21
INFO  [main]: FibonacciGuavaCacheTest - Calculating f(9)
INFO  [main]: FibonacciGuavaCacheTest - f(9) = 34
{% endhighlight %}
完整代码可以在GitHub上获得。