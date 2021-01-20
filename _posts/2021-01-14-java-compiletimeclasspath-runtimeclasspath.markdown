---
layout: post
title:  "Java compile time dependency vs runtime dependency"
date:   2021-01-14 22:57:36 +0530
categories: Java 
---
I have recently reading the **api vs implementaion** configuration of gradle [here][gradle-apiimple]. Then I thought that it will be very useful to create  some simple java programs which can easily explain the difference between compile time depencies and run time depencies. For that purpose, I have created some dummy
classes just to intoroduce the concepts.

[gradle-apiimple]: https://tomgregory.com/how-to-use-gradle-api-vs-implementation-dependencies-with-the-java-library-plugin/

Create a folder  **b** and create two classes **B.java** and **C.java** inside it


```java
# source code of B.java #
package b;
  
public class B {
  public String sayHello() {
  C c = new C();
  return "Hello world!"+c.inc(10);
  }
}
```
```java
# source code of C.java #
package b;
public class C {
 public String inc(int i) {
    return i+1+"";
   }

```

Create a folder  **hello** and create class **Greeter.java** inside it.


```java
# source code of Greeter.java #
package hello;
import b.B;
public class Greeter {
  public static void main (String []args) {
   B b = new B();
   System.out.println( b.sayHello());
  }
}
```

You can see that **B.java** depends on **C.java** and **Greeter.java**.

Now if we compile **B.java** by ```javac b/B.java```  then you can see that two class files **B.class** and **C.class** were created inside **b**.
Then if we compile **Greeter.java** by ```javac  hello/Greeter.java``` then you can see that **Greeter.class** was created inside **hello**.

Then if we run by ```java hello.Greeter``` then  it will print **Hello world!11**

Now we will create a new folder **f** and will move **C.class** from **b** to **f**.
Then if we recompile ```javac  hello/Greeter.java```  then we can see that it's compiling suceessfully but if we run **Greeter**
 ```java hello.Greeter``` then we can see some run time error.

``` java
Exception in thread "main" java.lang.NoClassDefFoundError: b/C
	at b.B.sayHello(B.java:5)
	at hello.Greeter.main(Greeter.java:6)
Caused by: java.lang.ClassNotFoundException: b.C
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
```
 This happened because **B.class** needs **C.class** to run but at the time 
 of compiling **Greeter.java**, it wont throw compilation error because **Greeter.java** depends only on **B.class** and that's avaialble in **b**.
 But at run time  **Greeter.class** instance invokes **B.class**  but **B.class** needs **C.class** for running but it can not find **C.class** in the directory ( in the provided class path).
 That's why it's failing. Therefore to make it working, we will add  **f/C.class** in the runtime class path as
 ```java -cp f:.  hello.Greeter```and you can see that it's printing *** Hello world!11*** as expected
 
