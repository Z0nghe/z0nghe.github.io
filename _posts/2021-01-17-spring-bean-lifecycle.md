---
layout: post
title:  "Spring Bean生命周期"
date:   2021-01-21 22:53:11 +0800
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

## 四、代码演示

### 1.Bean类

创建一个`Student`类，实现一个简单Bean需要的四个接口`BeanFactoryAware`，`BeanNameAware`，`InitializingBean`，`DisposableBean`：

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class Student implements BeanFactoryAware, BeanNameAware, InitializingBean, DisposableBean {
    private String name;
    private int age;
    private BeanFactory beanFactory;
    private String beanName;

    public Student() {
        log.info("Student无参构造器");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        log.info("BeanFactoryAware接口实现，注入BeanFactory:{}", beanFactory.getClass().getName());
        this.beanFactory = beanFactory;
    }

    @Override
    public void setBeanName(String name) {
        log.info("BeanFactoryAware接口实现，注入BeanName：{}", name);
        this.beanName = name;
    }

    @Override
    public void destroy() throws Exception {
        log.info("DisposableBean接口实现，销毁bean方法");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        log.info("InitializingBean接口实现，初始化Bean方法");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        log.info("注入name：{}", name);
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        log.info("注入age：{}", age);
        this.age = age;
    }

    public BeanFactory getBeanFactory() {
        return beanFactory;
    }

    public String getBeanName() {
        return beanName;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", beanName='" + beanName + '\'' +
                '}';
    }
}

```

### 2.自定义`BeanPostProcessor`实现类

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class MyBeanPostProcessor implements BeanPostProcessor {
    public MyBeanPostProcessor() {
        super();
        log.info("初始化MyBeanPostProcessor");
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        log.info("postProcessBeforeInitialization对属性进行更改, beanName:{}", beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        log.info("postProcessAfterInitialization对属性进行更改, beanName:{}", beanName);
        return bean;
    }
}

```

### 3.自定义`BeanFactoryPostProcessor`实现类

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    public MyBeanFactoryPostProcessor() {
        super();
        log.info("初始化MyBeanFactoryPostProcessor");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition bd = beanFactory.getBeanDefinition("student");
        bd.getPropertyValues().addPropertyValue("age", "18");
        bd.getPropertyValues().addPropertyValue("name", "小明");
    }
}

```

### 4.自定义`InstantiationAwareBeanPostProcessor`实现类

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class MyInstantiationAwareBeanProcessor implements InstantiationAwareBeanPostProcessor {
    public MyInstantiationAwareBeanProcessor() {
        log.info("初始化MyInstantiationAwareBeanProcessor");
    }

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        log.info("实例化Bean之前调用, beanName:{}", beanName);
        return null;
    }

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        log.info("实例化Bean之后调用， beanName:{}", beanName);
        return true;
    }

    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        log.info("设置Bean属性时调用, beanName:{}",  beanName);
        return pvs;
    }
}

```

### 5.启动类

```java
package com.example.learn;

import com.example.learn.student.Student;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@Slf4j
@SpringBootApplication
public class LearnApplication implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(LearnApplication.class, args);
    }

    @Autowired
    private Student xiaoming;

    @Override
    public void run(String... args) throws Exception {
        log.info("xiaoming toString(): {}", xiaoming);
    }
}

```

### 6.运行log

```text
初始化MyBeanFactoryPostProcessor
初始化MyBeanPostProcessor
初始化MyInstantiationAwareBeanProcessor
实例化Bean之前调用, beanName:learnApplication
实例化Bean之后调用， beanName:learnApplication
设置Bean属性时调用, beanName:learnApplication
实例化Bean之前调用, beanName:student
Student无参构造器
实例化Bean之后调用， beanName:student
设置Bean属性时调用, beanName:student
注入age：18
注入name：小明
BeanFactoryAware接口实现，注入BeanName：student
BeanFactoryAware接口实现，注入BeanFactory:org.springframework.beans.factory.support.DefaultListableBeanFactory
postProcessBeforeInitialization对属性进行更改, beanName:student
InitializingBean接口实现，初始化Bean方法
postProcessAfterInitialization对属性进行更改, beanName:student
postProcessBeforeInitialization对属性进行更改, beanName:learnApplication
postProcessAfterInitialization对属性进行更改, beanName:learnApplication
...
其他Bean初始化log，不做展示
...
xiaoming toString(): Student{name='小明', age=18, beanName='student'}
DisposableBean接口实现，销毁bean方法
```



## 五、总结

Bean的生命周期分为：实例化->赋值->初始化->销毁 四个阶段。

由于我是用的是Spring Boot框架来测试的Bean生命周期，所以不需要执行BeanFactory的getBean()方法也可以初始化一个Bean。

### 1.初始化PostProcessor

在几个PostProcessor接口的源码中都有一段关于注册到ApplicationContext的相似的注释。

BeanFactoryPostProcessor：

> An ApplicationContext auto-detects BeanFactoryPostProcessor beans in its bean definitions and applies them before any other beans get created. A BeanFactoryPostProcessor may also be registered programmatically with a ConfigurableApplicationContext.

BeanPostProcessor：

> An ApplicationContext can autodetect BeanPostProcessor beans in its bean definitions and apply those post-processors to any beans subsequently created. A plain BeanFactory allows for programmatic registration of post-processors, applying them to all beans created through the bean factory.

`InstantiationAwareBeanPostProcessor`接口继承了`BeanPostProcessor`接口的子类，所以本质上也是一个BeanPostProcessor。

这三个接口实现类的Bean会先于其他Bean注册到ApplicationContext，为后续Bean的生命周期处理做准备。

### 2.实例化Bean

实例化的过程分为四步：

1. 调用`InstantiationAwareBeanPostProcessor`接口实现类的`postProcessBeforeInstantiation`方法
2. 调用Spring Bean类的构造方法
3. 调用`InstantiationAwareBeanPostProcessor`接口实现类的`postProcessAfterInstantiation`方法
4. 调用`InstantiationAwareBeanPostProcessor`接口实现类的`postProcessProperties`方法

### 3.Bean注入属性

1. 调用`InstantiationAwareBeanPostProcessor`接口实现类的`postProcessProperties`方法
2. 调用`BeanFactoryPostProcessor`接口实现类的`postProcessBeanFactory`方法配置Bean属性
3. 调用Bean的Aware接口方法配置Bean的BeanName和BeanFactory

### 4.初始化

执行Bean的构造函数并且在调用构造函数前后分别调用`postProcessBeforeInitialization`和`postProcessAfterInitialization`方法

### 5.销毁

在BeanFactory shutdown时，调用Bean的`DisposableBean`接口实现方法销毁Bean

至此，Spring Bean生命周期结束。