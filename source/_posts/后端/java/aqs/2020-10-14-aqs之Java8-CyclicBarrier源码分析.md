---
title: aqs之Java8 CyclicBarrier源码分析
date: 2020-10-14 20:14:02
categories:
  - 后端
  - java
tags:
  - aqs
  - 源码
---

## 一、介绍

让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障

```java
private static CyclicBarrier c=new CyclicBarrier(); 
public static void main(String[] args) {
        new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {

                }
                System.out.println(1);
            }
        }).start();

        try {
            c.await();
        } catch (Exception e) {

        }
        System.out.println(2);
}
```

## 二、源码分析

```java
public class CyclicBarrier {
	/** 锁，锁:用于守卫入口的锁 */
    private final ReentrantLock lock = new ReentrantLock();
    /**等待直到触发的条件 */
    private final Condition trip = lock.newCondition();
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();
			//如果当前线程打断过
            if (Thread.interrupted()) {
                //这里会唤醒所有等待线程，并且抛出异常
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
   public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
    /**
     * 设置当前屏障生成被打破并唤醒所有人。只在持有锁时调用。
     */
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
    //这个方法就是为什么CyclicBarrier可以重置数量
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
}
```

从代码中可以看出CyclicBarrier和CountDownLatch的区别：

- CountDownLatch是通过共享锁机制实现的计数阻塞
- CyclicBarrier使用的是Condition实现的计数阻塞，CyclicBarrier是自己维护了一个count，当count=0时，唤醒所有的条件队列中的节点，然后重置count的数量。