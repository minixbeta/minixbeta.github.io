---
layout: post
title: 生产环境 JDK6 升级 JDK8
category: 技术
comments: true
---

由于 Oracle 已经不对  JDK6 和 JDK7 进行支持，同时为了利用 G1 收集器。所以我们在生产环境中，将项目从 JDK6 升级至 JDK8，并将垃圾收集器由 CMS 换成了 G1。下面对这次升级作一个总结，并且给出一些大家可能需要用到的资源。


## 升级指引
升级前首先需要了解一下 Oracle 对 JDK 各个版本的支持时间，JDK6, JDK7 分别于 2013, 2015 年停止了公开更新(public update)，而 JDK8 将于 2017 年，或者更晚的时间，停止公开更新，详细内容可以参考 [Oracle Java SE Support Roadmap](http://www.oracle.com/technetwork/java/eol-135779.html)。

Java 版本之间是有二进制向后兼容性的，即 JDK8 的虚拟机可以运行由 JDK7, JDK6 编译的 class 文件。如果有例外的话，Oracle 会在升级中明确说明。但是如果用 JDK8 了，还将代码编译成之前版本的类文件，就意味着无法使用 JDK8 中的新语法特性。Oracle 在大版本之间的升级，都会提供兼容性指南，例如 [Compatibility Guide for JDK 8](http://www.oracle.com/technetwork/java/javase/8-compatibility-guide-2156366.html)，[Java SE 7 and JDK 7 Compatibility](http://www.oracle.com/technetwork/java/javase/compatibility-417013.html)。兼容性指南包含的主要内容包括：

 * 二进制兼容性
 * 源代码兼容性
 * 行为兼容性
 * 类文件变化
 * Java SE 与上一版本相比不兼容的地方
 * JDK 与上一版本相比不兼容的地方
 * Java SE 中移除的特性
 * JDK 中移除的特性
 * 不建议使用的 API


###  二进制兼容性
JDK7 编译的代码可以在 JDK8 下运行，JDK8 下编译的代码不可以在之前版本的 JDK 中运行 。

### 源代码兼容性
使用了 JDK8 语法新特性的源代码，不能在之前版本的 Java 平台上编译。不建议使用的 API 在用新 `javac` 编译时，会给出警告，除非通过 `-nowarn` 关闭。`sum.*` 下的 `API` 有所变化，所以，一定要记得不要使用这个包下的 `API`，不然升级会很痛苦。

### 行为兼容性
行为兼容性意味着同样的输入，有着同样的行为。Java 平台中会有一些故意未指定的行为，在不同的 JDK 版本中，可能会有变化，所以代码不应该依赖于这些行为。如果 JDK 版本变了，行为也变了，可能说明代码里有 bug。

### 类文件
JDK8 编译的类文件，版本号为 `52.0`。

### JDK 及 Java SE 与上一版本的不兼容性
这一部分内容比较多，如果从 JDK7 升级至 JDK8，可以参考 [Compatibility Guide for JDK 8](http://www.oracle.com/technetwork/java/javase/8-compatibility-guide-2156366.html)；如果从 JDK6 升级至 JDK8，需要同时参考 [Compatibility Guide for JDK 8](http://www.oracle.com/technetwork/java/javase/8-compatibility-guide-2156366.html) 和 [Java SE 7 and JDK 7 Compatibility](http://www.oracle.com/technetwork/java/javase/compatibility-417013.html)。

这一部分内容里包含了源代码不兼容性，例如原来在 JDK7 下可以编译的代码，在 JDK8 下就不能编译了。这类错误还是比较好查的，编译一下就可以找出这类错误。

或者是一些 GC 选项会被忽略，例如 `PermSize`。

或者是类文件中添加了一些新内容，例如 StackMapTable ，在运行时会对类文件的这个字段进行验证，一些可能会调整字节码的工具，需要更新这个字段。例如代码覆盖率统计工具，这些工具如果调整了字节码，又没有对 StackMapTable 进行更新。那么高版本的 JDK 就无法运行经过调整的字节码，在字节码校验阶段就会失败。

或者是同一份代码，由于 JDK8 引入的特性，生成不同字节码，最终导致 JDK8 下无法正常运行。这类错误通常只会在运行时才体现出来，比较难以察觉。例如我在这篇 [JDK8 中的类型推断与重载解析](http://blog.csdn.net/on_1y/article/details/50650014) 中描述的这类情况。对这类情况，我们应该做到：

 * 平时多写测试用例
 * 用 tcpcopy 引线上流量打压

这样，可以让运行时错误或者行为不兼容性更早地暴露出来
 
## GC
从 JDK6 向上升级，在 GC 方面比较重要的就是 G1 这个垃圾收集器。G1 垃圾收集器在 JDK6u14 中作为实验特性发布，在 JDK7 中正式发布。未来，在 JDK9 中，G1 将会作为默认的垃圾收集器，可见 G1 已经达到了比较成熟的阶段。所以在 JDK8 中完全可以使用 G1。

而 G1 中比较重要的特性包括：

 * 同时实现了新生代和老年代两种 GC 算法 
 * 各代之间不再有物理隔离，虚拟机管理的内存分为多个 region
 * 通过设置 GC 停顿的期望时间来调优(`-XX:MaxGCPauseMillis`)
 
使用建议包括：

 * 适用于管理大内存的场景
 * 适用于对延迟要求高于吞吐的场景
 * 总是将详细的 GC 日志记录在 log 中
 * 总是打开 `-XX:+ParallelRefProcEnabled` 选项，实际环境中发现处理引用的过程确实也比较耗时
 * 避免 Humongous 区域，即单个对象大小大于 region 区大小的 50%
 * 尽力避免 Full GC

注意事项包括：
 * 不要再设置新生代大小了，通过调整停顿时间，G1 会自动调整新生代大小
 * 停顿时间只是期望时间，并不保证 STW 一定在这个时间内完成。根据经验，如果将停顿时间设置为 x, 那么 50% 的 GC 会在 x ms 内完成
 

在此次升级过程中，我看过的最重要的资料是一份 presentation: [G1 Garbage Collector
Details and Tuning](http://presentations2015.s3.amazonaws.com/40_presentation.pdf)。其中对 G1 中的重要特性，原理，使用建议，参考资料进行了说明，强烈建议阅读。

除此之外，还有几个和 G1 相关的博客可以参考:

 * [G1GC Concept and Performance Tuning](https://blogs.oracle.com/g1gc/)
 * [Poonam's Weblog](https://blogs.oracle.com/poonam/tags/g1)
 
## 第三方库
升级 JDK8 的过程中，一些第三方库也需要同步更新。比较重要的一个原因在于，一些库是依赖类文件格式的，如果把代码编译到 JDK8 的版本，那么这些库也需要同步更新。例如：

  * 代码覆盖率工具：Emma 不支持 JDK8，可以选用 Jacoco
  * Spring: 4.x 版本才全面支持 JDK8，之前的版本可能无法正确读取类文件。可以参考 [How Spring achieves compatibility with Java 6, 7 and 8](https://spring.io/blog/2015/04/03/how-spring-achieves-compatibility-with-java-6-7-and-8)。

同时，也需要注意，第三方库的升级，也会带来另外一些问题。例如，我们在升级 Spring 的过程中，就遇到了死锁问题。
  
## 参考资料

* [Upgrading major Java versions](https://blogs.oracle.com/java-platform-group/entry/upgrading_major_java_versions)
* [Compatibility Guide for JDK 8](http://www.oracle.com/technetwork/java/javase/8-compatibility-guide-2156366.html)
* [Java SE 7 and JDK 7 Compatibility](http://www.oracle.com/technetwork/java/javase/compatibility-417013.html)
* [G1 Garbage Collector
Details and Tuning](http://presentations2015.s3.amazonaws.com/40_presentation.pdf)
* [Spring Framework 4.0 M1 & 3.2.3 available](http://spring.io/blog/2013/05/21/spring-framework-4-0-m1-3-2-3-available/)




