> 主要
>
> + 补坑：分布式链路跟踪设计与实现
> + Java 方法重载、覆盖、桥接以及参数匹配的秘密
> + 面试题资源分享
>





## 分布式链路跟踪设计与实现
### Apache Skywalking


一次完整链路请求具备唯一的 Id，称为 Trace ID，在分布式系统中，不同的服务部署在不同的物理节点上个，Trace ID 需要在所有的调用节点上传递。不过每个节点有自身跟踪信息，每个节点可以简称为 Span（间隔、阶段、片段），一般而言，一个 Trace ID 对应 N 个 Span Id，Span 粒度取决于实现。

比如：一个 Web 服务，它内部 Service 可能会调用 RPC 服务（外部），本地的数据库事务（本地）

本地事务可能是通过 JPA、MyBatis -> JDBC 实现 -> 数据库连接池

跟踪记录类型：

+ 时间
    - 开始时间（毫秒）、结束时间（毫秒），统计消耗时间（纳秒）
+ 系统信息
    - CPU
    - Load
    - 内存
    - 网络等
+ 资源
    - Endpoint
    - Resource



Tracing

Logging

Metrics

Fault-Tolerance

Profiling



AOP 非常重要技术



Agent-Based：Java Agent，更好地实现可插拔，性能通常使用字节码提升来静态的方式拦截。实现属于黑盒，不容易被调试，同时 API 兼容性不好测试（接入方），高阶扩展存在局限性。

API-Based：性能取决于动态代理或静态代理，学习成本相对比较好，对系统可能存在一定的稳定性影响。





生态拦截点扩展点

+ HTTP Server
    - Servlet Engine
    - Spring Reactive Server
    - Vert.x
    - Playframework 
    - ...
+ HTTP Client
    - HTTP Client 3.x
    - HTTP Component 4.x
    - Netty HTTP
    - ...
+ JDBC
    - JDBC Drivers
    - JPA
    - MyBatis
+ RPC
+ MQ
+ NoSQL
+ Service Discovery



如何让 Trace ID 透传

+ Servlet Engine
    - Filter API
    - ServletRequestListener API
+ JDBC
    - Druid API
    - P6Spy API
+ RPC
    - Apache Dubbo	
        * Filter
    - Feign
        * Capacity





## Java 方法重载、覆盖、桥接以及参数匹配的秘密
重载：Overload

覆盖：Override

桥接：Brige



问：写一个 Java 程序来判断，判断两个方法之间是否存在重载或覆盖关系？

重载：Overload，方法名相同，方法参数不同，方法返回值不关注，被调用采用参数匹配机制。对于类的层次关系没有直接联系。

覆盖：Override，方法名相同，方法不同，非静态方法，非私有方法，非默认方法。

覆盖方法与被覆盖方法不属于同一个类或接口，覆盖方法所声明的类或接口必须是被覆盖方法所声明的类或接口的子类或子接口。

方法参数：

+ 参数数量相同
+ 参数类型需要匹配

方法返回类型：被覆盖的方法要返回类型是覆盖方法的返回类型的相同类或子类。

参数匹配：匹配的原则，精确匹配（具体到某种类型） -> 类型匹配（可以接受子类）-> 层次（层次路径短优先）匹配





Method

Method



class C extends B {

}



class B  {

public void m(){

}

}





