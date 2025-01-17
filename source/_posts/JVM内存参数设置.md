---
title: JVM内存参数设置
date: 2022-09-14 16:02:28
tags:
- 性能调优
categories: JVM调优
---

# **JVM内存参数设置**

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h664sj34uvj20tc0igq4p.jpg" style="zoom:50%;" />

## JVM参数设置格式

Spring Boot程序的JVM参数设置格式(Tomcat启动直接加在bin目录下catalina.sh文件里)：

```java
java ‐Xms2048M ‐Xmx2048M ‐Xmn1024M ‐Xss512K ‐XX:MetaspaceSize=256M ‐XX:MaxMetaspaceSize=256M ‐jar microservice‐eureka‐server.jar 
```

## 元空间的JVM参数

- 关于元空间的JVM参数有两个：-XX:MetaspaceSize=N和 -XX:MaxMetaspaceSize=N

  - **-XX：MaxMetaspaceSize**： 设置元空间最大值， 默认是-1， 即不限制， 或者说只受限于本地内存大小。 

  - **-XX：MetaspaceSize**： 指定元空间触发Fullgc的初始阈值(元空间无固定初始大小)， 以字节为单位，默认是21M，达到该值就会触发full gc， 同时收集器会对该值进行调整： 如果释放了大量的空间， 就适当降低该值； 如果释放了很少的空间， 那么在不超过-XX：MaxMetaspaceSize（如果设置了的话） 的情况下， 适当提高该值。（比如初始值是21M，full gc后，释放了20M，只剩1M，那么垃圾收集器就会将该值调成2M；相反，如果释放了1M，还剩20M，下次就会调整到40M）

- 由于调整元空间的大小需要Full GC，这是非常昂贵的操作，如果应用在启动的时候发生大量Full GC，通常都是由于永久代或元空间发生 了大小调整，基于这种情况，一般建议在JVM参数中将MetaspaceSize和MaxMetaspaceSize设置成一样的值，并设置得比初始值要大。

- 对于8G物理内存的机器来说，一般我会将这两个值都设置为256M。常见的问题：项目jar包只有1G，但是启动了好久，就可能是因为由于元空间设置的太小，一直在触发full GC。

## 栈的JVM参数

Xss设置越小，说明一个线程栈里能分配的栈帧就越少，但是对JVM整体来说能开启的线程数会更多 

**StackOverflowError**示例：

```java
public class StackOverflowTest {
    
    static int count = 0;
    
    static void redo() {
        count++;
        redo();
    }

    public static void main(String[] args) {
        try {
            redo();
        } catch (Throwable t) {
            t.printStackTrace();
            System.out.println(count);
        }
    }
}
```

运行代码：

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h6661qdwhlj20y00don0m.jpg" style="zoom:50%;" />

调整-Xss参数：

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h666103f0cj214c0kmdhs.jpg" style="zoom:50%;" />

打印结果：

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h665zbm64qj21160e4q6p.jpg" style="zoom:50%;" />
