---
layout: post
title:  "Spring Bean生命周期"
date:   2020-01-17 22:53:11 +0800
categories: java spring bean
abstract: "阅读BeanFactory接口源码并对bean生命周期进行简要分析"
---
# Spring Bean生命周期

## 一、BeanFactory接口功能

众所周知，Bean工厂接口的主要方法就是提供Bean实例的getBean()方法和多个重载方法。除此之外还包括一些通过bean name进行判断的辅助方法比如isSingleton()、isPrototype()、contanceBean()等。

在源码的接口文档中有这样一段描述：

> This interface is implemented by objects that hold a number of bean definitions, each uniquely identified by a String name. Depending on the bean definition, the factory will return either an independent instance of a contained object (the Prototype design pattern), or a single shared instance (a superior alternative to the Singleton design pattern, in which the instance is a singleton in the scope of the factory). Which type of instance will be returned depends on the bean factory configuration: the API is the same. Since Spring 2.0, further scopes are available depending on the concrete application context (e.g. "request" and "session" scopes in a web environment).

所以，BeanFactory接口的作用就是提供Bean，Bean的整个生命周期除了销毁阶段，都在这个接口的getBean()方法前完成。

## 二、BeanFactory初始化Bean时的标准生命周期接口

> Bean factory implementations should support the standard bean lifecycle interfaces as far as possible. The full set of initialization methods and their standard order is:
>
> 1. BeanNameAware's setBeanName
> 2. BeanClassLoaderAware's setBeanClassLoader
> 3. BeanFactoryAware's setBeanFactory
> 4. EnvironmentAware's setEnvironment
> 5. EmbeddedValueResolverAware's setEmbeddedValueResolver
> 6. ResourceLoaderAware's setResourceLoader (only applicable when running in an application context)
> 7. ApplicationEventPublisherAware's setApplicationEventPublisher (only applicable when running in an application context)
> 8. MessageSourceAware's setMessageSource (only applicable when running in an application context)
> 9. ApplicationContextAware's setApplicationContext (only applicable when running in an application context)
> 10. ServletContextAware's setServletContext (only applicable when running in a web application context)
> 11. postProcessBeforeInitialization methods of BeanPostProcessors
> 12. InitializingBean's afterPropertiesSet
> 13. a custom init-method definition
> 14. postProcessAfterInitialization methods of BeanPostProcessors

创建标准的Bean的生命周期包含了以上14个步骤，包括对Bean属性的注入、执行自定义初始化方法(custom init-method)以及在自定义初始化方法前后执行的处理器(BeanPostProcessors)。

## 三、BeanFactory关闭时执行的Bean销毁生命周期接口

> On shutdown of a bean factory, the following lifecycle methods apply:
>
> 1. postProcessBeforeDestruction methods of DestructionAwareBeanPostProcessors
> 2. DisposableBean's destroy
> 3. a custom destroy-method definition

关闭BeanFactory时，Bean也会随之销毁。

## 四、总结

Bean的生命周期分为：实例化->赋值->初始化->销毁 四个阶段。

在调用BeanFactory的getBean()方法后：

1. 
2. 

其中涉及的接口包括各种aware和各种processor，在后续过程中会对Aware接口和BeanPostProcessor接口进行分析。

## 五、代码演示

```java

```

