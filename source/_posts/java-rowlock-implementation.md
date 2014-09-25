title: java rowlock implementation
date: 2014-04-18 16:52:11
tags: java
description: "java 细粒度锁的实现"
toc: true
category: java
---

在编写多线程的Code时， 我们尽量将锁的粒度更小。本文尝试用ConcurrentHashMap和CountDownLatch减小锁的粒度。

<!-- more -->
例如Hbase的HRegion在进行Mutation的操作，只尝试去锁Rowkey，下面是一个简单的code， 描述这一行为。

其实也就是说对于相同Rowkey的修改操作，我们必须加lock。

```java
//store all the locks
ConcurrentMap<K, CountDownLatch> rowLocks = new ConcurrentHashMap<>();
/**
* lock current rowkey
* @param key
* @throws Exception 
*/
public void lock(String key) throws Exception {
     CountDownLatch countDownLatch = new CountDownLatch(1);
     while (true) {
        CountDownLatch lock = rowLocks.putIfAbsent(key, countDownLatch);
        if (lock == null) {
            //we already get the lock, so break;
            break;
        } else {
           //another request use the lock, we need to await.
            try {
                if (!lock.await(MAX_WAIT_TIMEOUT_SECS, TimeUnit.SECONDS)) {
                    //wait to timeout, so throw the exception
                    throw new Exception("one rowkey wait timeout, rowkey=" + key);
                }
            } catch (InterruptedException e) {
                logger.warn("InterruptedException by user.");
            }
        }
    }
}
/**
* release the lock
* @param key
*/
public void unlock(String key) {
    CountDownLatch lock = rowLocks.remove(key);
    if (lock == null) { 
       logger.error("unlock a not exist rowkey, is there some bugs ?"); 
    } else {
       //notify other request to use this rowkey
       lock.countDown();
    }
}
```

使用ConcurrentMap获取保存锁， 使用CountDownLatch可以相互令竞争者相互等待。 