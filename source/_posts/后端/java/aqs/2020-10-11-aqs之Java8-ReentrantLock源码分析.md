---
title: aqs之Java8 ReentrantLock源码分析
date: 2020-10-11 20:14:02
categories:
  - 后端
  - java
tags:
  - aqs
  - 源码
---

## 一、介绍

一个可重入互斥锁，其基本行为和语义与使用同步方法和语句访问的隐式监视锁相同，但具有扩展功能。 

ReentrantLock由上次成功锁定但尚未解锁的线程拥有。当锁不属于另一个线程时，调用锁的线程返回成功则获得锁。如果当前线程已经拥有锁，该方法将立即返回。 

公平锁：

在争用状态下，锁倾向于将访问权授予等待时间最长的线程。使用公平锁被多个线程访问的程序可能显示较低的总体吞吐量

非公平锁：

此锁不能保证任何特定的访问顺序。

锁的公平性并不能保证线程调度的公平性。因此，使用公平锁的多个线程中的一个可以连续多次获得它，而其他活动线程没有进展，当前也没有持有该锁。

```java
 class X {    
     private final ReentrantLock lock = new ReentrantLock();    
     // ...      
     public void m() {      
         lock.lock();  
         // block until condition holds      
         try {        
             // ... method body      
         } finally {        
             lock.unlock()      
         }    
     }  
 }
```

## 二、源码分析

此锁的同步控制基础。下面分为公平版本和不公平版本。使用aqs状态表示锁上的持有次数。 

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * 非公平
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //可重入锁逻辑，如果当前申请执行权的线程就是当前线程，则只是增加当前state的大小
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                //因为nextc是int，所以最大为Integer.MAX_VALUE,当再加1就会抛出error异常
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
	
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            //ReentratLock是线程分配限制为1，这里报错的原因：如果没有lock，直接unlock
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {//当资源释放完了，这个时候需要设置当前拥有线程为null
                free = true;
                setExclusiveOwnerThread(null);
            }
            //这句代码非常重要，因为ReentratLock设计的是同一时刻只会有一个线程的量，即从lock的compareAndSetState(0, 1)开始到
            //setState(0),这中间的逻辑都是线程安全的，不用过多考虑，因为它就是同一时刻的单线程操作
            setState(c);
            return free;
        }

        /**
         * 是否当前线程拥有者
         * @return
         */
        protected final boolean isHeldExclusively() {
            // 检查当前线程是否是所有者
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        /**
         * 拥有者线程
         */
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }
    }
```

### 2.1 非公平锁

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * 执行锁
         */
        final void lock() {
            //这句代码就是公平锁和非公平锁最大的区别，如果获取到资源就会获取执行权，而不用考虑阻塞队列中是否有node
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

### 2.2 公平锁

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * 试图申请资源
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //对公平锁，不同于非公平锁，主要还是hasQueuedPredecessors()查看阻塞队列中是否有非当前线程的阻塞node
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
    /**
     * 是否有线程等待获取锁的时间比当前线程长
     */
    public final boolean hasQueuedPredecessors() {
        Node t = tail; 
        Node h = head;
        Node s;
        //判断阻塞队列中是否有节点:
        //(1)头节点不等于尾节点
        //(2)如果头节点的下一个节点为空，因为在添加节点的时候先设置prev，再设置next，所以这个时候阻塞队列中肯定有下一个节点，
        //只是还没来得及设置，则表明当前操作和节点处于不同线程
        //（3）如果下一个节点的线程和当前线程不一样
        return h != t &&
                ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

