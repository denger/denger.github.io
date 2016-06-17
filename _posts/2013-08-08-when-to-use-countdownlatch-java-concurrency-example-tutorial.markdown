---
layout: post
title: 何时使用 CountDownLatch：Java 并发编程实例
---

根据 Java Docs 中的说明，[CountDownLatch](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html) 一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。CountDownLatch 的概念在 Java 并发方面的面试题中非常常见。所以你要确保对它有所了解。在该文章中，我将介绍 CountDownLatch 在 Java 并发方面的以下几点内容：

* 什么是 CountDownLatch ?
* CountDownLatch 的工作机制？
* 在实际应用程序可能用途
* 示例程序
* 常见的面试问题


<h4>什么是 CountDownLatch ?</h4>
CountDownLatch 在 JDK1.5 中开始引入，同时引入类似的还有 CyclicBarrier, Semaphore, ConcurrentHashMap and BlockingQueue 并发工具类，都属于 java.util.concurrent 包中。该类能使一个 Java 线程等待其它线程执行完成。例如应用程序的主线程需要等待其它服务的线程启动从而完成所有服务的启动。


CountDownLatch 的工作是通过线程数来初始化的一个计数器，每当一个线程执行完成计数器则为递减一次。当计数值为零时，也就意味着所有线程执行完成，些时正在等待的线程则恢复执行。

![alt "CountDownLatch"](http://howtodoinjava.com/wp-content/uploads/CountdownLatch_example.png?3f94dc)

CountDownLatch 的伪代码可以象这样来写：

{% highlight java %}
// 主线程启动
// 根据其它 N 个线程数创建 CountDownLoatch
// 创建并启动其它 N 个线程
// 主线程等待锁存器
// N 个线程完成任务并返回
// 主线程恢复执行
{% endhighlight java %}


<h4>CountDownLatch 的工作机制？</h4>
CountDownLatch.java 类中定义了一个构造函数:

{% highlight java %}
//Constructs a CountDownLatch initialized with the given count.
public void CountDownLatch(int count) {...}
{% endhighlight java %}

这里的 count 参数实质上是必须等待锁存器的线程数。该值只能设置一次，并且 CountDownLatch 没有提供其它来修改该 count 的机制。

在 CountDownLatch 首次与主线程交互要等待其它线程。这里的主线程必须在调用 CountDownLatch.await() 后立即启动其它线程。该主线程将停止在 await() 方法执行的时间，到其它线程完成它们的执行。

其它 N 个线程必须拥有锁存器的对象引用，因为它必须要通知 CountDownLatch 对象它已经完成任务。这里的通知完成的方法是：CountDownLatch.countDown()；每个方法在调用的时候会将构造函数中初始的计算器变量的值减1，所以当所有线程都调用了这个方法，计数器达到零之后，主线程在就恢复执行之前的 await() 方法。

<h4>在实际应用程序可能用途</h4>

让我们试图找出一些可能在实际 Java 应用程序中的用途。这里我尽可能的列举出我所记得的用法，如果您有任何其它可能的用法，请发表在评论中，这将会帮助其它人。

**1. 实现最大并行：** 有时候我们需要在同一时间启动多个线程以达到最大的并行。例如，我们要测试一个类是否是单例。这将很容易做到，如果我们创建 CountDownLatch 初始化计算器为1，并且等待所有线程在等待锁存器。单独调用 countDown() 方法将所有正在等待的线程在同一时间恢复执行。

**2. 等待多个线程完成之后再开始执行：** 例如应用程序启动类要确保所有 N 个外部系统已经运行之后才能处理用户请求。

**3. 检测死锁：** 一个非常方便的使用情况，在每个测试您可以在其中使用 N 个线程访问共享资源的不同数量的线程，并且尝试创建一个死锁。


<h4>CountDownLatch 示例程序</h4>
在这个例子中，模拟启动 N 个用于检查外部服务的线程，其主要启动类将等待外部服务并将结果返回给锁存器。一旦所有服务进行了验证和检查，启动将继续执行。

BaseHealthChecker.java : 该类实现了 Runnable 接口作为所有外部健康检查服务的父类。 在这里它避免的代码的重复和对锁的控制。

{% highlight java %}
public abstract class BaseHealthChecker implements Runnable {
     
    private CountDownLatch _latch;
    private String _serviceName;
    private boolean _serviceUp;
     
    //Get latch object in constructor so that after completing the task, thread can countDown() the latch
    public BaseHealthChecker(String serviceName, CountDownLatch latch)
    {
        super();
        this._latch = latch;
        this._serviceName = serviceName;
        this._serviceUp = false;
    }
 
    @Override
    public void run() {
        try {
            verifyService();
            _serviceUp = true;
        } catch (Throwable t) {
            t.printStackTrace(System.err);
            _serviceUp = false;
        } finally {
            if(_latch != null) {
                _latch.countDown();
            }
        }
    }
 
    public String getServiceName() {
        return _serviceName;
    }
 
    public boolean isServiceUp() {
        return _serviceUp;
    }
    //This methos needs to be implemented by all specific service checker
    public abstract void verifyService();
}
{% endhighlight java %}
NetworkHealthChecker.java : 这个类继承了 BaseHealthChecker 并且只需要提供实现 verifyService() 方法。 DatabaseHealthChecker.java 和 CacheHealthChecker.java 同样如此，只不过他们的服务名和休眠的时间不一样。


{% highlight java %}
public class NetworkHealthChecker extends BaseHealthChecker
{
    public NetworkHealthChecker (CountDownLatch latch)  {
        super("Network Service", latch);
    }
     
    @Override
    public void verifyService() 
    {
        System.out.println("Checking " + this.getServiceName());
        try
        {
            Thread.sleep(7000);
        } 
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        System.out.println(this.getServiceName() + " is UP");
    }
}
{% endhighlight java %}

ApplicationStartupUtil.java :这个是应用程序的启动类，用来初始化锁存器并且等待所有服务已经检查完成。


{% highlight java %}
public class ApplicationStartupUtil 
{
    //List of service checkers
    private static List<BaseHealthChecker> _services;
     
    //This latch will be used to wait on
    private static CountDownLatch _latch;
     
    private ApplicationStartupUtil()
    {
    }
     
    private final static ApplicationStartupUtil INSTANCE = new ApplicationStartupUtil();
     
    public static ApplicationStartupUtil getInstance()
    {
        return INSTANCE;
    }
     
    public static boolean checkExternalServices() throws Exception
    {
        //Initialize the latch with number of service checkers
        _latch = new CountDownLatch(3);
         
        //All add checker in lists
        _services = new ArrayList<BaseHealthChecker>();
        _services.add(new NetworkHealthChecker(_latch));
        _services.add(new CacheHealthChecker(_latch));
        _services.add(new DatabaseHealthChecker(_latch));
         
        //Start service checkers using executor framework
        Executor executor = Executors.newFixedThreadPool(_services.size());
         
        for(final BaseHealthChecker v : _services) 
        {
            executor.execute(v);
        }
         
        //Now wait till all services are checked
        _latch.await();
         
        //Services are file and now proceed startup
        for(final BaseHealthChecker v : _services) 
        {
            if( ! v.isServiceUp())
            {
                return false;
            }
        }
        return true;
    }
}
{% endhighlight java %}

接下来你可以些一些测试类来检查有关锁存器的功能：

{% highlight java %}
public class Main {
    public static void main(String[] args) 
    {
        boolean result = false;
        try {
            result = ApplicationStartupUtil.checkExternalServices();
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("External services validation completed !! Result was :: "+ result);
    }
}
 
Output in console:
 
Checking Network Service
Checking Cache Service
Checking Database Service
Database Service is UP
Cache Service is UP
Network Service is UP
External services validation completed !! Result was :: true
{% endhighlight java %}

<h4>常见的面试问题</h4>

准备在你下次面试时可能会面临有关 CountDownLatch 的一些问题：

* 解释 CountDownLatch 的一些基本概念?
* CountDownLatch 和 CyclicBarrier 之前有哪些差异?
* 给出一些使用 CountDownLatch 的例子?
* CountDownLatch 类中最主要的方法有哪些？


你可以下载文章中示例的完整代码：

[Sourcecode Download](https://docs.google.com/file/d/0B7yo2HclmjI4Rl9EUUl2cmI0X28/edit?usp=sharing)



**Original: [When to use CountDownLatch : Java concurrency example tutorial](http://howtodoinjava.com/2013/07/18/when-to-use-countdownlatch-java-concurrency-example-tutorial/)**
