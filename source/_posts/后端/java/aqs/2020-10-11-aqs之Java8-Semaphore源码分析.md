---
title: aqs之Java8 Semaphore源码分析
date: 2020-10-11 20:14:02
categories:
  - 后端
  - java
tags:
  - aqs
  - 源码
---

## 一、介绍

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

Semaphore的构造方法Semaphore（int permits）接受一个整型的数字，表示可用的许可证数量。Semaphore（10）表示允许10个线程获取许可证，也就是最大并发数是10。

Semaphore的acquire()方法获取一个许可证，使用完之后调用release()方法归还许可证。还可以用tryAcquire()方法尝试获取许可证。

```java
class Pool {    
    private static final int MAX_AVAILABLE = 100;    
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);      
    public Object getItem() throws InterruptedException {      
        available.acquire();      
        return getNextAvailableItem();    
    }      
    public void putItem(Object x) {      
        if (markAsUnused(x))        
            available.release();    
    }      
}
```

## 二、源码解析

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {//执行许可量
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }
		//非公平锁的共享节点的申请
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                //许可量剩余
                int remaining = available - acquires;
                //如果许可量小于0将会进入阻塞队列，cas设置状态量设置成功也会返回
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        //共享锁释放
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
        //减少信号量
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    /**
     * 非公平锁
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                //公平锁，如果阻塞队列中有组赛节点，则返回<0,将当前节点添加到tail
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

