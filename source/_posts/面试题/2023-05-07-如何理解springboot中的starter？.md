---
title: 如何理解springboot中的starter？
date: 2023-04-07 12:38:02
categories:
  - 面试题
tags:
  - redis
  - key冲突
author: leellun
---

## 1、starter由来  

使用 spring + springmvc 框架进行开发的时候，如果需要引入 mybatis 框架，那么需要在 xml 中定义需要的 bean 对象，这个过程很明显是很麻烦的，如果需要引入额外的其他组件，那么也需要进行复杂的配置，因此在 springboot 中引入了 starter 

## 2、starter是什么？

Spring Boot的starter是一种依赖包，可以通过一种简单的方式来自动配置应用程序所需的所有依赖项。starter包含了一组预先定义的依赖关系和设置，可以简化应用程序的配置和构建过程，避免了繁琐的依赖管理工作。

写一个＠ Configuration 的配置类，将这些 bean 定义在其中，然后再 starter 包的 META - INF / spring . factories 中写入配置类，那么 springboot 程序在启动的时候就会按照约定来加载该配置类。

开发人员只需要将相应的 starter 包依赖进应用中，进行相关的属性配置，就可以进行代码开发，而不需要单独进行 bean 对象的配置

