---
layout:     post
title:      "Java基础知识总结"
subtitle:   " \"String，Java类加载\""
date:       2019-07-28 16:30:00
author:     "Clinat"
header-img: "img/post-bg-kuaidi.jpg"
catalog: true
tags:
    - Java
---

> “String字符串以及类加载的相关内容”

## String,StringBuffer和StringBuilder

**StringBuffer**是**线程安全**的，而且对字符串的修改是直接作用在对象本身的。

**StringBuilder**：字符串变量（**非线程安全**）。在内部，StringBuilder对象被当作是一个包含字符序列的变长数组。是StringBuffer的简易替换。

**String**是**字符串常量，字符串的长度是不可变的**，每次对String类型进行改变时，都会产生一个新的String对象，然后将指针指向新的String对象，因此经产改变字符串内容时最好不要使用String类型，因为每次生成对象都会对系统性能产生影响，特别当内存中无引用对象多了以后， JVM 的 GC 就会开始工作，性能就会降低。

**对于String类型的变量，使用‘+’来连接字符串的时候，如果连接的是字符串常量，例如“hello”，Java编译器会直接将连接操作编译成整个的字符串，而对于字符串变量的连接操纵，Java编译器转换成StringBuffer，然后使用append方法进行拼接。**



**对于字符串String：**

（1）String s=“Hello World”

​		  String w=“Hello World”

​		**s == w**  返回true

 **(因为对于String常量，数据是存放在常量池中，所以s和w是指向同一个地址，由于==是比较指向的地址是否相同，所以返回的为true)**

（2）String s = new  String( "Hello World" );

​		  String w = new  String( "Hello World" );

​        **s == w**  返回false

**(使用new创建的对象是存放在堆中的，两次new创建的对象不是同一个对象，所以s和w指向的地址是不同的，所以返回false)**

（3）	String a =  "tao" + "bao" ;

​			  String b =  "tao" ;

​			  String c =  "bao" ;

​			  (b+c) == a 返回false

​		**(由于java编译器的优化，“tao”+”bao”会被直接编译为“taobao”常量，并且存放在常量池中，而对于b和c，“tao”和“bao”也会存放在常量池中，但是对于(b+c)，编译器会转化成StringBuffer，然后调用append进行编译，组合后的数据存放在堆中，所以地址是不同的，所以返回false)**

（4）	String a =  "tao" + "bao" ;

​			  final String b =  "tao" ;

​			  final String c =  "bao" ;

​			  (b+c)== a 返回true

​		**(由于b和c前面有final关键字修饰，也就是说该变量是不能被修改的，所以编译器会认为(b+c)是不能不修改的，所以会将组合后的数据存放在常量池中，作为常量，由于栈中的数据是共享的，所以指向的地址和a是相同的，所以返回true)**



## 类的加载顺序

以下内容参考《深入理解Java虚拟机》

Java文件从被加载到被卸载一共分为5个阶段：**加载，链接（验证，准备，解析），初始化，使用，卸载**

(1)加载：通过类的全限定名来获得定义此类的二进制字节流，将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构，在内存中生成一个代表这个类的Class对象，作为方法区这个类的各种数据的访问入口。

(2)验证：验证是链接阶段的第一步，确保类正确被加载，进行一下验证工作。

(3)准备：正式为静态变量分配内存并设置静态变量的初始值，注意该阶段不是分配程序中定义的值，而是初始值，例如0。

(4)解析：该阶段是将符号引用转换为直接引用的阶段。

(5)初始化：为静态变量分配程序中定义的值(准备阶段已经分配过一次初始值)。



**创建实例时初始化顺序：**

父类静态代码块(静态初始化块，静态属性，但不包括静态方法) -> 子类静态代码块(静态初始化块，静态属性，但不包括静态方法) -> 父类非静态代码块(非静态代码块，非静态属性) -> 父类构造函数 -> 子类非静态代码块(非静态代码块，非静态属性) -> 子类构造函数。



## 类加载器

类加载器包括：**根加载器（** **BootStrap** **）、扩展加载器（** **Extension** **）、系统加载器（** **System** **）和用户自定义类加载器（** **java.lang.ClassLoader** **的子类）**。

从 Java 2 （ JDK 1.2 ）开始，类加载过程采取了**父亲委托机制（** **PDM** **）**。

PDM 更好的保证了 Java 平台的安全性，在该机制中， JVM 自带的 Bootstrap 是根加载器，其他的加载器都有且仅有一个父类加载器。类的加载首先请求父类加载器加载，父类加载器无能为力时才由其子类加载器自行加载。 JVM 不会向 Java 程序提供对 Bootstrap 的引用。

1. **Bootstrap** ：一般用本地代码实现，负责加载 JVM 基础核心类库（rt.jar）
2. **Extension** ：从 java.ext.dirs 系统属性所指定的目录中加载类库，它的父加载器是 Bootstrap
3. **system class loader** ：又叫应用类加载器，其父类是 Extension 。它是应用最广泛的类加载器。它从环境变量 classpath 或者系统属性 java.class.path 所指定的目录中记载类，是用户自定义加载器的默认父加载器。
4. **用户自定义类加载器**： java.lang.ClassLoader 的子类



**JVM判断两个类相同的依据：**类的全称，类加载器





## 其他

**（1）Class.forName方法和ClassLoader.loadClass方法区别**

Class.forName默认执行初始化过程，ClassLoader.loadClass默认不执行初始化过程

**（2）JVM判断两个类相同的依据：**类的全称，类加载器














