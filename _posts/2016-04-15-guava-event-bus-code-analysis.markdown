---
layout: post
title: Guava EventBus 发布 / 订阅源码分析
---

如果想了解[观察者模式](https://zh.wikipedia.org/zh-cn/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F)的实现或应用场景，[Google Guava](https://github.com/google/guava) [EventBus](https://github.com/google/guava/wiki/EventBusExplained)  所提供的[发布/订阅](https://zh.wikipedia.org/zh-cn/%E5%8F%91%E5%B8%83/%E8%AE%A2%E9%98%85)功能非常简洁优雅的实现这一模式，且对于使用者不需要依赖复杂的接口或实现。其整体设计非常值得学习和参考，本文将从其源码分析其实现细节。


#### 示例代码

以下使用 EventBus 的简要示例代码，针对不同的 Mouse Event 类型，会触发相应 Event handler 方法：

{% highlight java %}
// 定义事件实体
public class ClickEvent {}
public class MouseMoveEvent {}

// 定义事件监听者
public class MouseListener {
    @Subscribe
    public void handleClickEvent(ClickEvent event) {
        println("click event trigger");
    }
    @Subscribe
    public void handleMouseMoveEvent(MouseMoveEvent event) {
        println("mouse move event trigger");
    }
}

EventBus bus = new EventBus();

// 注册订阅者
bus.register(new MouseListener());
// 发送事件给相应事件订阅者
bus.post(new ClickEvent());
bus.post(new MouseMoveEvent());

// Output:
click event trigger
mouse move event trigger
{% endhighlight java %}

更加详细的使用说明可以查看 [EventBusExplained](https://github.com/google/guava/wiki/EventBusExplained)。

#### 实现细节

从代码示例来看，用到 EventBus 的主要行为为 `register` 和 `post` ，分别是注册订阅者和事件发送：

##### EventBus.register

调用  `EventBus.register` 将事件类型与所有订阅该事件的订阅者进行关联，该方法通过调用 `SubscriberRegistry.register` 来执行注册。
当注册完成后，并由全局 `subscribers`  存储关联关系：

{% highlight java %}
/**
 * All registered subscribers, indexed by event type.
 */
private final ConcurrentMap<Class<?>, CopyOnWriteArraySet<Subscriber>> subscribers = Maps.newConcurrentMap();
{% endhighlight java %}

![img](/images/2016/47cb0c629160a6edd40abd4d423cfe84.png)

**那么 `SubscriberRegistry.register` 是如何实现注册的呢？**

**STEP1：**
如下代码，首先通过 `findAllSubscribers` 查找 listener 实例标识为 `@Subscribe` 的方法，并通过 Map 将方法参数类型(即事件类型)与对应的方法实例(由 `Subscriber` 包装 Method Object)进行关联。

{% highlight java %}
void register(Object listener) {
  // 这里的 Multimap 实则为 Map<Class<?>, Collection<Subscriber>> 结构
  Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);
{% endhighlight java %}

**STEP2：**
接着再将该 listener 所有的事件类型与对应的 `Subscriber` 关系增量到全局的 `subscribers` 变量中。以下 `for` 相当于将 `listenerMethods` 及全局的 `subscribers` 进行 Map 合并操作：

{% highlight java %}
  for (Map.Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
    Class<?> eventType = entry.getKey();
    Collection<Subscriber> eventMethodsInListener = entry.getValue();
    CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

    if (eventSubscribers == null) {
      CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<Subscriber>();
      eventSubscribers =
          MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
    }

    eventSubscribers.addAll(eventMethodsInListener);
  }
}
{% endhighlight java %}

至此已经完成对监听器的注册，当具体事件类型触发时，可以根据事件类型从全局的 `subscribers` 中获取该事件类型所有订阅者。
另外，以上注册过程中使用了 `ConcurrentMap` 和  `CopyOnWriteArraySet` 进行数据操作，实则为了避免在多线程并发执行 `register` 时导致的[线程安全](https://zh.wikipedia.org/zh-cn/%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8)问题，通过以下链接可以了解具体信息：

* [深入了解 ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap)
* [Java 中的 CopyOnWrite 容器](http://coolshell.cn/articles/11175.html)


#####  EventBus.post

当执行相应事件 `post(event)` 时，先根据事件类型获取到所有 `Subscriber`，再通过 `Dispatcher` 将事件通知到对应 `Subscriber`，最终由 `Subscriber` 实例执行其 `Method` 的 `invoke` 方法，即完成对具体 Listener 中方法的调用。
![EventBus.post UML](/images/2016/259b0e2bd4c279f7ae554b080b25f89a.png)


EventBus 提供了三种 `Dispatcher` 事件分发机制：

**PerThreadQueuedDispatcher**

如不指定默认该 dispatcher，通过队列及 `ThreadLocal` 来保证在同一个线程上发送的事件能够按发布的顺序分发给所有订阅者。
首先将需要分发的事件和对应的订阅者列表添加至当前线程的队列中：

{% highlight java %}
void dispatch(Object event, Iterator<Subscriber> subscribers) {
  checkNotNull(event);
  checkNotNull(subscribers);
  Queue<Event> queueForThread = queue.get();
  queueForThread.offer(new Event(event, subscribers));
{% endhighlight java %}


这里使用了 `ThreadLocal<Boolean> dispatching` 状态来标识当前线程是否分发完成，以避免重入分发。([什么是可重入？](http://blog.csdn.net/tennysonsky/article/details/45127125))

{% highlight java %}
  if (!dispatching.get()) {
    dispatching.set(true);
{% endhighlight java %}

标识为"分发中"状态后，再通过两层循环，先后遍历事件队列和订阅者，将相应事件送达到订阅者中。

{% highlight java %}
    try {
      Event nextEvent;
      while ((nextEvent = queueForThread.poll()) != null) {
        while (nextEvent.subscribers.hasNext()) {
          nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
        }
      }
    } finally {
      dispatching.remove();
      queue.remove();
    }
  }
}
{% endhighlight java %}

**LegacyAsyncDispatcher**

在使用 `AsyncEventBus` ，则默认使用该分发器。与 `PerThreadQueuedDispatcher` 区别在于通过维护全局事件分发队列，使事件发布及订阅者能在多线程下并行执行。但无法完全保证在多线程下按事件发布顺序进行分发。

**ImmediateDispatcher**

遍历 `subscribers` 直接进行事件分发。

最后，由对应 `Subscriber` 接收到相应事件，并完成对订阅方法的调用：

{% highlight java %}
final void dispatchEvent(final Object event) {
  // 如果是 AsyncEventBus 则可以支持通过指定 executor 来异步执行 invoke
  executor.execute(
      new Runnable() {
        @Override
        public void run() {
          try {
            invokeSubscriberMethod(event);
          } catch (InvocationTargetException e) {
            bus.handleSubscriberException(e.getCause(), context(event));
          }
        }
      });
}
{% endhighlight java %}
