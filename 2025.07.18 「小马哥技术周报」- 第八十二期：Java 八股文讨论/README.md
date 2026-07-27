> 主要议题：
>
> 1. 这个“坑”是否为 Integer 设计的缺陷，或者是 Java 并发的 Bug？如果不是，为什么这么设计？   
原视频：[有人说这个题太坑了，一起来review！ ](https://www.bilibili.com/video/BV1yDgwzpEFc/)
> 2. ThreadLocal 在线程池真的不存在内存泄漏吗？如果泄漏的话，实际情况是如何的？   
原视频：[ThreadLocal会产生内存泄漏，不存在的兄弟！](https://www.bilibili.com/video/BV11RgKzKEv4/)
> 3. 这种方式的转换真的高效么？  
原视频：[ 阿里二面：100行User的List怎么高效转为Map？直接被问懵了。。](阿里二面：100行User的List怎么高效转为Map？直接被问懵了。。)
> 4. Java 创建线程到底有多少种方法？  
原视频：[阿里一面：Java如何创建线程？创建线程的方式有几种？一通问下来被整懵了。。](https://www.bilibili.com/video/BV1oUKGzfEzL)
>



## 议题一：Integer 八股文
Java 中不支持操作符重载（C++），但是 Java 内部保留这个能力，但是不想开发者扩展，比如：

+ String "+" 
+ Number ++



Integer ++ 到底有哪些底层实现：

Integer -> int

> 字节码调用：Integer#initValue()
>



int ++

> iload
>
> iconst_1
>
> iadd
>



count ++ 

int -> Integer

> 字节码调用：Integer.valueOf(int)
>



字节码分析：

```java
            for (int i = 0; i < 10000; i++) {
                /**
                 14: monitorenter
                 15: getstatic     #19                 // Field count:Ljava/lang/Integer;
                 18: astore_2
                 19: getstatic     #19                 // Field count:Ljava/lang/Integer;
                 22: invokevirtual #25                 // Method java/lang/Integer.intValue:()I
                 25: iconst_1
                 26: iadd
                 27: invokestatic  #31                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
                 30: putstatic     #19                 // Field count:Ljava/lang/Integer;
                 33: aload_2
                 34: pop
                 35: aload_1
                 36: monitorexit
                 */
                synchronized (count) { // getstatic
                    // getstatic : 读取 count 静态字段, c0 对象（Integer）
                    // invokevirtual : 调用 count 这个 Integer 对象的 intValue() 方法，生成临时变量 i0（栈）
                    // iconst_1 : 加载一个 int 类型常量 1
                    // iadd : i0 + int 类型常量 1 = 2(int)（栈）
                    // invokestatic : 调用 Integer.valueOf(int) 方法，将 i0 int 类型转为 Integer 对象 c1(堆）
                    // putstatic ： 将 Integer 对象 c1(堆）存储到 count 字段（Class 对象中的成员，元空间 metaspace）
                    count++;
                }
            }
        });
```



### 议题二：ThreadLocal 八股文
ThreadLocal



Thread



ThreadLocalMap



### 强引用或弱引用关系分析
Entry 对象是特殊强引用，扩展 WeakReference。

ThreadLocal 对象是弱引用，又是 ThreadLocalMap Key

Entry 对象中的 value 是任意对象引用

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

Entry 对象存储，即 ThreadLocalMap 中的 <font style="color:#080808;background-color:#ffffff;">table 字段（Entry 对象数组）也是强引用，只要 ThreadLocalMap 不被GC，那么，table 数组以及成员也不会被回收。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">ThreadLocalMap 又是 Thread 对象的成员，所以当 Thread 对象被 GC 时，ThreadLocalMap 才会被 GC</font>

<font style="color:#080808;background-color:#ffffff;">-> table 数组以及成员 被 GC</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">Thread 对象如果在线程池中，由于非异常执行不会被销毁，ThreadLocalMap 也不会随之 GG。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">假设 ThreadLocal 对象（代码定义的）如果将被 GC（弱可达状态），那么，ThreadLocal 对象作为 ThreadLocalMap Key 将会为 null。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">ThreadLocalMap 中 table 数组是</font>

<font style="color:#080808;background-color:#ffffff;">通过 Entry Key 即 ThreadLocal 对象来计算 hash 值，Value 则是任意对象引用。</font>

<font style="color:#080808;background-color:#ffffff;">如果 Key 不存在（ThreadLocal 对象弱可达）的话，hash 映射关系无法建立，Value 就成了引用的“孤岛”。但是 Entry 对象不会被 GC。</font>

<font style="color:#080808;background-color:#ffffff;">只有该 Entry 被 GC 后，Value 也被 GC。</font>

<font style="color:#080808;background-color:#ffffff;"></font>

<font style="color:#080808;background-color:#ffffff;">当 ThreadLocalMap#getEntry(ThreadLocal) 方法调用时，</font>

```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```

<font style="color:#080808;background-color:#ffffff;">会移除“孤岛”，条件如下：</font>

```java
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                if (e.refersTo(key))
                    return e;
                if (e.refersTo(null)) // e 为 Entry 对象，也是 WeakReference 实例
                    expungeStaleEntry(i); // 移除脏数据
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```



### <font style="color:#080808;background-color:#ffffff;">代码分析</font>
```java
    public void action() { // 按照 up 说法，会发生内存泄露
        ThreadLocal<Integer> tl = new ThreadLocal<Integer>();
        tl.set(1);
    }
```

由于 ThreadLocal 是方法内部对象，当 action 方法执行结束，ThreadLocal 即将被 GC（对象非逃逸状态）。

ThreadLocal 对象引用可达性 强 -> 软 -> 弱 -> 虚拟 -> 最终





### OOM 泄露风险代码
```java
    private static final ThreadLocal<Map<String, Object>> contextThreadLocal = withInitial(HashMap::new);
    public void actionInOOM(long userId) {
        // 如果 contextThreadLocal 不调用 remove 方法时，在线程池环境中 context 将可能变膨胀。
        Map<String, Object> context = contextThreadLocal.get();
        context.put("id-" + userId, "a");
    }
```

### 议题三：HashMap 八股文
如何定义高效？

+ 空间
+ 时间



100 对象 List 存放到 HashMap



HashMap size = 100， factor = 1.00f

100 -> 128



## 议题四：Java Threading 八股文
创建 Java 线程有且仅有一种方法，即 Thread#start() 方法，当 OS 支持线程的话。

运行 Java 线程任务的方法， 有 N 种：

+ Runnable
+ Thread#run() 方法覆盖
+ Callable
+ Supplier
+ Function
+ Consumer
+ 其他







