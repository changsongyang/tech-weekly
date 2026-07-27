> 主要议题：
>
> 1. [Spring声明式事务多线程失效？2句代码即可解决！](https://www.bilibili.com/video/BV1QJL7z4EUh)
> 2. <font style="color:rgb(24, 25, 28);">Code Review：Redis 分布式锁在数据一致性可能存在的问题</font>
>
> <!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/222258/1747227550391-b29cce8b-4d63-410b-a9d0-90b3686f7f01.png)
>
> 3. Log4j2 异步线程不用是为什么？
>
> <!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/222258/1747227577768-a2658943-6dc6-4f3a-9ece-2f4933e3d783.png)
>



## 议题一：数据库事务
数据库层面不关心数据库命令客户端的实现，它只关心接受数据库命令的网络参数或者数据

以 MySQL 为例，MySQL Client 可以是 Java（JDBC）、C#、GoLang 等，甚至普通的 MySQL Workbench，网络架构来说是典型的 C/S。在同一次网络连接或者数据库 Session 中，执行命令实际上是串行。



按照视频中的案例，其实是一个串行的执行。

一般而言，当代码中遇到了多线程执行数据库事务，尽量避免这种代码。

如果采用视频中的方法的话，若需保证数据一致性，必然会带来内存和资源更多消耗，并且与串行调用没有区别。



在 Java 中，几乎所有的关系型数据库均提供了 JDBC 驱动适配/实现，只不过不一定完全实现所有的规范细节。

几乎所有的 Statement 执行 SQL 均发送网络命令。

```java
public UserService {


    @Transactional
    @Async
    public void b() {
    }
}
```



假设 @Async 未规定其线程池 Bean 的话，在 Spring 应用上下文默认情况下，@Async 对应的 TaskExecutor 是一个非线程池实现 - `SimpleAsyncTaskExecutor`，每次执行 execute 方法会创建独立的线程。所以，这种情况，是无法插入 Spring 数据库事务资源绑定代码。

假设，Spring 应用上下文中存在一个 TaskExecutor Bean 定义，那么 @Async  注解将使用该 TaskExecutor 实现。再者，如果 TaskExecutor Bean 属于 ThreadPoolTaskExecutor 的话，还需要通过自定义实现 `TaskDecorator`接口，来包装 Runnable 实现：

```java
class TaskDecoratorImpl implements TaskDecorator {

        @Override
        public Runnable decorate(Runnable runnable) {
            return () -> {
                beforeRun();
                runnable.run();
                afterRun();

            };
        }

        protected void beforeRun() {
            // 如何实现 ConnnectionHolder 获取？
        }


        protected void afterRun() {

        }
    }
```



在每次事务调用之前，拿到 ThreadPoolTaskExecutor，并且动态地植入 TaskDecorator 实现，并且还需要注意线程安全，<font style="color:#080808;background-color:#ffffff;">taskDecorator 字段未经过 volatile 的修饰，属于线程不安全的赋值：</font>

```java
public class ThreadPoolTaskExecutor extends ExecutorConfigurationSupport
		implements AsyncListenableTaskExecutor, SchedulingTaskExecutor {
    ...
    @Nullable
	private TaskDecorator taskDecorator;
    ...
}
```

假设存在 <font style="color:#080808;background-color:#ffffff;">volatile 的字段修饰，也无法保证状态一致性，因为 setTaskDecorator 与 事务处理线程是多阶段。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/222258/1747230011801-6326116f-9b53-46f2-a662-206c4c6fba0d.png)

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:rgb(24, 25, 28);">假设方法内部只有一个子线程参与事务，它的父线程在调用它的join 方法时，必须等他执行结束，这样就是串行。具体而言，</font>

<font style="color:rgb(24, 25, 28);">第一种情况，如果父线程在 join 方法之前插入了一些数据，子线程也修改一些数据，那么就是串行的。</font>

<font style="color:rgb(24, 25, 28);"></font>

<font style="color:rgb(24, 25, 28);">第二种情况，如果父线程在子线程 join 方法之后操作数据，那么还是串行的。</font>

<font style="color:rgb(24, 25, 28);"></font>

<font style="color:rgb(24, 25, 28);">第三种情况，比较复杂，如果父线程在子线程 start 和 join 方法之间操作数据，虽然子线程可以部分并行，然而数据插入变得无序，何来保障数据的一致性。再者，多线程共享同一个数据库连接，它们复用了一个 数据库 Session，其数据操作顺序依赖于命令到达的先后，会变成了无序的串行命令执行，所以想知道意义何在。</font>

<font style="color:rgb(24, 25, 28);"></font>

> <font style="color:rgb(24, 25, 28);">还是那句话  AP和CP只能保证其一， 你也是个讲师， 这个都不懂？   这视频讲的是</font>[<font style="color:rgb(0, 138, 197);">多线程事务</font>](https://search.bilibili.com/all?from_source=webcommentline_search&keyword=%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%BA%8B%E5%8A%A1&seid=7089411082740255972)<font style="color:rgb(24, 25, 28);">的一致性，  你在这跟我讲性能，有意义吗？    多线程事务如果要一致性一定会牺牲性能， 要么就用最终一致性， 你在这杠意义何在？</font>
>

<font style="color:rgb(24, 25, 28);"></font>

<font style="color:rgb(24, 25, 28);">AP 和 CP 问题存在于分布式环境。</font>



<font style="color:rgb(24, 25, 28);">本地事务尽量避免多线程环境下执行。</font>

<font style="color:rgb(24, 25, 28);"></font>

<font style="color:rgb(24, 25, 28);">S</font>pring 事务资源同享不采用 `InheritableThreadLocal`，而要选择 `ThreadLocal`来管理事务资源？

在 Sprng 事务管理中，不一定是简单地创建子线程的情况，有可能使用 ThreadPoolExecutor 这类线程池。



ThreadPoolExecutor 虽然内部的线程也是被父线程创建，可是它们的父线程比较复杂。

首先，ThreadPoolExecutor 存在核心线程和最大线程，其中核心线程有可能被提前启动。

这样，这些核心线程的父线程则是调用 `prestartAllCoreThreads` 或 `prestartCoreThread`方法的线程。

假设，核心线程不够时，剩余的子线程有可能是其他线程创建的。所以，线程池中的线程不好确定父线程的具体情况。



![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/222258/1747231061268-78f3ba4b-0f39-44bc-8f1b-971ecfc49ede.jpeg)



## <font style="color:rgb(24, 25, 28);">议题二：Code Review - Redis 分布式锁在数据一致性可能存在的问题</font>
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/222258/1747227550391-b29cce8b-4d63-410b-a9d0-90b3686f7f01.png)



tryLock 实现中缺少重进入的逻辑判断。



![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/222258/1747231668236-f50800f4-8243-49a5-865a-9aefc6c7c2fb.jpeg)



## 议题三：Log4j2 异步线程不用是为什么？
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2025/png/222258/1747227577768-a2658943-6dc6-4f3a-9ece-2f4933e3d783.png)



典型的线程池问题。开启线程操作是相对耗时，并且什么启动是一个 CPU 调度问题。在 main 线程执行应用启动时，通常会预启动相关的线程，介绍执行线程时的消耗。







Java AI

Java AI 主流框架在做 Client，Server 层才是关键。



AI

+ DevOps
+ Code Generation
+ Code Review



