# 1. jdk-endorsed
本项目是使用 java endorsed 技术，修改 jdk 的若干实现类，满足业务需要。每个模块都是可以单独部署的功能。把模块产生的 jar 文件，拷贝到 $JAVA_HOME/jre/lib/endorsed 目录里，重启相关服务。

## 1.1 java endorsed 技术简介
可以的简单理解为-Djava.endorsed.dirs指定的目录面放置的jar文件，将有覆盖系统API的功能。可以牵强的理解为，将自己修改后的API打入到JVM指定的启动API中，取而代之。但是能够覆盖的类是有限制的，其中不包括 java.lang 包中的类。
根据官方文档描述：如果不想添加-D参数，如果我们希望基于这个JDK下的都统一改变，那么我们可以将我们修改的jar放到 目录 $JAVA_HOME/jre/lib/endorsed

# 2 模块列表
## 2.1 default-thread-factory-trace

本模块用于辅助解决线程池泄露的问题。当出现大量线程池中的线程，但是不知道这些线程池是在什么地方创建的？如以下两个线程，使用了 defaultFactory。 defaultFactory 创建的线程名以 `pool` 开头， 159859 是线程池编号。thread 后面的编号是同一线程池中的线程编号。
```bash
"pool-159859-thread-2" #1921190 prio=5 os_prio=0 tid=0x00007f889829b800 nid=0x2ef6 waiting on condition [0x00007f85450ad000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00007f8c0bf468f0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

"pool-159859-thread-1" #1921187 prio=5 os_prio=0 tid=0x00007f889342c000 nid=0x2ef4 waiting on condition [0x00007f852a604000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00007f8c0bf468f0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

这两个线程在等待从队列里取任务。由于线程池没有 shutdown，所以一直等待。

## 2.2 线程池溢出的代码示例

```java
ExecutorService es = Executors.newFixedThreadPool(4);
es.submit(() -> System.out.println("thread pool leak"));
```
以上代码创建了一个线程池，有4个线程，提交了一个任务。但是没有执行 shutdown 或者 shutdownNow。这些线程都不会释放。

## 2.3 使用示例
本示例解决 HiveServer 线程溢出问题。HiveServer 中出现了大量线程，线程的名称都是 'pool' 开头，不知道哪个模块创建的。
JAVA_HOME=/opt/jdk1.8.0_211/

### 2.3.1 拷贝 default-thread-factory-trace-0.1.0.jar
```bash
cp default-thread-factory-trace-0.1.0.jar /opt/jdk1.8.0_211/jre/lib/endorsed/
```

### 2.3.2 重启 HiveServer2 

### 2.3.3 使用 jstack 观察新的线程的名称
可以发现，pool 开头的线程名，变成了类似以下的线程名。

`com.baidu.hive.ql.hooks.ThreadLeakQueryLiftTimeHook.beforeCompile` 是创建线程池的所在的方法。`ThreadLeakQueryLiftTimeHook.java:17` 是创建线程池的代码所在的文件名和行号。24 是线程池编号，thread 后面的数字代表线程池内线程的编号。
```bash
"com.baidu.hive.ql.hooks.ThreadLeakQueryLiftTimeHook.beforeCompile(ThreadLeakQueryLiftTimeHook.java:17)-24-thread-1" #221 prio=5 os_prio=0 tid=0x000000000255f000 nid=0x5228 waiting on condition [0x00007ff960fb9000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000f8080630> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
```
### 2.3.4 找到相关代码，进行修复