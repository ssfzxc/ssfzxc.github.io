# TraceID

# 日志跟踪

在生产报错查询日志恢复当时场景时, 或者在测试环境进行查询 BUG 时,
可能出现其他服务或业务日志的干扰,
这时我们系统有工具或者其他什么可用的帮助我们是筛选出我们所需要的日志.

基于通常情况, 我们会使用在定位到 Exception 处后, 使用 Grep 筛选 Thread
Name, 这是一个可行有效的方案, 但是当遇到 Async 异步调用以及发送消息队列,
接下来日志的查询会是一个繁琐复杂的流程.

**如何高效地, 更少地侵入业务代码的情况下, 新增一个 ID 号贯穿整个业务流程
(biz chain)?**

# 日志新增 TraceId

接下来, 我们局部分析整个问题. 先从单应用服务开始, 引入 traceId
即一个贯穿头尾的 id, 将其在日志中打印, 实现一个 ID 贯穿业务

## 什么是 TraceId？

在业内, 日志跟踪有个高大上的名字叫做: 全链路跟踪, 其包括解决方案有
traceId 日志贯穿, 日志归集, 日志检索(ELK 全家通).

简单来说现在日志跟踪就是 **使用一个 id 流水号去贯穿整个业务调用**,

而 trace 用 **跟踪;追溯** 的意思, 所以我们用 traceId 代指
整个贯穿业务的流水号以便于区分其他如业务流水号 bizNo 等.

traceId 可以使用 UUID, MongoDB ID, SnowflakeId, Random 等方式生成,
只需要保证 id 号的唯一性即可.

## 简单实现

我们单纯的自己使用代码去实现, 首先上文也提到了使用 threadName,
因此可以使用 ThreadLocal 对象存放 traceId, 再对所有使用 Logger
打印日志的行新增打印 traceId, 这是大家以及大佬们多使用的方法.

但是对每一行 Logger 打印日志都修改工程量太过庞大,
以及不符合我们低侵入性的想法.

**如何低侵入性地修改?**

## 什么是 MDC?

首先看看 SLF4J 的官方文档是怎么说的:

> 此类隐藏并充当底层日志记录系统的 MDC 实现的替代品。
>
> 如果底层日志系统提供 MDC 功能，则 SLF4J 的
> MDC（即此类）将委托给底层系统的 MDC。注意，目前只有两个日志系统，即
> log4j 和 logback，提供了 MDC 功能。 对于不支持 MDC 的
> java.util.logging，将使用 BasicMDCAdapter。对于其他系统，即 slf4j
> simple 和 slf4j nop，将使用 NOPMDCAdapter。 因此，作为 SLF4J
> 用户，您可以在存在 log4j、logback 或 java.util.logging 的情况下利用
> MDC，但不必强制将这些系统作为依赖项依赖于您的用户。
>
> 有关 MDC 的更多信息，请参阅 logback 手册中有关 MDC 的章节。
>
> 请注意，此类中的所有方法都是静态的。

还是没有说明白, MDC 全称是 Mapped Diagnostic
Context，可以粗略的理解成是一个线程安全的存放诊断日志的容器。
就是一个存放当前线程上下文信息的对象. slf4j
文档说的是我们无需关注底层日志框架如何实现,slf4j 会委托给底层日志系统,

正如 slf4j 想实现的 **日志门面框架** 将代码和具体底层日志框架分开, 由
slf4j 做适配器桥接双方.

## 使用 MDC 实现

说到底 MDC 是对 ThreadLocal 的包装. 相对于我们自建 ThreadLocal 而言,
使用 MDC 我们可以直接在日志配置文件中配置输出 traceId

``` java
// 将 traceId 放入 MDC
MDC.put("traceId", newTraceId());
// 将 traceId 放入 MDC
String traceId = MDC.get("traceId");
// 清除MDC 中的 traceId
MDC.remove("traceId");
// 清空 MDC
MDC.clear();
// %X{treadId} 大跨号内的字符串保持和 MDC 的 key 相同即可
// logback 配置样例
// %date{yyyy-MM-dd HH:mm:ss.SSS} %green(%5level) --- [%15.15thread] %cyan(%-40.40logger{39}) : %magenta([%X{traceId}]) %message%n
// log4j 配置样例
// %d{yyyy-MM-dd HH:mm:ss.SSS} %t %c [%X{traceId}]  %m  %n
```

使用 MDC 和少量的修改下配置文件后, 对打印 traceId 就不需要一行一行的修改

traceid 工具类样例
```java

import org.slf4j.MDC;
import java.util.Objects;

/**
 * @author ssf on 2020-06-04
 */
public final class TraceUtil {
    private TraceUtil() {
    }

    public static final String TRACE_ID = "traceId";

    public static final String TRACE_FLAG_REQUEST = "REQUEST-";

    public static final String TRACE_FLAG_TIMER = "TIMER-";

    public static String newId(String flag) {
        Objects.requireNonNull(flag);
        String id = Ids.simpleUUID();
        return flag.trim() + id;
    }

    public static String newId() {
        return newId("");
    }

    public static String newRequestId() {
        return newId(TRACE_FLAG_REQUEST);
    }

    public static void setTraceId() {
        MDC.put(TRACE_ID, newId());
    }

    public static void setTraceId(String traceId) {
        MDC.put(TRACE_ID, traceId);
    }

    public static String getTraceId() {
        return MDC.get(TRACE_ID);
    }

    public static void removeTraceId() {
        MDC.remove(TRACE_ID);
    }
}
```

## Controller 新建 traceId

完成日志打印问题后新问题接踵而至, 在何处将 traceId 放入 MDC 对象中?

首先我们想到的是就是在每个 Controller 方法头和尾新增 put 和 remove
方法的调用. 但是还是不符合 *低侵入性*. 想想其他方式. 都是方法头和尾 -
before 和 after, 可以使用 **AOP** 面向切面的编程.

``` java
/**
 * aop 切面, Controller
 * @author ssf on 2020-06-15
 */
@Aspect
@Component
public class RequestAndResponseDumper { //implements HandlerInterceptor {

    @Pointcut("execution(* a.b.c.*.controller.*Controller*.*(..))")
    public void dump() {

    }


    @Before("dump()")
    public void trace(JoinPoint joinPoint) {
        TraceUtil.setTraceId(TraceUtil.newRequestId());
    }

    @AfterReturning(pointcut = "dump()", returning = "result")
    public void log(JoinPoint joinPoint, Object result) {
        TraceUtil.removeTraceId();
    }
}
```

还有方式: 因为是 Controller 即接口端, 也可使用 Web 容器的拦截器实现
`org.springframework.web.servlet.HandlerInterceptor`, preHandle 调用 put
方法, afterCompletion ( postHandle 也可, afterCompletion 更佳) 调用
remove 方法

# 异步 & 线程

解决了单线程业务, 需要面对多线程, 而多线程问题比较麻烦, 要想低侵入性,
那么只有几点建议

## 公共线程池管理

如果项目有公共的线程池管理线程, 则可以参考以下代码

``` java
/**
 * 线程池工具
 * @author ssf on 2020-06-15
 */
public final class ExecutorUtil {
    private ExecutorUtil() {
    }

    private static ExecutorService executor = Executors.newCachedThreadPool(new ThreadFactory() {

        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                    Thread.currentThread().getThreadGroup();
            namePrefix = "async-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                    namePrefix + threadNumber.getAndIncrement(),
                    0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    });


    public static void execute(final Runnable command) {
        final String traceId = TraceIdUtils.getTraceId();
        executor.execute(new Runnable() {
            @Override
            public void run() {
                TraceIdUtils.newTraceId(traceId);
                command.run();
                TraceIdUtils.removeTraceId();
            }
        });
    }

    public static <T> Future<T> submit(final Callable<T> task) {
        final String traceId = TraceIdUtils.getTraceId();
        return executor.submit(new Callable<T>() {
            @Override
            public T call() throws Exception {
                TraceIdUtils.newTraceId(traceId);
                T res = task.call();
                TraceIdUtils.removeTraceId();
                return res;
            }
        });
    }
}
```

## SpringBoot 注解 @Async

### AOP & 约定

使用 AOP 以及约定相同前缀的函数名如: async......

``` java
/**
* 实现异步服务的链路跟踪, traceId 的设置
*
* @author ssf on 2020-06-11
*/
@Aspect
@Component
public class AsyncServiceTracer {
    private final static Logger logger = LoggerFactory.getLogger(AsyncServiceTracer.class);

    @Pointcut("execution(* a.b.c.*.manager.impl.*ManagerImpl.async*(..))")
    public void trace() {

    }


    @Before(value = "trace()")
    public void before(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();
        if (args != null && args.length > 0 && args[0] instanceof DelegateTaskDTO) {
            DelegateTaskDTO<?> delegateTask = (DelegateTaskDTO<?>) args[0];
            TraceUtil.setTraceId(delegateTask.getDelegateId());
        } else {
            logger.info("async delegate task no delegateId");
            TraceUtil.setTraceId();
        }
    }

    @AfterReturning(pointcut = "trace()")
    public void after(JoinPoint joinPoint) {
        TraceUtil.removeTraceId();
    }
}
```

### 修改 SpringBoot @Async 注解配置的线程池?

未尝试, 有兴趣的可以尝试下.

### new Thread

那就只能侵入代码, 修改原有代码, 或者将 new Thread()
修改为使用公共线程池.

# 分布式 & RPC

实现单点应用, 可以大踏步就进入分布式应用.

**如何将 traceId 从 应用 A 传至应用 B 实现跨应用?**

主要以 RPC 框架 Dubbo 进行说明. 主要思想就是使用其 **上下文对象** 存放
traceId, 将 traceId 从应用 A 传至应用 B.

## Dubbo

Dubbo 使用过滤器功能对 Dubbo 上下文对象 RpcContext 进行操作

``` java

import org.apache.dubbo.rpc.*;

/**
 * @author ssf on 2020-06-09
 */
public class TraceIdFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        RpcContext rpcContext = RpcContext.getContext();
        // provider
        if (rpcContext.isProviderSide()) {
            String traceId = rpcContext.getAttachment("traceId");
            MDC.put("traceId", traceId);
        } else if (rpcContext.isConsumerSide()) {
            rpcContext.setAttachment("traceId", MDC.get("traceId"));
        }

        Result result = invoker.invoke(invocation);

        if (rpcContext.isProviderSide()) {
            TraceUtil.removeTraceId();
        }
        return result;
    }
}
/// 新增文件 META-INF/dubbo/com.alibaba.dubbo.rpc.Filter
/// 内容:
/// traceIdFilter=a.b.c.dubbo.filter.TraceIdFilter
```

## SpringCloud

SpringCloud 没进行实操, 其解决思路大致为: 因为 SpringCloud 的 RPC
交互协议为 HTTP 协议, 可以将 traceId 放入 Headers 中传送至下一个应用服务
B, 应用 B 在接受到请求将 Headers 中的 traceId 取出放至 MDC

# [TODO]{.todo .TODO} 其他组件 {#其他组件}

将 TraceID 带入 MessageQueue 的上下文信息中, 实现 biz chain
业务链路的一体化

