---
title: "Spring"
date: 2020-01-05T18:05:19+08:00
lastmod: 2020-01-05T18:05:19+08:00
draft: false
keywords:
-
description: ""
tags:
-
categories:
-
author: "lyoga"
---

<!--more-->
# **Spring**

## **IOC**

- 即控制反转，一种设计思想。IOC容器来控制对象的创建，装配组件依赖关系、负责组件生命周期管理
- 优点：松耦合，减少类与类之间依赖，利用对象管理，利于功能复用

**初始化**

- Resource定位（配置xml文件转换成resource）
- BeanDefinition的载入和解析（利用XmlBeanDefinitionReader完成对xml的解析，将xml Resource里定义的bean对象转换成统一的BeanDefinition）
- BeanDefinition注册（将BeanDefinition注册到BeanFactory，完成对BeanFactory的初始化。BeanFactory里将会维护一个BeanDefinition的Map）

## **Spring Bean 生命周期**

- Bean 容器找到配置文件中 Spring Bean 的定义
- Bean 容器利用 Java Reflection API 创建一个Bean的实例
- Spring对bean进行依赖注入
- bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁
- 若bean实现了DisposableBean接口，spring将调用它的distroy()接口方法；同样的，如果bean使用了destroy-method属性声明了销毁方法，则该方法被调用

## **bean作用域**

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
- session : 每一次HTTP请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。
- global-session： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

## **Spring 中的单例 bean 的线程安全问题**
**主要是因为当多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题 - 在类中定义一个ThreadLocal成员变量，将需要的可变成员变量保存在 ThreadLocal 中**

## **AOP**
**Spring AOP就是基于动态代理的，如果要代理的对象，实现了某个接口，那么Spring AOP会使用JDK Proxy，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用Cglib ，这时候Spring AOP会使用 Cglib 生成一个被代理对象的子类来作为代理**

### **JDK 动态代理实现原理**

- 通过实现 InvocationHandler 接口创建自己的 调用处理器
- 通过为 Proxy 类指定 ClassLoader 对象和一组 Interface 来创建 动态代理
- 通过 反射 机制获取 动态代理类 的构造函数，其唯一参数类型，就是 调用处理器 的接口类型
- 通过 构造函数 创建 动态代理类 实例，构造时 调用处理器对象 作为参数传入

### **CGLib 动态代理**
**基于ASM的字节码生成库，采用底层的字节码技术，其可以为类创建一个子类，在子类之中采用方法拦截的技术来拦截所有父类的方法的调用，并且顺势织入横切逻辑**

### **cglib与动态代理区别**

- 使用动态代理的对象必须实现一个或多个接口(InvocationHandler接口)
- 使用cglib代理(基于ASM的字节码生成库)的对象则无需实现接口，达到代理类无侵入
- cglib代理无法处理final的情况（final修饰的方法不能被覆写）
- JDK动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为Proxy，Java类继承机制不允许多重继承）；CGLIB能够代理普通类，但是该普通类必须能够被继承(不能用final修饰符)。
- JDK动态代理使用Java原生的反射API进行操作，在生成类上比较高效；CGLIB使用ASM框架直接对字节码进行修改，使用了FastClass的特性，在某些情况下类的方法执行会比较高效


![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200106145619.png)

- 连接点：在哪里能够功能 方法上
- 切入点：想在哪里加入功能
- 通知：加入什么功能
- 切面： 通知+切入点  在哪加什么功能
- 目标对象：对谁加功能

## **Spring AOP 和 AspectJ AOP 有什么区别？**

- Spring AOP 属于运行时增强，而 AspectJ 是编译时增强
- Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)

## **Spring MVC 处理流程**
![](https://lyoga-1257336739.cos.ap-beijing.myqcloud.com/20200106161613.png)

- 客户端（浏览器）发送请求，直接请求到 DispatcherServlet。
- DispatcherServlet 根据请求信息调用 HandlerMapping，解析请求对应的 Handler。
- 解析到对应的 Handler（也就是我们平常说的 Controller 控制器）后，开始由 HandlerAdapter 适配器处理。
- HandlerAdapter 会根据 Handler来调用真正的处理器开处理请求，并处理相应的业务逻辑。
- 处理器处理完业务后，会返回一个 ModelAndView 对象，Model 是返回的数据对象，View 是个逻辑上的 View。
- ViewResolver 会根据逻辑 View 查找实际的 View。
- DispaterServlet 把返回的 Model 传给 View（视图渲染）。
- 把 View 返回给请求者（浏览器）

## **Spring Boot**
**Spring 组件一站式解决方案，简化了使用 Spring 的难度，简化了配置；使用“习惯优于配置”的理念让项目快速运行起来；**

- 优点：快速构建项目；对当前主流框架SSM等，无配置集成；项目可以独立运行；提供监控；提高开发效率；
- 缺点：依赖太多，中文文档较少

**spring boot运行原理就是容器的思想，先将普遍使用的配置项加载入容器，只有在加载相应jar包的情况下，才能将配置属性通过配置文件加载入服务类**


- 编程式事务
- 声明式事务
  - 基于XML
  - 基于注解

```
@Transaction
// public ; 自调问题; 基于切面代理实现
```
