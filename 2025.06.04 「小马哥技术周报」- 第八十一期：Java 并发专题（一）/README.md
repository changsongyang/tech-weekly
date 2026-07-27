> 主要议题：
>
> 1. 阿里内部真的不使用 Executors 工具类创建线程池吗？
> 2. 是线程池超隐蔽的坑？还是代码实现存在瑕疵？
> 3. 如何判断线程池中的线程以及任务执行结束？
> 4. 如何解决 ThreadLocal 在 J.U.C 线程池场场景下父子线程对象传递问题？
> 5. 如何避免 CountDownLatch 滥用的问题？
>



## 阿里内部真的不使用 Executors 工具类创建线程池吗？
[https://www.bilibili.com/video/BV1ZK78zrEmk](https://www.bilibili.com/video/BV1ZK78zrEmk)



### Executors#newFixedThreadPool 方法
创建一个固定大小的线程池，使用过程中需要注意任务数量不可长时间高于线程总数量。

如果不使用该方法的话，那么，需要特别注意 BlockingQueue 与 <font style="color:#080808;background-color:#ffffff;">RejectedExecutionHandler 之间的关系。</font>

### <font style="color:#080808;background-color:#ffffff;">Executors#newCachedThreadPool 方法</font>
需要注意线程任务不可过多，导致 JVM 创建过多的 OS Threads。



如果 RPC 框架处理任务，交给 <font style="color:#080808;background-color:#ffffff;">newCachedThreadPool 来处理的话，容易导致 OOM。</font>

<font style="color:#080808;background-color:#ffffff;">RPC 如果 Dubbo 的话，默认大概几百线程（n），总体线程最大值 2n，使用不当。</font>

<font style="color:#080808;background-color:#ffffff;">一个 Boss 线程数量 < Worker 线程数量 >= 实际线程数量</font>

<font style="color:#080808;background-color:#ffffff;"></font>

### <font style="color:#080808;background-color:#ffffff;">Executors#newSingleThreadExecutor 方法</font>
适合于单任务执行，或者少量并发任务。



### Executors#<font style="color:#080808;background-color:#ffffff;">newScheduledThreadPool 方法</font>
适合于任务调度场景，不适合高并发处理。



## 是线程池超隐蔽的坑？还是代码实现存在瑕疵？
[https://www.bilibili.com/video/BV12K7LzLECm](https://www.bilibili.com/video/BV12K7LzLECm)



问题分析：

1. 代码瑕疵

将 Boss 线程与 Worker 线程公用一个线程池，这样在高并发场景下，很容易发生相互等待的等待（死锁）。

Boss 线程：多个用户迁移线程（BT）

Worker 线程：单个用户迁移线程（WT），交给异步线程处理用户同步于账户同步。

线程池配置：core 2，max 10，queue 200，同步调用拒绝策略

假设用户数量大于 size (core + queue) =  2 +200 = 202

如果用户数量大于 size (max + queue) = 10 + 200 = 210

第 211 个用户开始（假设前面的用户尚未处理完成），执行线程为调用 moveUsers 方法线程，即 main 线程。



在 210 个用户内部，max 线程集合均在处理  Boss 线程，即moveUsers 方法线

第一个用户，core 2 = 1 moveUsers 方法 + 1 syncUser 方法，那么 syncAcount 被扔到 queue，因为 check 方法让 CountDownLatch#await()，这里就出现了死锁。

视频中解决办法：调整 CountDownLatch#await(long) 方法，传递一个超时时间：5000ms（5秒）

比较合理方法：

1. 将 Boss 线程与 Worker 线程分离，不要使用同一个线程池
2. 减少使用 CountDownLatch 的方式
3. 同步方案：使用过 <font style="color:#080808;background-color:#ffffff;">CompletableFuture 来处理 syncUser 与 syncAcount 方法，这两个方法调用变成串行（如果需要强制让 User 和 Accout 完成，做一次迁移校验）</font>
4. <font style="color:#080808;background-color:#ffffff;">异步方案：假设syncUser 与 syncAcount 方法 必须是异步的话，可以考虑使用 CompletionService</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">CountDownLatch</font>

<font style="color:#080808;background-color:#ffffff;">Future</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">都是 Condition 变种</font>

<font style="color:#080808;background-color:#ffffff;"></font>

## <font style="color:#080808;background-color:#ffffff;">如何判断线程池中的线程以及任务执行结束？</font>
### ExecutorService#isTerminated 方法(不适合）
isTerminated 是判断线程池所有的任务是否执行结束（必须调用 ExecutorService#shutdown() 方法或者 #shutdownNow() 方法）



### Future#get 方法
强制等待线程任务执行完成，Runnable#run() 方法被调用。



Future#isDone() 方法也可以判断任务是否执行结束

Future#get() 方法：

1. 正常：方法执行结束
2. <font style="color:#080808;background-color:#ffffff;">ExecutionException：内部执行失败</font>
3. <font style="color:#080808;background-color:#ffffff;">InterruptedException：线程被中止</font>
4. <font style="color:#080808;background-color:#ffffff;">TimeoutException：执行时间大于最大等待时间</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">建议 isDone 与 get 方法配合：</font>

```java
Future future = ...

while(!future.isDone()) { // 这里如果发生异常：ExecutionException、InterruptedException 是无法感知。
}

future.get(); // 此处处理异常分支
```

<font style="color:#080808;background-color:#ffffff;"></font>

### <font style="color:#080808;background-color:#ffffff;">CountDownLatch 方法（不推荐）</font>
不如使用 Future 来判断，底层逻辑类似，不侵入代码。



## 如何解决 ThreadLocal 在 J.U.C 线程池场景下父子线程对象传递问题？
ThreadLocal 是线程的本地变量存储结构

<font style="color:#080808;background-color:#ffffff;">InheritableThreadLocal 是父子线程的本地变量存储结构</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">InheritableThreadLocal 为什么无法解决J.U.C 线程池场场景下父子线程对象传递问题？</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">父子线程关系：</font>

<font style="color:#080808;background-color:#ffffff;">在 Java 应用中，所有的 Java 线程均有 main 线程创建（JVM 创建）。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

```java
public class Demo {

    public static void main(String[] args){
        Thread t = new Thread(); // main 线程创建 Java Thread 对象
    }
}
```



<font style="color:#080808;background-color:#ffffff;">InheritableThreadLocal 之所有能够使用父线程（即创建该 Java Thread 对象的线程） ThreadLocal，是因为在 Thread 构造期间：</font>

```java
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        ...
        Thread parent = currentThread();
        ...
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        ...
    }
```

众所周知，J.U.C 线程池创建内部线程的线程是不确定的。

假设 Tomcat 容器（线程池 core  = 10 ， max = 200）创建 线程池 TP-1 中的线程，TP-1（core = 2 ， max = 10，queue = <font style="color:#080808;background-color:#ffffff;">SynchronousQueue</font>）

如果并发 HTTP 请求发送到 Tomcat 容器 上，将 HTTP 请求提交给程池 TP-1 ，开始创建 TP-1 中的线程。

假设 HTTP-T-1 创建 TP-1-T-1，依次类推，HTTP-T-10 -> TP-1-T-10。

那么， HTTP-T-11 不再为 线程池 TP-1 创建线程，而是复用其线程。由于 HTTP-T-11 与其前 10 个线程是并行，无状态的关系，因此，HTTP-T-11之后的 Tomcat 线程与 TP-1 线程不再存在父子线程关系。所以，TP-1 中的线程即使使用 <font style="color:#080808;background-color:#ffffff;">InheritableThreadLocal，也不一定能够正确地获取到 Tomcat 线程中的 ThreadLocal 引用。</font>

如何解决以上问题？本质上是需要解决执行线程池提交任务线程与线程池任务线程的 ThreadLocal 引用。

假设 TP-1 线程池存在一个等待任务的线程（TP-1-T-1 线程），接下来执行 HTTP-T-11 的任务，它可能会使用 TP-1-T-1 线程。



Worker 中的 task 来自于 HTTP-T-11 线程的任务，其 <font style="color:#080808;background-color:#ffffff;">runWorker(Worker) 方法会被回调：</font>

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread(); // TP-1-T-1 线程
        ...
        beforeExecute(wt, task);
        ...
                    Throwable thrown = null;
                    try {
                        task.run(); // HTTP-T-11 线程的任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
        ...
    }
```

 

解决方案之一：

```java

    class MyThreadPoolExecutor extends ThreadPoolExecutor {

        static final String THREAD_LOCAL_MAP_CLASS_NAME= "java.lang.ThreadLocal$ThreadLocalMap";

        public MyThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                                    TimeUnit unit, BlockingQueue<Runnable> workQueue,
                                    ThreadFactory threadFactory,
                                    RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        }

        @Override
        public void execute(Runnable command) {
            Thread bossThread = currentThread();
            super.execute(new MyRunnable(bossThread, command));
        }

        // 有没有可能在 beforeExecute执行的时候 boss 线程已经结束了，或者被其他请求复用了呢？
        // 答：boss 线程执行结束不会存在问题，因为 boss 线程 Java 对象是不会马上 GC 的。

        @Override
        protected void beforeExecute(Thread workerThread, Runnable taskWrapper) {
            if (taskWrapper instanceof MyRunnable) {
                MyRunnable myRunnable = (MyRunnable) taskWrapper;
                Thread bossThread = myRunnable.bossThread;
                // bossThread ThreadLocal 合并到  workerThread ThreadLocal
                // 通过反射方法来执行
                // 假设 workerThread ThreadLocal 已经存在了，无法通过覆盖的方式来获取 bossThread ThreadLocal

                // 1. 获取 bossThread ThreadLocalMap
                // 2. 获取 workerThread ThreadLocalMap
                // 3. 迭代 bossThread ThreadLocalMap(threadLocals 字段) 中的 Entry[] table 合并到 workerThread ThreadLocalMap
                // （inheritableThreadLocals 字段）
                // 4. ThreadLocalMap 中的 Entry 是一个 WeakReference<ThreadLocal<?>>，workerThread ThreadLocalMap 需要的是
                // Entry 中的 Value，而不是 ThreadLocal 这个 Key。将该 Value 放入 workerThread ThreadLocalMap 中
                // 5. workerThread ThreadLocalMap 改造成首先通过 bossThread ThreadLocalMap，再获取自身 ThreadLocalMap
            }
        }

        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            // 还原 workerThread ThreadLocal
        }
    }

    class MyRunnable implements Runnable {

        private final Thread bossThread;

        private final Runnable delegate;

        MyRunnable(Thread bossThread, Runnable delegate) {
            this.bossThread = bossThread;
            this.delegate = delegate;
        }

        @Override
        public void run() {
            Thread workerThread = currentThread();
            delegate.run();
        }
    }
```



## 如何避免 CountDownLatch 滥用的问题？
CountDownLatch 推荐在精确知晓并发线程数量时使用，强制任务线程执行结束。

