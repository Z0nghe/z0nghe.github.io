---
layout: post
title:  "Java8新特性（一）forEach()"
date:   2020-02-16 14:36:11 +0800
categories: jekyll update
---
> # Java8新特性（一）forEach()

&nbsp;
最近在家办公，但是许多代码都需要连接内网的服务才能运行。所以，我需要在服务器上把vim搭建成一个简易的ide配合框架来使用。具体配置信息可以查看我的<a href="https://github.com/Z0nghe/My-Vim-plugin-Configuration" target="_blank">GitHub仓库</a>。
&nbsp;

因为项目搭建我用的Gradle，所以创建Java项目很方便。给vim加了一些插件之后许多操作变得方便了，但是由于自动补全没办法像Eclipse或IDEA那么方便，所以写Java代码本身还是有点麻烦。
&nbsp;

循环遍历是我们经常要用到的一种操作，今天就简单记录一下Java8新特性下的forEach()方法
&nbsp;

首先我们有一个数组：

```java
List<String> list = Arrays.asList("1", "2", "3", "4", "5", "6");

```

通常我们遍历这个数组需要for循环或者增强型for循环：
> 
```java
for (String num : list) {
    System.out.println(num);
}
```

而forEach()方法通过lambda表达式可以十分简单的完成这个操作
>
```java
list.forEach(num -> System.out.println(num));
```

或者
>
```java
list.forEach(num -> {
    //其他操作
    System.out.println(num);
});
```

还可以使用方法引用(method reference)进一步简化&nbsp;
>
```java
list.forEach(System.out::println);
```

在一些情况下，还可以在一些流上使用forEach()
在并行流上调用forEach时可以提高性能，但是顺序和正常list顺序可能不同&nbsp;
>
```java
list.parallelStream().forEach(System.out::println);
/* output
3
5
1
2
6
4
*/
```

关于更多Java8流(Stream)操作和方法引用(method reference)的介绍下次再说。


