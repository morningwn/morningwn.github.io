---
title: JVM 中都有哪些线程池：实现原理、如何使用与注意事项
summary: 从 ThreadPoolExecutor、ScheduledThreadPoolExecutor、ForkJoinPool 到 Executors 各种工厂方法，系统梳理 Java/JVM 应用里常见线程池的分类、内部实现、使用方式与生产实践注意事项。
created: 2026-06-20
updated: 2026-06-25
tags: JVM, 线程池, 并发, ThreadPoolExecutor, ForkJoinPool
---

# JVM 中都有哪些线程池：实现原理、如何使用与注意事项

线程池是 Java 并发编程里最常用、也最容易被误用的基础设施之一。很多线上问题看起来是“接口慢”“CPU 飙高”“线程打满”“任务堆积”，本质上都和线程池配置或使用方式有关。

很多文章会直接列出 `newFixedThreadPool`、`newCachedThreadPool`、`newSingleThreadExecutor` 这些 API，但如果只停留在“会用”，实际遇到任务堆积、阻塞、拒绝策略、上下文丢失、定时任务漂移时，往往很难判断问题在哪。

这篇文章会从三个层面展开：

- Java/JVM 应用开发里常见的线程池到底有哪些。
- 这些线程池底层是怎么实现的。
- 实际使用时应该怎么选、怎么配、怎么避坑。

先说一个容易混淆但必须讲清楚的点。

严格来说，线程池主要是 Java 并发库提供给业务代码使用的能力，而不是 JVM 规范里专门定义的一种“内建业务线程池类型”。JVM 自己内部也有很多线程，比如 GC 线程、JIT 编译线程、信号处理线程、引用处理线程，但它们属于虚拟机运行时实现的一部分，不是业务代码通常所说的“线程池”。

业务开发里讨论“JVM 有哪些线程池”，通常指的是 Java 标准库 `java.util.concurrent` 里的这几类执行器：

- 基于 `ThreadPoolExecutor` 的通用线程池。
- 基于 `ScheduledThreadPoolExecutor` 的定时线程池。
- 基于 `ForkJoinPool` 的工作窃取线程池。
- 基于 `Executors` 工厂方法创建出来的若干包装线程池。

理解这些类型时，最重要的一条主线是：

- 线程池对外暴露的是任务提交与执行能力。
- 真正决定行为的是任务队列、线程创建策略、线程回收策略、拒绝策略以及调度模型。

## 一、为什么需要线程池

如果每来一个请求就新建一个线程，会出现几个问题：

- 线程创建和销毁本身有成本。
- 线程数量失控会导致频繁上下文切换。
- 线程栈、任务对象、队列对象会持续占用内存。
- 没有统一的流量控制时，系统在高峰期很容易被打穿。

线程池本质上是把“任务”和“线程”解耦：

- 业务代码提交的是任务。
- 线程池决定任务什么时候执行、由哪些线程执行、排队多久、满了以后怎么办。

所以线程池的核心价值并不只是“复用线程”，而是：

- 控制并发度。
- 建立排队与背压机制。
- 统一线程生命周期管理。
- 为监控、隔离、降载提供抓手。

## 二、Java/JVM 应用开发里常见的线程池有哪些

从使用层面看，最常见的是下面几类。

### 1. FixedThreadPool

固定大小线程池，通常通过 `Executors.newFixedThreadPool(n)` 创建。

它的特点是：

- 核心线程数等于最大线程数。
- 线程数基本固定。
- 多余任务进入阻塞队列等待。

适合任务量比较稳定、希望严格限制并发线程数的场景。

### 2. SingleThreadExecutor

单线程线程池，通常通过 `Executors.newSingleThreadExecutor()` 创建。

它的特点是：

- 池中始终只有一个工作线程。
- 所有任务串行执行。
- 如果工作线程异常退出，线程池会补一个新线程继续执行后续任务。

适合需要串行化处理、又不想手动管理线程生命周期的场景。

### 3. CachedThreadPool

可缓存线程池，通常通过 `Executors.newCachedThreadPool()` 创建。

它的特点是：

- 基本不排队。
- 来一个任务就尽量直接交给线程执行。
- 没有空闲线程时会继续创建新线程。
- 空闲线程达到超时时间后会被回收。

适合短小、突发、异步化程度高的任务，但如果使用不当，风险很大。

### 4. ScheduledThreadPool

定时线程池，通常通过 `Executors.newScheduledThreadPool(n)` 创建。

它的特点是：

- 支持延迟执行。
- 支持固定频率执行。
- 支持固定间隔执行。

适合定时任务、轮询任务、延迟任务。

### 5. SingleThreadScheduledExecutor

单线程定时线程池，通常通过 `Executors.newSingleThreadScheduledExecutor()` 创建。

它本质上是只有一个核心线程的定时线程池，适合严格串行的定时任务。

### 6. WorkStealingPool

工作窃取线程池，通常通过 `Executors.newWorkStealingPool()` 创建，底层本质是 `ForkJoinPool`。

它的特点是：

- 每个工作线程维护自己的双端队列。
- 本地任务优先在本线程内处理。
- 空闲线程可以去“偷”其他线程队列里的任务。

适合可以拆分成很多小任务、且任务之间相对独立的并行计算场景。

### 7. 自定义 ThreadPoolExecutor

生产环境里最常见、也最推荐的方式，通常直接 new 一个 `ThreadPoolExecutor`。

因为大多数线上系统都需要明确控制：

- 核心线程数。
- 最大线程数。
- 队列容量。
- 空闲回收时间。
- 线程命名。
- 拒绝策略。

真正的工程实践里，线程池的重点不是“会不会用 Executors 工厂”，而是“能不能根据业务特征把参数配对”。

## 三、ThreadPoolExecutor 是最核心的线程池实现

`ThreadPoolExecutor` 是 Java 并发库里最核心的通用线程池实现。很多表面上不同的线程池，底层都只是它的不同参数组合。

先看它的构造参数：

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,
    maximumPoolSize,
    keepAliveTime,
    TimeUnit.SECONDS,
    workQueue,
    threadFactory,
    handler
);
```

这几个参数几乎决定了线程池的全部行为。

### 1. corePoolSize

核心线程数。

可以把它理解成线程池希望长期维持的工作线程数量。需要注意，核心线程默认是按需创建的，不是在线程池初始化时一次性全部创建；创建后即使空闲，也不会立即销毁。

### 2. maximumPoolSize

最大线程数。

当核心线程都忙、队列又放不下任务时，线程池才会继续扩容到这个上限。

### 3. keepAliveTime

非核心线程的空闲存活时间。

如果线程数超过核心线程数，多出来的线程空闲超过这个时间后，就会被回收。

如果调用了 `allowCoreThreadTimeOut(true)`，核心线程也会按这个超时时间回收。

### 4. workQueue

任务队列。

这是线程池行为差异最大的地方之一。常见队列包括：

- `LinkedBlockingQueue`：链表阻塞队列，可选有界或无界。
- `ArrayBlockingQueue`：数组阻塞队列，固定容量。
- `SynchronousQueue`：不存储元素，提交任务时必须直接交给线程。
- `PriorityBlockingQueue`：优先级队列，适合特殊调度需求。

### 5. threadFactory

线程工厂。

用于控制线程如何创建，比如：

- 线程名称。
- 是否守护线程。
- 优先级。
- 未捕获异常处理器。

### 6. RejectedExecutionHandler

拒绝策略。

当线程池关闭，或线程数已经到最大且队列也满了时，新任务会被拒绝。JDK 内置策略有：

- `AbortPolicy`：直接抛异常。
- `CallerRunsPolicy`：由提交任务的线程自己执行。
- `DiscardPolicy`：静默丢弃。
- `DiscardOldestPolicy`：丢弃队列里最旧的任务，再尝试提交。

## 四、ThreadPoolExecutor 的执行流程是怎样的

理解线程池是否“合理配置”，一定要理解 `execute` 的核心决策流程。它可以概括成四步。

### 1. 如果工作线程数小于核心线程数，优先创建核心线程

即使队列里还没满，也先尝试新增一个线程来执行当前任务。

### 2. 如果核心线程已经满了，尝试把任务放进队列

如果队列接收成功，当前任务先排队，等待工作线程去消费。入队后线程池还会做一次运行状态校验：如果线程池已经不再接收任务，会尝试把该任务移出队列并执行拒绝策略。

### 3. 如果队列放不下，再尝试扩容到 maximumPoolSize

这一步说明线程池认为：

- 核心线程不够。
- 队列也满了。
- 需要临时增加更多线程顶住压力。

### 4. 如果扩容也失败，则执行拒绝策略

这通常意味着线程池已经达到它允许的承载极限。

把这四步记清楚后，就能理解为什么不同队列会显著改变线程池行为：

- 如果使用无界队列，线程池通常很难增长到最大线程数。
- 在无界队列配置下，`maximumPoolSize` 往往基本不生效。
- 如果使用小容量有界队列，线程池更容易触发扩容。
- 如果使用 `SynchronousQueue`，线程池几乎不排队，会优先扩线程。

## 五、ThreadPoolExecutor 的内部实现原理

如果只停留在参数层面，很多问题仍然解释不清，比如：线程池如何同时维护运行状态与线程数量，工作线程如何循环取任务，为什么线程池关闭后还会处理部分队列任务。

下面看几个关键实现点。

### 1. 用 ctl 同时维护运行状态和工作线程数

`ThreadPoolExecutor` 内部有一个很关键的整型控制字段，通常称为 `ctl`。它把一个 int 拆成两部分（高 3 位表示运行状态，低 29 位表示工作线程数）：

- 高位表示线程池运行状态。
- 低位表示当前工作线程数量。

这种设计的好处是：

- 状态和计数可以在一个原子变量上完成 CAS 更新。
- 减少额外锁竞争。
- 在高并发提交任务时，状态判断和线程数量控制更紧凑。

线程池状态大致包括：

- `RUNNING`：接受新任务，也处理队列中的任务。
- `SHUTDOWN`：不再接受新任务，但继续处理队列中已有任务。
- `STOP`：不再接受新任务，不再处理队列任务，并尝试中断工作线程。
- `TIDYING`：所有任务已结束，线程数为 0，准备收尾。
- `TERMINATED`：终止完成。

这也是为什么 `shutdown()` 和 `shutdownNow()` 的语义完全不同。

### 2. Worker 是线程池里的实际工作单元

线程池内部不是简单维护一个裸线程列表，而是维护 `Worker` 对象。

`Worker` 一般会包含：

- 一个真正执行任务的线程。
- 初始任务 `firstTask`。
- 已完成任务计数。
- 用于中断控制和运行状态协调的同步结构。

很多实现细节依赖 `Worker` 来完成，而不是直接对线程对象做散乱控制。

### 3. runWorker 是工作线程的主循环

线程池创建出工作线程后，并不是执行完一个任务就结束，而是进入一个持续循环：

- 先执行 `firstTask`。
- 然后不断从队列里取任务。
- 取到任务就执行。
- 取不到且满足退出条件时，工作线程回收。

伪代码可以理解成这样：

```java
while (task != null || (task = getTask()) != null) {
    Throwable thrown = null;
    beforeExecute(thread, task);
    try {
        task.run();
    } catch (RuntimeException | Error x) {
        thrown = x;
        throw x;
    } catch (Throwable x) {
        thrown = x;
        throw new Error(x);
    } finally {
        afterExecute(task, thrown);
    }
    task = null;
}
processWorkerExit(worker);
```

这段模型非常重要，因为它解释了几个常见现象：

- 线程池中的线程会复用，而不是每个任务一个线程。
- 执行钩子 `beforeExecute` 和 `afterExecute` 可以做监控扩展。
- 工作线程退出时会触发清理与补线程逻辑。

### 4. getTask 决定线程是否继续存活

工作线程能不能继续活着，核心取决于 `getTask()`。

它通常会综合判断：

- 线程池当前状态。
- 当前工作线程数是否超过核心线程数。
- 是否允许核心线程超时。
- 从队列取任务时是否超时。

所以线程池的“回收线程”不是某种后台神秘机制，而是工作线程在获取任务时根据状态主动退出。

### 5. 线程池为什么要加锁

很多人以为线程池只依赖 CAS。实际上不是。

线程池内部既使用 CAS，也使用显式锁，它们各自负责不同问题：

- CAS 适合处理高频、短路径的状态更新，比如工作线程数变化。
- 锁适合保护工作线程集合、最大线程数变化、关闭流程等复合操作。

这是一种典型的并发实现思路：

- 热路径用原子操作降低竞争。
- 复杂一致性操作用锁保证正确性。

## 六、Executors 工厂方法到底做了什么

很多人日常用的是 `Executors`，但这个类只是提供了若干线程池的快捷构造方法。它本身不是线程池实现。

### 1. newFixedThreadPool

大致等价于：

```java
new ThreadPoolExecutor(
    nThreads,
    nThreads,
    0L,
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>()
)
```

关键点在于：默认使用的是容量接近无界（`Integer.MAX_VALUE`）的 `LinkedBlockingQueue`。

这意味着：

- 线程数不会继续增长。
- 高峰期多余任务会不断积压在队列里。
- 如果任务生产速度持续大于消费速度，可能导致内存占用持续上升。

### 2. newSingleThreadExecutor

大致等价于核心线程数和最大线程数都为 1 的 `ThreadPoolExecutor`，同样默认使用无界队列。

优点是串行、稳定，缺点是如果前面一个任务卡住，后面所有任务都会排队。

### 3. newCachedThreadPool

大致等价于：

```java
new ThreadPoolExecutor(
    0,
    Integer.MAX_VALUE,
    60L,
    TimeUnit.SECONDS,
    new SynchronousQueue<Runnable>()
)
```

它最关键的特征是 `SynchronousQueue`：

- 队列本身不存任务。
- 提交任务时必须立刻交给某个线程。
- 如果没有空闲线程接手，就新建线程。

所以它在任务突发时扩容非常快，但如果任务执行较慢或存在阻塞，也会非常快地把线程数撑高。

### 4. newScheduledThreadPool

底层是 `ScheduledThreadPoolExecutor`，它本质上继承了 `ThreadPoolExecutor`，但用了自己的延时队列和调度逻辑。

### 5. newWorkStealingPool

底层是 `ForkJoinPool`。无参版本通常以 `Runtime.getRuntime().availableProcessors()` 作为目标并行度，且不保证任务执行顺序。

## 七、ScheduledThreadPoolExecutor 的原理与使用方式

如果任务不仅要“执行”，还要“在某个时间点执行”或者“周期性执行”，通用线程池就不够了，需要 `ScheduledThreadPoolExecutor`。

### 1. 它解决的是什么问题

普通 `ThreadPoolExecutor` 只关心：

- 有没有线程。
- 队列里有没有任务。

它不关心“任务什么时候才允许执行”。

而定时线程池要额外解决：

- 哪些任务还没到执行时间。
- 哪个任务最早到期。
- 周期任务下一次调度时间如何计算。

### 2. 底层核心是延时队列

`ScheduledThreadPoolExecutor` 内部使用的是一种延时优先队列，JDK 里通常是 `DelayedWorkQueue`。

队列会按“下次触发时间”排序：

- 最早到期的任务排在前面。
- 工作线程取任务时，如果队首还没到时间，就继续等待。

所以定时线程池不是靠不断轮询整个队列实现的，而是靠“按最早截止时间唤醒”。

### 3. 两种周期任务的区别

定时线程池最容易混淆的是这两个 API：

- `scheduleAtFixedRate`
- `scheduleWithFixedDelay`

它们的区别非常关键。

#### scheduleAtFixedRate

按固定频率执行。

假设周期设为 10 秒，那么理论上的触发时间是：

- 第 1 次：0 秒
- 第 2 次：10 秒
- 第 3 次：20 秒

如果某次执行时间超过周期，下一次会在本次结束后尽快触发（表现为“追赶”），但同一个任务实例不会并发重入执行。

适合心跳、监控采样、固定节拍任务。

#### scheduleWithFixedDelay

按固定间隔执行。

它是“上一次执行结束后，再等待 delay 时间再执行下一次”。

适合需要保证两次执行之间有明确间隔的任务，比如周期性同步、轮询外部系统。

### 4. 如何使用

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

scheduler.schedule(() -> {
    System.out.println("5 秒后执行一次");
}, 5, TimeUnit.SECONDS);

scheduler.scheduleAtFixedRate(() -> {
    System.out.println("固定频率执行");
}, 0, 10, TimeUnit.SECONDS);
```

### 5. 使用时的注意事项

- `scheduleAtFixedRate` 或 `scheduleWithFixedDelay` 的周期任务如果某次执行抛出未捕获异常，该任务后续调度通常会被取消。
- 当可用工作线程不足时，单个任务执行时间过长会拉高同池其他任务的调度延迟。
- 不要把耗时 I/O、大量阻塞操作塞进小型定时线程池。

## 八、ForkJoinPool 的原理与使用方式

`ForkJoinPool` 和普通线程池的差异非常大。它不是围绕“共享阻塞队列”设计的，而是围绕“分治任务”和“工作窃取”设计的。

### 1. 适合什么场景

适合可以拆成很多子任务的问题，比如：

- 大数组分段计算。
- 并行递归处理。
- 图遍历、树遍历。
- 并行流计算。

不适合长时间阻塞、强依赖 I/O 等场景。

### 2. 什么是 fork/join

它的基本思想是：

- 大任务拆成小任务，这一步叫 fork。
- 子任务执行完成后合并结果，这一步叫 join。

典型代码模型如下：

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int start;
    private final int end;

    SumTask(long[] arr, int start, int end) {
        this.arr = arr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= 1000) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += arr[i];
            }
            return sum;
        }

        int mid = (start + end) >>> 1;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();
        long rightResult = right.compute();
        long leftResult = left.join();
        return leftResult + rightResult;
    }
}
```

### 3. 工作窃取是怎么做的

在普通线程池里，大家通常共享一个任务队列。

而在 `ForkJoinPool` 中：

- 每个工作线程维护一个本地双端队列。
- 自己提交的小任务优先从本地队列取。
- 本地没活干时，再去别的线程队列尾部偷任务。

这样做的好处是：

- 本地任务命中率更高。
- 线程间争用更少。
- 更适合大量细粒度任务。

### 4. commonPool 是什么

很多框架和 API 默认会用到 `ForkJoinPool.commonPool()`，比如：

- `parallelStream()`
- 一部分 `CompletableFuture` 异步方法

这意味着如果你在 commonPool 里跑了很多阻塞任务，可能会连带影响其他默认依赖它的并行逻辑。
另外，commonPool 的默认并行度通常是 `max(1, availableProcessors - 1)`，和 `newWorkStealingPool()` 的默认目标并行度并不完全一样。

### 5. 使用时的注意事项

- 不要把大量阻塞 I/O 任务直接扔进 `ForkJoinPool`。
- 任务粒度不要过细，否则拆分和调度成本会超过并行收益。
- 如果必须阻塞，可以评估 `ManagedBlocker` 之类机制，但业务系统更常见的做法是改用专用线程池。

## 九、如何正确使用线程池

线程池的正确使用，核心不是“选一个 API”，而是围绕任务类型建立明确的执行模型。

### 1. 优先自己显式创建 ThreadPoolExecutor

生产环境更推荐直接构造：

```java
ThreadFactory threadFactory = r -> {
    Thread thread = new Thread(r);
    thread.setName("order-worker-" + thread.getId());
    return thread;
};

BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(200);

ThreadPoolExecutor executor = new ThreadPoolExecutor(
    8,
    16,
    60,
    TimeUnit.SECONDS,
    queue,
    threadFactory,
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

这段配置的含义是：

- 常驻 8 个线程。
- 峰值最多扩到 16 个线程。
- 队列最多堆 200 个任务。
- 满载后由调用方线程自己执行，形成反压。

### 2. 根据任务类型估算线程数

线程池大小不能靠拍脑袋。一个常见经验是：

- CPU 密集型任务：线程数通常接近 CPU 核数或核数加一。
- I/O 密集型任务：线程数通常可以高于 CPU 核数，但必须结合阻塞比例和压测结果。

如果需要一个更可操作的起点，可用经验公式：`线程数 ≈ CPU 核数 × 目标 CPU 利用率 × (1 + 等待时间/计算时间)`。

原因很简单：

- CPU 密集型线程过多，只会增加上下文切换。
- I/O 密集型线程过少，会导致 CPU 经常空转等待。

不过经验值只能当起点，真正上线前仍然要靠压测数据和监控指标修正。

### 3. 不同类型任务不要混用一个线程池

这是生产环境最常见的问题之一。

例如：

- HTTP 调用任务。
- 数据库写入任务。
- 本地 CPU 计算任务。
- 定时补偿任务。

如果全塞进同一个线程池，一种任务堆积就会拖垮其他任务。正确方式是按业务职责或资源模型拆池，至少做到：

- 阻塞任务和计算任务分开。
- 核心链路和低优先级任务分开。
- 实时任务和离线任务分开。

### 4. 线程池必须可观测

如果线程池没有暴露监控，你几乎不可能在问题发生前发现它快满了。

至少建议监控这些指标：

- 当前线程数。
- 活跃线程数。
- 队列长度。
- 已完成任务数。
- 任务拒绝次数。
- 平均执行时长。
- 最大执行时长。

### 5. 要有明确的关闭逻辑

线程池创建后如果不关闭，应用退出流程、测试流程、容器重启流程都可能出问题。

标准关闭方式通常是：

```java
executor.shutdown();
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    executor.shutdownNow();
}
```

这里的语义是：

- 先平滑关闭，不再接收新任务。
- 等待已提交任务执行完成。
- 超时后再尝试中断。

## 十、生产环境中最需要注意的坑

下面这些问题最常见，也最值得重点记住。

### 1. 不要迷信 Executors 默认工厂

很多团队规范里都会明确要求：生产代码避免直接使用 `Executors.newFixedThreadPool()` 或 `Executors.newCachedThreadPool()`。

不是因为它们不能用，而是因为默认参数过于隐蔽：

- `newFixedThreadPool` 默认无界队列，容易把问题从“线程满”变成“内存涨”。
- `newSingleThreadExecutor` 也是无界队列。
- `newCachedThreadPool` 最大线程数接近无上限，容易把问题从“排队”变成“线程爆炸”。

所以工程上更稳妥的做法是显式 new `ThreadPoolExecutor`。

### 2. 线程池满了必须知道怎么退化

拒绝策略不是一个“兜底选项”，而是系统容量边界的一部分。

例如：

- 核心链路任务如果被丢弃，可能是严重事故。
- 非核心异步任务在高峰时允许丢弃，反而是合理策略。
- `CallerRunsPolicy` 虽然能形成反压，但也可能拖慢调用线程。

拒绝策略必须和业务可接受的失败模式匹配。

### 3. 小心 ThreadLocal 与上下文污染

线程池线程会复用，这意味着：

- 一个任务留下的 `ThreadLocal` 值，可能污染下一个任务。
- 日志链路追踪、租户信息、用户上下文如果不清理，会出现串数据。

因此提交任务时如果依赖上下文，必须明确：

- 是否做上下文传递。
- 执行结束后是否清理上下文。

### 4. 注意异常不会自动帮你兜底

如果用 `execute()` 提交任务，任务内部抛异常时，不会像调用普通方法那样在提交线程直接暴露；异常通常会交给工作线程的 `UncaughtExceptionHandler`（默认情况下可能只是打印日志）。

如果用 `submit()` 提交任务，异常通常会被包在 `Future` 里，只有调用 `get()` 时才会抛出。

这意味着：

- 异步任务出错可能悄无声息。
- 如果不做日志和监控，问题会长期潜伏。

### 5. 不要在线程池任务里长时间阻塞

例如：

- 长时间等待外部接口。
- 大量同步锁竞争。
- 无限期等待队列或 CountDownLatch。

这些行为会直接吞掉线程池里的工作线程，导致后续任务无法及时处理。

### 6. 队列不是越大越好

很多人觉得“队列大一点更安全”，其实未必。

队列过大意味着：

- 任务延迟被拉长。
- 高峰期问题暴露变慢。
- 系统表面上不报错，但实际上已经进入严重积压状态。

一个合理的有界队列，通常比一个巨大的无界队列更容易让系统维持可控。

### 7. 线程名称一定要可读

如果线程名全是默认格式，排查问题时会很痛苦。

建议线程名包含：

- 系统名或模块名。
- 线程池职责。
- 线程编号。

例如：

- `order-async-1`
- `report-export-3`
- `inventory-retry-2`

### 8. ForkJoinPool 不适合作为万能异步池

默认 commonPool 很方便，但也很容易被误用。

如果你的任务：

- 强阻塞。
- 耗时长。
- 需要强隔离。

那就不应该依赖 commonPool，而应该使用专用线程池。

## 十一、几种典型线程池怎么选

如果只想快速建立选型直觉，可以按下面这张思路来判断。

### 1. 严格串行执行

选择：`SingleThreadExecutor` 或单线程 `ThreadPoolExecutor`

适用：顺序消费、串行状态机、单线程事件处理。

### 2. 常规异步任务处理

选择：自定义 `ThreadPoolExecutor`

适用：接口异步化、业务解耦、消息消费、后台任务。

### 3. 延迟任务与周期任务

选择：`ScheduledThreadPoolExecutor`

适用：轮询、补偿、心跳、定时同步。

### 4. 大量可拆分的并行计算

选择：`ForkJoinPool`

适用：分治计算、并行遍历、并行流。

### 5. 突发短任务且能接受快速扩线程

选择：谨慎评估 `CachedThreadPool`，或者更可控的自定义池

适用：任务极短、突发明显、阻塞很少的场景。

## 十二、一个更工程化的理解方式

把线程池看成一个小型调度系统，会更容易做出正确决策。

它至少包含四个问题：

- 任务从哪里来。
- 任务在哪里等。
- 谁去执行。
- 满了以后怎么办。

对应到线程池参数，就是：

- 提交方流量模型。
- 队列模型。
- 线程模型。
- 拒绝与降级模型。

一旦用这个视角看线程池，你会发现很多“参数调优”问题其实是业务容量设计问题，而不是单纯的 API 使用问题。

## 十三、总结

Java/JVM 应用开发里常见的线程池，核心可以归成三大类：

- `ThreadPoolExecutor`：通用任务执行线程池。
- `ScheduledThreadPoolExecutor`：定时与延迟任务线程池。
- `ForkJoinPool`：基于工作窃取的并行计算线程池。

而 `Executors` 提供的那些常用工厂方法，本质上只是这几类实现的预设参数包装。

真正决定线程池行为的，不是名字，而是：

- 线程数怎么增长。
- 队列能不能积压。
- 空闲线程何时回收。
- 满载时如何拒绝。
- 任务是否阻塞。

如果要记住最重要的实践结论，可以只保留下面几条：

- 生产环境优先显式创建 `ThreadPoolExecutor`。
- 不同类型任务分池隔离。
- 队列尽量有界，拒绝策略要和业务语义匹配。
- 定时任务关注异常、漂移和阻塞。
- `ForkJoinPool` 适合分治计算，不适合拿来兜所有异步任务。

线程池配置没有放之四海而皆准的万能参数，只有基于任务模型、机器资源、延迟目标和压测结果得到的相对最优解。理解实现原理之后，再去做参数选择，很多线上问题就能在设计阶段提前避免。