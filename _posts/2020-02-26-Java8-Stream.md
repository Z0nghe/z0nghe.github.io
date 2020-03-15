---
layout: post
title:  "Java8新特性（三）流操作(Stream)"
date:   2020-02-26 20:30:11 +0800
categories: java java8 stream
abstract: "在Java8中新添加了Stream，Stream是对集合对象（Collection）的增强处理，它可以很简介方便的处理数据。Stream和原来的InputStream、FileReader等完全不是一个概念，虽然他们都叫流。"
---

### 一、前言
在Java8中新添加了Stream，Stream是对集合对象（Collection）的增强处理，它可以很简介方便的处理数据。Stream和原来的InputStream、FileReader等完全不是一个概念，虽然他们都叫流。

### 二、使用方法

#### 1.创建流
首先随便创建一个List

```java
List<String> list = Arrays.asList("这里", "有", "", "许多", "字符串", "其中", "还有", "一个", "空字符串");
```
Stream默认创建的是串行流，也可以创建并行流
创建一个串行流
forEach的参数是一个Consumer<? super T> 是一个函数式接口
```java
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
```

创建并行流的打印结果是乱序的
但是批量处理的速度会比串行流快
```java
Stream<String> parallelStream = list.parallelStream();
parallelStream.forEach(System.out::println);
```

#### 2.map操作
一个流只能进行一个操作
map是将流进行加工后返回一个新的流
forEach由于没有返回值所以流自动close了
```java
list.stream().map(word -> word + '*').forEach(System.out::println);
```

#### 3.filter操作
filter可以设置条件过滤流中的元素
```java
list.stream().filter(word -> word.length() > 2).forEach(System.out::println);
```

#### 4.limit操作
limit可以获取指定数量的元素
limit的参数是long
```java
list.stream().limit(4L).forEach(System.out::println);
```

#### 5.sort操作
sorted()方法和集合的sort()方法类似
也可以传入一个Comparator接口
在Java8中Comparator接口也加上了函数式接口注解
```java
list.stream().sorted((Comparator.comparingInt(String::length))).forEach(System.out::println);
```

#### 6.collect操作
collect可以使用Collectors类将流转换成List或者Set等
```java
List<String> newList = list.stream().filter(word -> !word.isEmpty()).collect(Collectors.toList());
System.out.println(newList.toString());
String string = list.stream().filter(word -> !word.isEmpty()).collect(Collectors.joining("", "=>", "<="));
System.out.println(string);
```

#### 7.match操作
match分三种
返回一个bool值
```java
boolean allMatch = list.stream().allMatch(s -> s.contains("这里"));
boolean anyMatch = list.stream().anyMatch(s -> s.contains("这里"));
boolean noneMatch = list.stream().noneMatch(s -> s.contains("这里"));
System.out.println(allMatch);
System.out.println(anyMatch);
System.out.println(noneMatch);
```

#### 8.统计操作
```java
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
IntSummaryStatistics stats = numbers.stream().mapToInt((x) -> x).summaryStatistics();
System.out.println("列表中最大的数 : " + stats.getMax());
System.out.println("列表中最小的数 : " + stats.getMin());
System.out.println("所有数之和 : " + stats.getSum());
System.out.println("平均数 : " + stats.getAverage());
```

### 三、最后附上测试代码和结果

```java
import java.util.Arrays;
import java.util.Comparator;
import java.util.IntSummaryStatistics;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class Java8StreamTest {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("这里", "有", "", "许多", "字符串", "还有", "一个", "空字符串");

        /*
         * 创建一个串行流
         * forEach的参数是一个Consumer<? super T> 是一个函数式接口
         * 我们可以创建一个Consumer实例并把lambda表达式赋值给它或者直接在参数位置写入lambda表达式或者方法引用
         * 一般我们都选择后者，因为这样看起来比较简洁易懂
         */
        System.out.println("\nStream forEach:");
        Stream<String> stream = list.stream();
        stream.forEach(System.out::println);

        /*
         * 创建一个并行流
         * parallelStream是并行流所以它输出的字符串是乱序的
         * 但是批量处理的速度会比串行流快
         */
        System.out.println("\nParallelStream forEach:");
        Stream<String> parallelStream = list.parallelStream();
        parallelStream.forEach(System.out::println);


        /*
         * 一个流只能进行一个操作
         * map是将流进行加工后返回一个新的流
         * forEach由于没有返回值所以流自动close了
         */
        System.out.println("\nStream map:");
        list.stream().map(word -> word + '*').forEach(System.out::println);

        /*
         * filter可以设置条件过滤流中的元素
         */
        System.out.println("\nStream filter:");
        list.stream().filter(word -> word.length() > 2).forEach(System.out::println);

        /*
         * limit可以获取指定数量的元素
         * limit的参数是long
         */
        System.out.println("\nStream limit:");
        list.stream().limit(4L).forEach(System.out::println);

        /*
         * sorted()方法和集合的sort()方法类似
         * 没有参数时按toString升序排列
         * 也可以传入一个Comparator接口
         * 在Java8中Comparator接口也加上了函数式接口注解
         */
        System.out.println("\nStream sorted:");
        list.stream().sorted((Comparator.comparingInt(String::length))).forEach(System.out::println);

        //临时想到的问题Comparator类加了函数式接口注解但是接口里有很多方法

        /*
         * collect可以使用Collectors类将流转换成List或者Set等
         */
        System.out.println("\nStream sorted:");
        List<String> newList = list.stream().filter(word -> !word.isEmpty()).collect(Collectors.toList());
        System.out.println(newList.toString());
        String string = list.stream().filter(word -> !word.isEmpty()).collect(Collectors.joining("", "=>", "<="));
        System.out.println(string);


        /*
         * 还有一种流的统计结果
         */
        System.out.println("\nStream statistics:");
        List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
        IntSummaryStatistics stats = numbers.stream().mapToInt((x) -> x).summaryStatistics();
        System.out.println("列表中最大的数 : " + stats.getMax());
        System.out.println("列表中最小的数 : " + stats.getMin());
        System.out.println("所有数之和 : " + stats.getSum());
        System.out.println("平均数 : " + stats.getAverage());

        /*
         * match匹配
         * match分三种
         * 返回一个bool值
         */
        System.out.println("\nStream match:");
        boolean allMatch = list.stream().allMatch(s -> s.contains("这里"));
        boolean anyMatch = list.stream().anyMatch(s -> s.contains("这里"));
        boolean noneMatch = list.stream().noneMatch(s -> s.contains("这里"));
        System.out.println(allMatch);
        System.out.println(anyMatch);
        System.out.println(noneMatch);

    }
}

```
打印结果为
```
Stream forEach:
这里
有

许多
字符串
还有
一个
空字符串

ParallelStream forEach:
还有
字符串

一个
有
许多
空字符串
这里

Stream map:
这里*
有*
*
许多*
字符串*
还有*
一个*
空字符串*

Stream filter:
字符串
空字符串

Stream limit:
这里
有

许多

Stream sorted:

有
这里
许多
还有
一个
字符串
空字符串

Stream sorted:
[这里, 有, 许多, 字符串, 还有, 一个, 空字符串]
=>这里有许多字符串还有一个空字符串<=

Stream statistics:
列表中最大的数 : 7
列表中最小的数 : 2
所有数之和 : 25
平均数 : 3.5714285714285716

Stream match:
false
true
true
```
