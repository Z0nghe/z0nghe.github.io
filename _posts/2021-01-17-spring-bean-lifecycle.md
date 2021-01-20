---
layout: post
title:  "Spring Bean生命周期"
date:   2021-01-17 22:53:11 +0800
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

自定义`BeanPostProcessor`实现类：

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class StudentBeanPostProcessor implements BeanPostProcessor {
    public StudentBeanPostProcessor() {
        super();
        log.info("StudentBeanPostProcessor构造器");
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

自定义`BeanFactoryPostProcessor`实现类:

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
public class StudentBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    public StudentBeanFactoryPostProcessor() {
        super();
        log.info("初始化BeanFactoryPostProcessor");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition bd = beanFactory.getBeanDefinition("student");
        bd.getPropertyValues().addPropertyValue("age", "18");
        bd.getPropertyValues().addPropertyValue("name", "小明");
    }
}

```

自定义`InstantiationAwareBeanPostProcessor`实现类：

```java
package com.example.learn.student;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class StudentInstantiationAwareBeanProcessor implements InstantiationAwareBeanPostProcessor {
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

启动类：

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

执行代码后部分log如下：

```
初始化BeanFactoryPostProcessor
StudentBeanPostProcessor构造器
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
...
无关log不做展示
...
xiaoming toString(): Student{name='小明', age=18, beanName='student'}
DisposableBean接口实现，销毁bean方法
```



## 五、总结

Bean的生命周期分为：实例化->赋值->初始化->销毁 四个阶段。

在调用BeanFactory的getBean()方法后：

1. 
2. 

其中涉及的接口包括各种aware和各种processor，在后续过程中会对Aware接口和BeanPostProcessor接口进行分析。