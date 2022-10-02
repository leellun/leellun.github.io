---
title: aqs之Java8 CountDownLatch源码分析
date: 2020-10-13 20:14:02
categories:
  - 后端
  - java
tags:
  - aqs
  - 源码
---

## 一、介绍

一种同步辅助，允许一个或多个线程等待其他线程中执行的一组操作完成。 

CountDownLatch使用给定的计数初始化。由于调用了countDown方法，await方法一直阻塞到当前计数为零，在此之后，所有等待线程被释放，await的任何后续调用立即返回。这是一种一次性现象——计数无法重置。如果您需要一个可以重置计数的版本，请考虑使用CyclicBarrier。

例子：

```java
 public static void main(String[] args) throws InterruptedException {
    	final CountDownLatch c = new CountDownLatch(2);
        new Thread(new Runnable() {
            @Override
            public void run() {
                c.countDown();
                System.out.println("线程1执行完成");
            }
        }).start();
        new Thread(new Runnable() {
        	@Override
        	public void run() {
        		c.countDown();
        		System.out.println("线程2执行完成");
        	}
        }).start();

        c.await();
        System.out.println("退出");
}
```

结果：

```java
线程1执行完成
线程2执行完成
退出
```

CountDownLatch的构造函数接收一个,其中构造参数n表示等待n个完成，即执行countDown n次。

## 二、源码分析

```java
public class CountDownLatch {
    /**
     * 同步控制器
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }
		//重这里可以看出申请共享节点资源的时候没有修改state，只是通过判断，当state变为0的时候
        //同步控制器就没用了，await也起不了作用
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    /**
     * 初始化数量
     */
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    /**
     * await，这里的await就是 可中断的acquireShared
     */
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * 等待时间await
     */
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * 减少闩锁的计数，如果计数为零则释放所有等待线程。
     */
    public void countDown() {
        sync.releaseShared(1);
    }

    /**
     * 返回当前计数。
     */
    public long getCount() {
        return sync.getCount();
    }

}
```

```java
public abstract class AbstractQueuedSynchronizer{
    /**
     * 获取共享锁。会调用tryAcquireShared方法
     */
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    /**
     * 释放共享锁
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
}
```

为什么CountDownLatch只能起一次作用而不能重置？

通过上面的源码分析可以看出tryReleaseShared将会把state释放到0，tryAcquireShared则返回-1，acquireShared则不会再进入阻塞