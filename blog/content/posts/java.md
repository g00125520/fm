---
title: "Java"
date: 2018-04-23T19:38:26+08:00
draft: true
tags:
 - java
---

函数式接口只定义了唯一的抽象方法的接口（除了隐含的Object对象的公共方法）,也叫做SAM类型的接口（Single Abstract Method）。利用SAM 接口作为 Lambda表达式的目标类型。

#### ref
- [Java 8函数式接口functional interface的秘密](http://colobu.com/2014/10/28/secrets-of-java-8-functional-interface/#JDK_8%E4%B9%8B%E5%89%8D%E5%B7%B2%E6%9C%89%E7%9A%84%E5%87%BD%E6%95%B0%E5%BC%8F%E6%8E%A5%E5%8F%A3)
- [深入浅出 Java 8 Lambda 表达式](http://blog.oneapm.com/apm-tech/226.html)

jstat -gcutil 4296 1000，检测jvm的内容，y和o的默认比例为82。

jmap -dump:live,format=b,file=heap.hprof 23436

父类中的方法声明了synchronized，但是在子类中却不能实现同步，因为，每个子类都拷贝了一份，每个子类在运行自己的方法，要想在调用父类方法的时候实现同步，需将父类方法声明为static。变量也是，每个子类会拷贝一份自己的变量，只有static的变量才是共享的。

序列化用于将类保存为文件或者在网络传输，序列化id类似版本号，防止类的不同版本。类的静态属性和transient属性不会被序列化。类在序列化时要求所有的属性都必须是可序列化的。

stream

