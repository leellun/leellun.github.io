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

静态方法runAsync和supplyAsync允许我们相应地使用Runnable和Supplier函数类型创建一个可完成的未来实例。

区别：

Runnable 没有返回值。

Supplier 拥有计算结果的返回值。

```
CompletableFuture<String> future
  = CompletableFuture.supplyAsync(() -> "Hello");

// ...

assertEquals("Hello", future.get());
```

## 2 异步计算的处理结果

thenApply 接受一个函数实例，用它来处理结果，并返回一个包含函数返回值的Future

```
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");

CompletableFuture<String> future = completableFuture
  .thenApply(s -> s + " World");

assertEquals("Hello World", future.get());
```

上面结果：true

thenAccept方法接收使用者并将计算结果传递给它。与thenApply 区别则是future.get（）返回Void类型：

```
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");

CompletableFuture<Void> future = completableFuture
  .thenAccept(s -> System.out.println("Computation returned: " + s));

future.get();
```

thenRun 既不需要计算的值，也不想返回值

```
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello");

CompletableFuture<Void> future = completableFuture
  .thenRun(() -> System.out.println("Computation finished."));

future.get();
```

| 方法         | 区别      |
| ---------- | ------- |
| thenApply  | 传参  返回值 |
| thenAccept | 传参      |
| thenRun    | 无       |

## 3 组合CompletableFuture

CompletableFuture API最好的部分是能够在一系列计算步骤中组合CompletableFuture实例。

这种链接的结果本身就是一个完整的Future，允许进一步的链接和组合。这种方法在函数语言中普遍存在，通常被称为享元模式。

使用thenCompose方法按顺序链接两个Future:

```
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World");

assertEquals("Hello World", completableFuture.get());
```

thenCompose方法与thenApply一起实现了享元模式的基本构建块。它们与流的map和flatMap方法以及java8中的可选类密切相关。

两个方法都接收一个函数并将其应用于计算结果，但是thencomose（flatMap）方法接收一个返回另一个相同类型对象的函数。这种功能结构允许将这些类的实例组合为构建块。

如果我们想执行两个独立的未来，并对它们的结果进行处理，我们可以使用thenCombine方法，该方法接受一个未来和一个具有两个参数的函数来处理这两个结果：

```
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCombine(CompletableFuture.supplyAsync(
      () -> " World"), (s1, s2) -> s1 + s2));

assertEquals("Hello World", completableFuture.get());
```

## 4 并行运行多个CompletableFuture

```
CompletableFuture<String> future1  
  = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2  
  = CompletableFuture.supplyAsync(() -> "Beautiful");
CompletableFuture<String> future3  
  = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<Void> combinedFuture 
  = CompletableFuture.allOf(future1, future2, future3);


combinedFuture.get();

assertTrue(future1.isDone());
assertTrue(future2.isDone());
assertTrue(future3.isDone());
```

注意CompletableFuture.allOf（）的返回类型是CompletableFuture。这种方法的局限性在于它不能返回所有Supplier的组合结果。

```
String combined = Stream.of(future1, future2, future3)
  .map(CompletableFuture::join)
  .collect(Collectors.joining(" "));

assertEquals("Hello Beautiful World", combined);
```

join（）方法类似于get方法，但是如果Future不能正常完成，它会抛出一个未检查的异常。这样就可以将其用作Stream.map（）方法中的方法引用。

## 5 异步方法

CompletableFuture类中fluentapi的大多数方法都有另外两个带有异步后缀的变体。这些方法通常用于在另一个线程中运行相应的执行步骤。

没有异步后缀的方法使用调用线程运行下一个执行阶段。相反，不带Executor参数的Async方法使用Executor的公共fork/join池实现运行一个步骤，该实现通过ForkJoinPool.commonPool（）方法访问。最后，带有Executor参数的Async方法使用传递的Executor运行步骤。

```
CompletableFuture<String> completableFuture  
  = CompletableFuture.supplyAsync(() -> "Hello");

CompletableFuture<String> future = completableFuture
  .thenApplyAsync(s -> s + " World");

assertEquals("Hello World", future.get());
```

# 四、CompletableFuture 与forkjoin区别

`CompletableFuture` 和 `ForkJoin` 都是 Java 并发编程中的重要组件

相同点：

- 都是基于线程池实现的异步执行机制。
- 都可以通过任务分解来实现并行计算。

不同点：

- `ForkJoin` 主要用于实现数据并行计算，适用于大批量的计算任务。而 `CompletableFuture` 主要用于实现异步编程，适用于 I/O 密集型的任务。
- `ForkJoin` 通过分治策略将任务分成较小的子任务来实现并行计算。而 `CompletableFuture` 通过 CompletableFuture 对象之间的组合来实现异步编程。多个 CompletableFuture 对象可以组合在一起，形成一个异步计算流水线，通过 thenApply、thenCompose、thenCombine 等方法组成的链式调用来串联多个异步任务。
- `ForkJoin` 在实现任务分解的过程中，需要手动编写代码来分解任务，并创建 ForkJoinTask 对象。而 `CompletableFuture` 可以通过 `runAsync`、`supplyAsync` 等方法来直接创建 CompletableFuture 对象，无需手动创建异步任务。

总的来说，`ForkJoin` 更适合实现 CPU 密集型的计算任务，而 `CompletableFuture` 更适合实现 I/O 密集型的异步任务，二者都可以通过并行化来提高任务的执行效率。

# 五、项目中的应用

比如最近我的微信小程序首页数据获取

```'
CompletableFuture<List<Ad>> bannerFuture = CompletableFuture.supplyAsync(() -> adService.queryIndex(),executor);
        CompletableFuture<List<Category>> channelFuture = CompletableFuture.supplyAsync(() -> categoryService.queryChannel(),executor);

        CompletableFuture<List<Coupon>> couponFuture;
        if (userId == null) {
            couponFuture = CompletableFuture.supplyAsync(() -> couponService.listCoupons(3));
        } else {
            couponFuture = CompletableFuture.supplyAsync(() -> couponService.listAvailableCoupons(userId, 3));
        }
        CompletableFuture<List<Goods>> newGoodsListCallable = CompletableFuture.supplyAsync(() -> goodsService.listByNew(configHelper.getNewLimit()),executor);
        CompletableFuture<List<Goods>> hotGoodsListCallable = CompletableFuture.supplyAsync(() -> goodsService.listByHot(configHelper.getHotLimit()),executor);
        CompletableFuture<List<Brand>> brandListCallable = CompletableFuture.supplyAsync(() -> brandService.query(configHelper.getBrandLimit()),executor);
        CompletableFuture<List<Topic>> topicListCallable = CompletableFuture.supplyAsync(() -> topicService.queryList(configHelper.getTopicLimit()),executor);
        //团购专区
        CompletableFuture<List<GrouponRuleVO>> grouponListCallable = CompletableFuture.supplyAsync(() -> grouponRulesService.listGrouponRuleVO(PageEntity.page(0, 5, "create_time", true)).getList(),executor);
        CompletableFuture<List<CategoryGoodsVO>> floorGoodsListCallable = CompletableFuture.supplyAsync(this::getCategoryList,executor);

        try {
            HomeIndexVO homeIndexVO = new HomeIndexVO();
            homeIndexVO.setBanner(bannerFuture.get());
            homeIndexVO.setBrandList(brandListCallable.get());
            homeIndexVO.setChannel(channelFuture.get());
            homeIndexVO.setCouponList(couponFuture.get());
            homeIndexVO.setNewGoodsList(newGoodsListCallable.get());
            homeIndexVO.setHotGoodsList(hotGoodsListCallable.get());
            homeIndexVO.setTopicList(topicListCallable.get());
            homeIndexVO.setGrouponList(grouponListCallable.get());
            homeIndexVO.setFloorGoodsList(floorGoodsListCallable.get());
            //缓存数据
            HomeCacheManager.loadData(HomeCacheManager.INDEX, homeIndexVO);
            return homeIndexVO;
        } catch (Exception e) {
            e.printStackTrace();
            throw new BusinessException("数据获取失败");
        }
```

