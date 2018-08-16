---
layout: post
title: Java线程安全的计数器
category: java
tags: Java@java
keywords: java clone deep copy collection
description: 
from: http://www.importnew.com/10761.html

---
## int（Integer）是否线程安全？
今天突然想到了一个问题，在很多需要计数器的地方，我都是用的int类型做计数，需要增加计数的时候使用++或者+n操作。那么这种操作是否安全？int类型是否是线程安全的？于是我做了一个实验
{% highlight java %}
public class TestInt {
	static int count = 0;
	public static void main(String[] args) throws InterruptedException {
		for(int i=0;i<100;i++){
			Run r = new Run();
			new Thread(r).start();;
		}
		Thread.sleep(4000);
		System.out.println(count);
	}
}
public class Run implements Runnable{
	public void run() {
		for (int i = 0; i < 100; i++) {
			TestInt.count++;
		}
	}
}
{% endhighlight %}

理论上，最后打印出的结果应该是100*100=10000,但是我多次测试结果如下：  
9907，9999，9861，9929，9800........  
每次都不到10000，说明int类型并不是线程安全的。

##那么应该如何解决这个问题？

###方法1：对int进行封装，封装一个线程安全的方法进行操作。
{% highlight java %}
public class TestInt {
	public static void main(String[] args) throws InterruptedException {
		for(int i=0;i<100;i++){
			Run r = new Run();
			new Thread(r).start();;
		}
		Thread.sleep(4000);
		System.out.println(ThreadSafeCounter.getcounter());
	}
}
public class Run implements Runnable{
	public void run() {
		for (int i = 0; i < 100; i++) {
			ThreadSafeCounter.add(1);
		}
	}
}
public class ThreadSafeCounter {
	private static int counter = 0;
	public synchronized static void add(int i) {
		counter += i;
	}
	public static int getcounter(){
		return counter;
	}
}
{% endhighlight %}
测试结果如下：
10000，1000，10000.......

###方法2：使用AtomicInteger
{% highlight java %}
public class TestInt {
	public static void main(String[] args) throws InterruptedException {
		for(int i=0;i<100;i++){
			Run r = new Run();
			new Thread(r).start();;
		}
		Thread.sleep(4000);
		System.out.println(AtomicCounter.getCount());
	}
}
public class Run implements Runnable {
	public void run() {
		for (int i = 0; i < 100; i++) {
			AtomicCounter.add(1);
		}
	}
}
public class AtomicCounter {
	static AtomicInteger counter = new AtomicInteger();
	public static void add(int i) {
		while (true) {
			int oldValue = counter.get();
			int newValue = oldValue + i;
			if (counter.compareAndSet(oldValue, newValue))
				return;
		}
	}
	public static int getCount() {
		return counter.get();
	}
}
{% endhighlight %}
测试结果如下：
10000，1000，10000.......

##性能
我又测试了以上两种方式的性能。在任意数量线程下进行计数操作。代码就不贴了。  
效率对比结果如下：  
<table class="table table-bordered table-condensed">
  <tr>
    <td>线程数</td>
    <td>ThreadSafeCounter</td>
    <td>AtomicCounter</td>
  </tr>
  <tr>
    <td>1线程</td>
    <td>10线程</td>
    <td>100线程</td>
  </tr>
  <tr>
    <td>1</td>
    <td>1</td>
    <td>1</td>
  </tr>
  <tr>
    <td>1.71</td>
    <td>1.26</td>
    <td>1.34</td>
  </tr>
</table>

可见synchronized消耗的性能大于AtomicInteger中的[CAS][cas]  
稍后有空补充一个CAS的介绍
[cas]:http://www.360doc.com/content/11/0914/16/7656248_148221200.shtml