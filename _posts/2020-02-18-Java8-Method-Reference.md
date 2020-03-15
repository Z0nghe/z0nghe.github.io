---
layout: post
title:  "Java8新特性（二）方法引用(Method Reference)"
date:   2020-02-18 21:58:11 +0800
categories: java java8 "Method Reference"
abstract: "在了解过Java8的lambda表达式之后，我第一次接触到了方法引用，当时对这种写法很疑惑，因为我对lambda表达式也不是很熟悉。今天就来认真的学习一下方法引用。"
---

### 一、前言
在了解过Java8的lambda表达式之后，我第一次接触到了方法引用，当时对这种写法很疑惑，因为我对lambda表达式也不是很熟悉。今天就来认真的学习一下方法引用。

### 二、什么是lambda表达式
> Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。
Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。
使用 Lambda 表达式可以使代码变的更加简洁紧凑。

lambda表达式是一种匿名函数，可以直接把对参数的操作通过箭头符号 `->` 直接指向方法体。

### 三、什么是方法引用
方法引用是函数式编程思想下的一种用已有方法(函数)作为参数传递给lambda表达式执行的一种方式。方法引用提供了一种引用而不执行方法的方式，它需要由兼容的函数式接口构成的目标类型上下文。

### 四、使用方法
方法引用的语法是通过 `::` 连接类或对象和方法的。
方法引用分为7种：
1. 类的构造方法引用 Class::new
2. 类的静态方法引用 Class::staticMethod
3. 对象的实例方法引用 instance::method
4. 类的任意方法引用 Class::method
5. 数组引用 Class[]::new
6. 父类方法引用 super::method
7. this获取指针引用 this::method

#### 1、类的构造方法引用

```java
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

public class App {
    public static void main(String[] args) {
        //lambda表达式形式的创建
        StudentCreate1 create1 = () -> new Student();
        StudentCreate2 create2 = (String name) -> new Student(name);
        StudentCreate3 create3 = (String name, int age) -> new Student(name, age);

        Student student1 = create1.getStudent();
        Student student2 = create2.getStudent("李雷");
        Student student3 = create3.getStudent("韩梅梅", 18);

        System.out.println(student1);
        System.out.println(student2);
        System.out.println(student3);

        //构造函数引用
        StudentCreate1 create4 = Student::new;
        StudentCreate2 create5 = Student::new;
        StudentCreate3 create6 = Student::new;

        Student student4 = create4.getStudent();
        Student student5 = create5.getStudent("李雷");
        Student student6 = create6.getStudent("韩梅梅", 18);

        System.out.println(student4);
        System.out.println(student5);
        System.out.println(student6);

        //可以通过流式处理引用构造函数
        List<String> nameList = new ArrayList<>();
        nameList.add("李雷");
        nameList.add("韩梅梅");
        List<Student> students = nameList.stream().map(Student::new).collect(Collectors.toList());
        System.out.println(students);
    }
}

//函数式接口，注解可以不添加
@FunctionalInterface
interface StudentCreate1 {
    Student getStudent();
}

@FunctionalInterface
interface StudentCreate2 {
    Student getStudent(String name);
}

@FunctionalInterface
interface StudentCreate3 {
    Student getStudent(String name, int age);
}

//@FunctionalInterface 由于存在多个抽象方法打开注解后会报错
interface StudentCreate4 {
    Student getStudent();
    Student getStudent(String name);
    Student getStudent(String name, int age);
}

class Student {
    private String name;
    private int age;

    public Student(String name) {
        this.name = name;
        this.age = 0;
    }

    public Student() {
        this.name = "";
        this.age = 0;
    }

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

@FunctionalInterface是Java8中为了配合lambda表达式增加的注解，而且在Runnable，Comparator等常用接口也增加了@FunctionalInterface注解。

在源码中关于@FunctionalInterface的描述：

> Conceptually, a functional interface has exactly one abstract method.
从概念上讲，函数接口只有一个抽象方法。

> Note that instances of functional interfaces can be created with lambda expressions, method references, or constructor references.
注意，可以使用lambda表达式、方法引用或构造函数引用创建函数接口的实例。

> If an interface declares an abstract method overriding one of the public methods of {@code java.lang.Object}, that also does <em>not</em> count toward the interface's abstract method count since any implementation of the interface will have an implementation from {@code java.lang.Object} or elsewhere.
如果接口声明了一个抽象方法重写了Object的public方法，则不会计入抽象方法。

> However, the compiler will treat any interface meeting the definition of a functional interface as a functional interface regardless of whether or not a FunctionalInterface annotation is present on the interface declaration.
但是，不管接口声明上是否存在FunctionalInterface注解，编译器都将满足函数接口定义的任何接口视为函数接口。

从源码中的描述看，函数式接口有且只有一个抽象方法。在上面代码函数式接口方法的实现则是引用了Student类的构造方法。在流式处理中可以简洁的批量创建Student类。

#### 2、类的静态方法引用

```java
import java.util.function.Function;

public class App {
    public static void main(String[] args) {
        //jdk1.8提供的函数式接口 Function<T, R> T为输入参数类型 R为输出参数类型
        Function<String, String> function = Student::print;
        System.out.println(function.apply("李雷"));

        //自定义函数式接口 其中的抽象函数参数列表和返回值要与print方法相同
        StudentInfo1 info1 = Student::print;
        System.out.println(create1.printNameAndAge("韩梅梅", 18));

        //定义泛型的函数式接口
        StudentInfo2<String, Integer, String> info2 = Student::print;
        System.out.println(create2.printNameAndAge("李雷", 19));
    }
}

@FunctionalInterface
interface StudentInfo1 {
    String printNameAndAge(String name, int age);
}

@FunctionalInterface
interface StudentInfo2<T1, T2, R> {
    R printNameAndAge(T1 name, T2 age);
}

class Student {
    public static String print(String name) {
        return "name: " + name;
    }

    public static String print(String name, int age) {
        return "name: " + name + " age: " + age;
    }
}
```

#### 3、对象的实例方法引用

```java
public class App {
    public static void main(String[] args) {
        StudentCreate<String, Integer, Student> studentCreate = Student::new;
        Student student = studentCreate.getStudent("李雷", 19);

        //由于参数列表相同Print可以绑定不同方法
        PrintInfo printInfo1 = student::printName;
        PrintInfo printInfo2 = student::printNameAndAge;
        System.out.println(printInfo1.apply());
        System.out.println(printInfo2.apply());
    }
}

@FunctionalInterface
interface StudentCreate<T1, T2, R> {
    R getStudent(T1 name, T2 age);
}

@FunctionalInterface
interface PrintInfo {
    String apply();
}

class Student {
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String printName() {
        return this.name;
    }

    public String printNameAndAge() {
        return "name: " + this.name + " age: " + this.age;
    }
}
```

#### 4、类的任意方法引用

```java
public class App {
    public static void main(String[] args) {
        StudentCreate<String, Integer, Student> studentCreate = Student::new;
        Student student = studentCreate.getStudent("李雷", 19);

        //由于参数列表都为空所以可以引用不同的方法
        PrintInfo<Student, String> printInfo1 = Student::printName;
        PrintInfo<Student, String> printInfo2 = Student::printNameAndAge;
        System.out.println(printInfo1.apply(student));
        System.out.println(printInfo2.apply(student));
    }
}

@FunctionalInterface
interface StudentCreate<T1, T2, R> {
    R getStudent(T1 name, T2 age);
}

@FunctionalInterface
interface PrintInfo<T1, R> {
    R apply(T1 student);
}

class Student {
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String printName() {
        return this.name;
    }

    public String printNameAndAge() {
        return "name: " + this.name + " age: " + this.age;
    }
}
```

#### 5、数组引用

```java
public class App {
    public static void main(String[] args) {
        Function<Integer, String[]> f1 = String[]::new;
        String[] strs = f1.apply(10);
        System.out.println(strs.length);
    }
}
```

#### 6、父类方法引用
使用 `super` 关键字对父类方法进行引用**super**::method

#### 7、this获取指针引用
使用 `this` 关键字对当前对象的方法进行引用**this**::method

### 五、Java8提供的函数式接口

|接口|描述|
|:--|:--|
|BiConsumer<T,U>|代表了一个接受两个输入参数的操作，并且不返回任何结果。|
|BiFunction<T,U,R>|代表了一个接受两个输入参数的方法，并且返回一个结果。|
|BinaryOperator\<T\>|代表了一个作用于于两个同类型操作符的操作，并且返回了操作符同类型的结果。|
|BiPredicate<T,U>|代表了一个两个参数的boolean值方法。|
|BooleanSupplier|代表了boolean值结果的提供方。|
|Consumer\<T\>|代表了接受一个输入参数并且无返回的操作。|
|DoubleBinaryOperator|代表了作用于两个double值操作符的操作，并且返回了一个double值的结果。|
|DoubleConsumer|代表一个接受double值参数的操作，并且不返回结果。|
|DoubleFunction\<R\>|代表接受一个double值参数的方法，并且返回结果。|
|DoublePredicate|代表一个拥有double值参数的boolean值方法。|
|DoubleSupplier|代表一个double值结构的提供方|
|DoubleToIntFunction|接受一个double类型输入，返回一个int类型结果。|
|DoubleToLongFunction|接受一个double类型输入，返回一个long类型结果。|
|DoubleUnaryOperator|接受一个参数同为类型double,返回值类型也为double 。|
|Function<T,R>|接受一个输入参数，返回一个结果。|
|IntBinaryOperator|接受两个参数同为类型int,返回值类型也为int。|
|IntConsumer|接受一个int类型的输入参数，无返回值。|
|IntFunction\<R\>|接受一个int类型输入参数，返回一个结果。|
|IntPredicate|接受一个int输入参数，返回一个布尔值的结果。|
|IntSupplier|无参数，返回一个int类型结果。|
|IntToDoubleFunction|接受一个int类型输入，返回一个double类型结果。|
|IntToLongFunction|接受一个int类型输入，返回一个long类型结果。|
|IntUnaryOperator|接受一个参数同为类型int,返回值类型也为int。|
|LongBinaryOperator|接受两个参数同为类型long,返回值类型也为long。|
|LongConsumer|接受一个long类型的输入参数，无返回值。|
|LongFunction\<R\>|接受一个long类型输入参数，返回一个结果。|
|LongPredicate\<R>|接受一个long输入参数，返回一个布尔值类型结果。|
|LongSupplier|无参数，返回一个结果long类型的值。|
|LongToDoubleFunction|接受一个long类型输入，返回一个double类型结果。|
|LongToIntFunction|接受一个long类型输入，返回一个int类型结果。|
|LongUnaryOperator|接受一个参数同为类型long,返回值类型也为long。|
|ObjDoubleConsumer\<T>|接受一个object类型和一个double类型的输入参数，无返回值。|
|ObjIntConsumer\<T>|接受一个object类型和一个int类型的输入参数，无返回值。|
|ObjLongConsumer\<T>|接受一个object类型和一个long类型的输入参数，无返回值。|
|Predicate\<T>|接受一个输入参数，返回一个布尔值结果。|
|Supplier\<T>|无参数，返回一个结果。|
|ToDoubleBiFunction <T,U>|接受两个输入参数，返回一个double类型结果。|
|ToDoubleFunction\<T>|接受一个输入参数，返回一个double类型结果。|
|ToIntBiFunction<T,U>|接受两个输入参数，返回一个int类型结果。|
|ToIntFunction\<T\>|接受一个输入参数，返回一个int类型结果。|
|ToLongBiFunction<T,U>|接受两个输入参数，返回一个long类型结果。|
|ToLongFunction\<T\>|接受一个输入参数，返回一个long类型结果。|
|UnaryOperator\<T\>|接受一个参数为类型T,返回值类型也为T。|

