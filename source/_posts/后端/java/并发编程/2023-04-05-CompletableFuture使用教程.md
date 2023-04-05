---
title: CompletableFuture使用教程
date: 2023-04-05 12:33:54
categories:
  - 后端
  - java
tags:
  - volatile 
---

# 一、简介

Future是Java 5中添加作为异步计算的结果，但它没有任何方法处理计算可能出现的错误。 而CompletableFuture是在Java8后开始引进的，CompletableFuture提供了非常强大的Future的扩展功能，可以简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合 CompletableFuture 的方法。 

CompletableFuture类，除Future接口外，它还实现了CompletionStage接口。该接口为异步计算步骤定义了合同，我们可以将其与其他步骤结合使用。 

# 二、方法介绍

## 2.1 创建CompletableFuture对象的方法 

- CompletableFuture.runAsync(Runnable runnable)：创建一个异步任务，没有返回值；
- CompletableFuture.supplyAsync(Supplier<U> supplier)：创建一个异步任务，有返回值。
- CompletableFuture.completedFuture(T value) 创建一个已经完成的CompletableFuture对象，并返回该对象的结果值。 
- allOf(CompletableFuture<?>... cfs)：等待所有的异步任务执行完成；
- anyOf(CompletableFuture<?>... cfs)：等待任意一个异步任务执行完成；

## 2.2 创建自带Executor

delayedExecutor(long delay, TimeUnit unit)  创建一个延迟执行任务的执行器对象 

## 2.3 异步任务的处理方法

- thenApply(Function<? super T, ? extends U> fn)：对异步任务的结果进行转换；
- thenAccept(Consumer<? super T> action)：对异步任务的结果进行消费，没有返回值；
- thenRun(Runnable action)：对异步任务的结果进行消费，没有参数和返回值；
- thenCombine(CompletionStage<? extends U> other, BiFunction<? super T, ? super U, ? extends V> fn)：将两个异步任务的结果合并，并返回一个新的结果；
- thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)：将一个异步任务的结果作为另一个异步任务的输入。

## 2.4 异步任务的异常处理方法

- exceptionally(Function<Throwable, ? extends T> fn)：捕获异步任务中的异常，并返回一个默认值或抛出一个新的异常；
- handle(BiFunction<? super T, Throwable, ? extends U> fn)：对异步任务的结果或异常进行处理，返回一个新的结果。

## 2.5 异步任务的执行方法

- join()：等待异步任务执行完成，并返回其结果；
- get()：等待异步任务执行完成，如果有异常则抛出异常。

## 2.6 完成异步任务的方法

- complete(T value)：将异步任务标记为已完成，并设置完成结果；
- completeExceptionally(Throwable ex)：将异步任务标记为已完成，并设置异常结果。

## 2.7 组合多个异步任务的方法

- allOf(CompletableFuture<?>... cfs)：等待所有的异步任务执行完成；
- anyOf(CompletableFuture<?>... cfs)：等待任意一个异步任务执行完成；
- thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)：等待两个异步任务都执行完成，并消费结果；
- runAfterBoth(CompletionStage<?> other, Runnable action)：等待两个异步任务都执行完成，并执行指定的操作；
- runAfterEither(CompletionStage<?> other, Runnable action)：等待两个异步任务中任意一个执行完成，并执行指定的操作；
- applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)：等待两个异步任务中任意一个执行完成，并对其结果进行转换。

## 2.8 超时处理方法

- completeOnTimeout(T value, long timeout, TimeUnit unit)：在指定时间内等待异步任务执行完成，如果还未执行完成则将其标记为已完成，并设置默认值；
- orTimeout(long timeout, TimeUnit unit)：在指定时间内等待异步任务执行完成，如果还未执行完成则将其标记为已完成，并设置异常结果。

## 2.9 辅助方法：

- isDone()：判断异步任务是否已完成；
- exceptionallyCompose(Function<Throwable, ? extends CompletionStage<T>> fn)：捕获异步任务中的异常，并返回一个新的异步任务；
- toCompletableFuture()：将CompletableFuture转换为Future。

## 2.10 同步化处理方法

- getNow(T value)：如果异步任务已完成，则返回异步任务的结果，否则返回指定的默认值；
- join()：等待异步任务执行完成，并返回其结果；
- complete(T value)：将异步任务标记为已完成，并设置完成结果；
- completeExceptionally(Throwable ex)：将异步任务标记为已完成，并设置异常结果。

## 2.11 取消异步任务：

- cancel(boolean mayInterruptIfRunning)：尝试取消异步任务的执行，如果任务已经执行完成，则返回false，否则返回true。

## 2.12 并发控制：

- CompletableFuture.thenApplyAsync(Function<? super T, ? extends U> fn, Executor executor)：指定异步任务的执行线程池；
- CompletableFuture.thenApplyAsync(Function<? super T, ? extends U> fn)：默认使用ForkJoinPool.commonPool()线程池执行异步任务。

# 三、使用

```
public Future<String> calculateAsync() throws InterruptedException {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();

    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.complete("Hello");
        return null;
    });

    return completableFuture;
}
```

## 1 封装计算逻辑

```
CompletableFuture<String> future
  = CompletableFuture.supplyAsync(() -> "Hello");

// ...

assertEquals("Hello", future.get());
```

## 2 异步计算的处理结果

```
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");

CompletableFuture<String> future = completableFuture
  .thenApply(s -> s + " World");

assertEquals("Hello World", future.get());
```









https://mp.weixin.qq.com/s?__biz=MzU4OTMwODQ4Nw==&mid=2247484679&idx=3&sn=d97e41ebd9260d9f7f8cff2c67c7be39&chksm=fdce322fcab9bb39dbdab0f0b68bc85846ba4c64646a1dd3f2f2cdb7e107034a8b03222978ff&scene=27