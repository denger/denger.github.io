---
layout: post
title: Java 常用类原理分析文章
---

如今，Java 基础方面的面试问题经常会涉及到常用类的实现原理，好在这方面已经有不少文章，这里收集一些分析不错的，并会持续更新。

#### Map

- [Java HashMap 工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/) - *非常全面，图文并茂，清晰易懂*
- [HashMap 的工作原理](http://www.importnew.com/7099.html) - *Q&A 方式解答工作原理，没准面试会碰到*
- [疫苗：Java HashMap 的死循环](http://coolshell.cn/articles/9606.html)  - *HashMap 在并发场景使用会导致的问题及原因分析*
- [深入分析 ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap) - *ConcurrentHashMap 如何解决 HashMap 线程问题的必读文章*
- [Java TreeMap工作原理及实现](http://yikun.github.io/2015/04/06/Java-TreeMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [通过分析JDK 源代码研究TreeMap 红黑树算法实现](https://www.ibm.com/developerworks/cn/java/j-lo-tree/)
- [Java WeakHashMap 源码解析](http://liujiacai.net/blog/2015/09/27/java-weakhashmap/)  - 与 GC 的那点事

#### List

- [ArrayList 源码分析（基于JDK1.6）](http://www.cnblogs.com/hzmark/archive/2012/12/20/ArrayList.html) - *深入剖析 ArrayList 的内部结构及实现原理*
- [Java 中的 CopyOnWrite 容器](http://coolshell.cn/articles/11175.html) - *如何解决 ArrayList 线程安全问题*
- [Java LinkedList工作原理及实现](http://yikun.github.io/2015/04/05/Java-LinkedList%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/) - *了解 LinkedList 内部存储结构*

#### Set

- [Java HashSet工作原理及实现](http://yikun.github.io/2015/04/08/Java-HashSet%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [TreeSet详细介绍(源码解析)和使用示例](http://www.cnblogs.com/skywang12345/p/3311268.html)


#### Queue

- [Java ArrayDeque 工作原理及实现](http://yikun.github.io/2015/04/11/Java-ArrayDeque%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/) - *简单的介绍了大致的工作原理*

#### Thread

- [正确理解 ThreadLocal](http://www.iteye.com/topic/103804) - 从源码的角度剖析 ThreadLocal
- [Java线程池架构：原理和源码解析](https://segmentfault.com/a/1190000000394999) - JUC 中提供的 ThreadPoolExecutor 源码分析

#### Reflect
- [Java 反射原理简析](http://www.fanyilun.me/2015/10/29/Java%E5%8F%8D%E5%B0%84%E5%8E%9F%E7%90%86/) - 反射实现原理入门文章
- [代理模式剖析](https://github.com/biezhi/java-bible/blob/master/designpatterns/proxy.md) - JDK 动态代理是如何实现"动态"的？
