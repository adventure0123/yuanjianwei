---
title: java class
date: 2016-06-28 23:27:54
tags: [java,Class.forName]
---

本文主要介绍Class.forName()的用法和new关键字的区别。<!--more-->首先class.forName()是一个方法，返回一个calss类，Class.forName(“”).newInstance()返回的是object。new是一个关键字，返回一个对象。其次，class下的newInstance()生成的对象只能是无参的构造函数，而new关键字对生成的对象没有限制。newInstance()：弱类型，效率低，只能调用无参构造函数，new：强类型，相对高效，能调用任何public构造。

#### 一：Java中的任何class都需要装载到虚拟机上才可以运行

forName这句话就是装载类使用的，new是根据加载到内存中的类创建一个实例。给定一个字符串变量，表示一个类的包名和类名，以下两种创建方式是等价的。
``` java
A a=(A)Class.forName("package.A").newInstacne();
A a=new A();
```
jvm在装载类时会执行类的静态代码段，静态代码段和class和绑定的，一旦class装载成功就表示执行了你的静态代码段，以后不会再执行这段静态代码了。Class.forName("")的作用是要求JVM查找并加载指定的类，也就说JVM会执行该类的静态代码段。Class.forName()可以动态加载和创建class对象，比如根据用户输入的字符串对象来创建对象。
```java
String s=输入的字符串;
Class c=Class.forName(s);
c.newInstance();
```

#### 二：生成一个实例的时候，newInstance()方法和new关键字的主要区别

他们的主要的区别是创建对象的方法不一样，前者使用类加载机制，后者创建一个新类。之所以有两种创建方法，主要是考虑到软件的可伸缩性，可扩展性和可重用性等设计思想。Java中的工厂模式经常使用newInstance()方法来创建对象。例如：
```java
String className="example"
class c=Class.forName(className);
t=(exampleInterface)c.newInstance();
```
进一步的可以根据配置文件生成对象，例如：
```java 
String className=XMLConfig;
class c=Class.forName(className);
t=(exampleInterface)c.newInstance();
```
上面的代码已经不存在具体的类型，它的优点在于不管example类怎么变化，只要他们的接口不变，上诉的代码就不需要变，这也符合Java面向接口的思想。
从JVM的角度看，使用new关键字创建一个类的时候，这个类可以没有被加载，但是使用newInstance的时候，就必须保证这个类已经被加载并且已经连接。上面两个步骤正式Class.forName()所完成的，这个静态方法启动了类加载器。由此可以看到，newInstance实际上把new这个过程分成了两步，首先调用class加载方法加载某个类，然后实例化。这样做的好处是，我们在调用Class.forName()时获得了更好的灵活性，提供了一种降低耦合的手段。

#### 三：使用场景

* 1：加载数据库驱动
```java
Class.forName("com.microsoft.sqlserver.jdbc.SQLServerDriver");  
Connection con=DriverManager.getConnection("jdbc:sqlserver://localhost:1433;DatabaseName==JSP","jph","jph");      
```
为什么有的jdbc连接数据库的写法是Class.forName(“”)，而有的是Class.forName(“”).newInstance()？上面我们提到Class.forName(“”)的作用是要求JVM查找并加载指定的类，如果在类中有静态初始化器的话，JVM必定会执行该类的静态代码段。而在JDBC的规范中明确要求这个Driver类必须要想DriverManager注册自己，即一个Driver类都类似于:
```java
  public class MyJDBCDriver implements Driver { 
    static { 
       DriverManager.registerDriver(new MyJDBCDriver()); 
    } 
  }

```
因为已经在静态初始化器中进行了注册，因此我们只需要调用Class.forName(）方法就可以使用了。

* 2：使用AIDL与电话管理Servic进行通信

