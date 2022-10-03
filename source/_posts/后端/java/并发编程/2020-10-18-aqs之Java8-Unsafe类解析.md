---
title: aqs之Java8 Unsafe类解析
date: 2020-10-18 20:14:02
categories:
  - 后端
  - java
tags:
  - 并发编程
---

# 一、介绍

Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，比如

- 直接访问系统内存资源
- 自主管理内存资源等

可以使用Unsafe的native方法间接访问底层方法，增强Java语言底层资源操作能力方面起到了很大的作用。通过native使Unsafe有操作内存空间的能力，这也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用Unsafe类会使得程序出错的概率变大，使得Java这种安全的语言变得不再“安全”，因此对Unsafe的使用一定要慎重。 

# 二、Unsafe怎么才能使用

```
public final class Unsafe {
  // 单例对象
  private static final Unsafe theUnsafe;

  private Unsafe() {
  }
  @CallerSensitive
  public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    // 仅在引导类加载器`BootstrapClassLoader`加载时才合法
    if(!VM.isSystemDomainLoader(var0.getClassLoader())) {    
      throw new SecurityException("Unsafe");
    } else {
      return theUnsafe;
    }
  }
}
```

通过Java命令行命令`-Xbootclasspath/a`把调用Unsafe相关方法 

```
java -Xbootclasspath/a: ${path}   // 其中path为调用Unsafe相关方法的类所在jar包路径 
```

通过反射获取单例对象theUnsafe 

```
private static Unsafe reflectGetUnsafe() {
    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      return (Unsafe) field.get(null);
    } catch (Exception e) {
      log.error(e.getMessage(), e);
      return null;
    }
}
```

# 三、功能介绍

相关文章：

https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html

https://www.cnblogs.com/zmhjay1999/p/15183390.html