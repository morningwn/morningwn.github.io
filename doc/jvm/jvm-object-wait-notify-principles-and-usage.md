---
title: Object.wait/notify/notifyAll 的作用、原理与使用方式
summary: 从对象监视器、等待集、线程状态流转、超时语义、虚假唤醒到生产实践，系统讲清 Object.notify()、notifyAll() 与 wait() 三组方法的作用、底层原理、正确写法和常见误区。
created: 2026-06-26
updated: 2026-07-01
tags: JVM, 并发, 线程同步, wait, notify
---

# Object.wait/notify/notifyAll 的作用、原理与使用方式

`Object.notify()`、`Object.notifyAll()`、`Object.wait()` 以及它的两个带超时参数的重载，是 Java 最基础的一组线程通信原语。

很多人知道它们要配合 `synchronized` 用，但一旦追问下面这些问题，往往就会开始混乱：

- `wait()` 和 `sleep()` 到底有什么本质区别。
- 为什么 `wait()` 一定要写在 `while` 里，而不是 `if`。
- `notify()` 唤醒的线程是不是马上开始执行。
- `notifyAll()` 会不会把所有线程都同时跑起来。
- `wait(long timeoutMillis, int nanos)` 里的 `nanos` 到底是什么意思。

这篇文章只围绕这五个方法展开，重点解释三件事：

- 它们分别起什么作用。
- JVM 监视器机制下它们是怎么工作的。
- 业务代码里应该怎么正确使用，哪些写法一定要避免。

## 一、先给结论：这几个方法是干什么的

这五个方法都定义在 `java.lang.Object` 上，而不是定义在 `Thread` 上。这一点很关键，因为它们操作的核心对象不是“线程本身”，而是“对象监视器”。

先看最简定义：

- `Object.wait()`：当前线程进入等待，直到被唤醒或被中断。
- `Object.wait(long timeoutMillis)`：当前线程进入等待，但最多等待指定毫秒数。
- `Object.wait(long timeoutMillis, int nanos)`：当前线程进入等待，但最多等待“毫秒 + 纳秒补充量”这段时间。
- `Object.notify()`：唤醒一个正在等待当前对象监视器的线程（具体唤醒哪个由 JVM 实现决定）。
- `Object.notifyAll()`：唤醒所有正在等待当前对象监视器的线程。

注意这里有两个限定语不能省：

- 等待的是“当前对象的监视器”。
- 唤醒的也是“等待当前对象监视器”的线程。

也就是说，它们不是一个全局的线程控制开关，而是某个共享对象上的条件等待与通知机制。

## 二、为什么这些方法定义在 Object 上，而不是 Thread 上

这是理解整套机制的第一步。

Java 的内置同步模型基于监视器，也就是 monitor。每个对象都可以天然作为一把锁使用：

- 进入 `synchronized(obj)`，本质上是在竞争 `obj` 的监视器。
- 调用 `obj.wait()`，本质上是当前线程进入 `obj` 的等待集。
- 调用 `obj.notify()` 或 `obj.notifyAll()`，本质上是从 `obj` 的等待集中唤醒线程。

所以这套 API 的关注点一直都是“对象关联的条件队列”，而不是“线程自己怎么睡、怎么醒”。

如果它们定义在 `Thread` 上，就会出现一个语义问题：

- 线程到底是在等哪个共享条件。
- 唤醒动作又应该和哪把锁绑定。

定义在 `Object` 上，语义就统一了：

- 某个线程持有某个对象的监视器。
- 如果条件不满足，就在这个对象上等待。
- 条件变化后，另一个持有同一监视器的线程负责通知。

## 三、先建立模型：监视器、入口集、等待集分别是什么

想真正理解 `wait/notify`，需要先有一个最小运行模型。

对任意一个可作为锁使用的对象 `lock`，你可以把它想成同时维护了三类状态：

- 当前持有监视器的线程，也就是 owner。
- 想进入 `synchronized(lock)` 但还没拿到锁的线程集合。
- 已经调用了 `lock.wait(...)` 并主动释放了 `lock` 的线程集合。

这里第二类通常可以理解成“锁竞争队列”或“入口集”，第三类就是“等待集”。

一个典型流程如下：

1. 线程 A 进入 `synchronized(lock)`，拿到 `lock` 的监视器。
2. A 发现条件不满足，于是调用 `lock.wait()`。
3. A 进入 `lock` 的等待集，同时释放 `lock` 的监视器。
4. 线程 B 进入 `synchronized(lock)`，修改共享状态。
5. B 调用 `lock.notifyAll()` 或 `lock.notify()`。
6. 被唤醒的线程不会立刻继续执行，而是先回到锁竞争阶段。
7. 等 B 退出同步块、释放监视器后，某个被唤醒线程重新拿到锁，再从 `wait()` 返回。

这也是为什么很多初学者会误解：

- `notify()` 不是“立即让对方线程执行”。
- 它只是“把等待线程从等待集挪到可竞争锁的状态”。

## 四、这几个方法的共同前提：必须先持有监视器

这五个方法有一个共同前提：

- 当前线程必须是目标对象监视器的持有者。

最常见的合法写法是：

```java
final Object lock = new Object();

synchronized (lock) {
    lock.wait();
}
```

或者：

```java
final Object lock = new Object();

synchronized (lock) {
    lock.notifyAll();
}
```

如果不在持有该对象监视器的前提下调用，就会抛出 `IllegalMonitorStateException`。

例如下面这种写法就是错的：

```java
final Object lock = new Object();

lock.wait();
```

为什么 JVM 要强制这个约束？因为 `wait/notify` 的核心语义就是：

- 在检查共享条件时和修改共享条件时，必须和同一把锁绑定。
- 否则等待、通知、条件修改之间就无法形成可靠的原子关系。

换句话说，这不是语法要求，而是并发正确性要求。

## 五、Object.wait() 的作用、原理与用法

### 1. 作用

`wait()` 的作用是：

- 让当前线程进入当前对象的等待集。
- 同时释放当前对象监视器。
- 直到被通知、被中断，或者出现虚假唤醒后，再重新竞争锁并返回。

它适合用于“条件不满足，先挂起等待条件变化”的场景。

最典型的是生产者消费者模型、资源就绪通知、状态机流转通知。

### 2. 原理

`wait()` 的行为并不是“单纯休眠一下”，而是一整套状态切换：

1. 当前线程必须已经持有对象监视器。
2. 当前线程把自己加入该对象的等待集。
3. 当前线程释放这个对象上的同步声明。
4. 当前线程进入等待状态，不再继续执行同步块后续代码。
5. 其他线程改变条件并通知后，它被移出等待集。
6. 它重新参与这把锁的竞争。
7. 重新拿到锁之后，`wait()` 才真正返回。

这里有两个细节经常被忽略：

- `wait()` 释放的只是“当前这个对象”的监视器，不会释放线程持有的其他锁。
- `wait()` 返回时，线程已经重新拿回了这把锁。

例如：

```java
synchronized (lockA) {
    synchronized (lockB) {
        lockA.wait();
    }
}
```

这段代码里，线程在 `lockA.wait()` 时会释放 `lockA`，但不会释放 `lockB`。这类写法非常容易制造复杂阻塞甚至死锁，生产代码通常应避免。

### 3. 正确用法

`wait()` 的标准写法一定是“条件判断 + while 循环”：

```java
final Object lock = new Object();
boolean ready = false;

public void awaitReady() throws InterruptedException {
    synchronized (lock) {
        while (!ready) {
            lock.wait();
        }
    }
}

public void markReady() {
    synchronized (lock) {
        ready = true;
        lock.notifyAll();
    }
}
```

这个模式里，`ready` 才是真正的业务条件，`wait()` 只是“条件不满足时的挂起机制”。

### 4. 为什么必须用 while，不能用 if

因为 `wait()` 可能在以下几种情况下返回：

- 被 `notify()` 唤醒。
- 被 `notifyAll()` 唤醒。
- 被中断。
- 发生超时。
- 发生虚假唤醒。

所谓虚假唤醒，就是线程没有收到你预期中的有效通知，也没有满足业务条件，但 `wait()` 仍然返回了。

虚假唤醒不是理论假设，而是真实会发生的现象。它的主要原因包括：

- JVM 的 `wait()` 底层在 HotSpot 中依赖操作系统的条件变量机制（如 Linux 的 `pthread_cond_wait`）。
- 操作系统在处理信号、中断或内部调度时，可能会让等待线程在没有收到显式通知的情况下被唤醒。
- Java 语言规范明确允许这种行为存在，因此不依赖任何特定 JVM 实现来消除它。

这意味着你不能假设"wait 返回了一定是因为收到了 notify"，只能假设"wait 返回了，需要重新检查条件"。

这意味着下面这种写法是不安全的：

```java
synchronized (lock) {
    if (!ready) {
        lock.wait();
    }
    useResource();
}
```

安全写法必须是：

```java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
    useResource();
}
```

因为线程即使被唤醒，也只能说明“你有资格重新检查条件”，不能说明“条件一定已经满足”。

### 5. 中断语义

`wait()` 会抛出 `InterruptedException`，这意味着它是可中断等待。

语义上要注意两点：

- 线程在等待前或等待中被中断，都会导致 `InterruptedException`。
- 异常抛出时，线程的中断标志会被清除。

如果上层还需要感知中断，通常要在 catch 中恢复中断标志：

```java
try {
    synchronized (lock) {
        while (!ready) {
            lock.wait();
        }
    }
} catch (InterruptedException ex) {
    Thread.currentThread().interrupt();
    return;
}
```

## 六、Object.wait(long timeoutMillis) 的作用、原理与用法

### 1. 作用

`wait(long timeoutMillis)` 的作用是：

- 在无限等待和忙等之间提供一个折中。
- 线程最多等待指定毫秒数。
- 时间到了还没被通知，也会从等待中返回并重新竞争锁。

它适合“希望等待条件成立，但又不能无限期卡住”的场景。

例如：

- 等待任务完成，但最多等 3 秒。
- 等待连接状态改变，但需要周期性检查退出标记。
- 实现简单阻塞队列时，支持带超时的获取操作。

### 2. 原理

它在语义上等价于：

```java
wait(timeoutMillis, 0);
```

也就是说，本质还是：

- 当前线程进入等待集。
- 释放当前对象监视器。
- 被通知、被中断，或者超时后重新竞争锁。

区别只是多了一个超时退出条件。

需要注意的是，实际超时精度受操作系统调度影响。线程不会在超时到达的瞬间立即返回，而是需要等待操作系统重新调度该线程。在高负载系统上，实际唤醒时间可能比指定超时晚几毫秒甚至更久。

### 3. 正确用法

带超时等待时，仍然必须写在 `while` 里，并且通常要自己维护剩余时间：

```java
public boolean awaitReady(long timeoutMillis) throws InterruptedException {
    long deadline = System.currentTimeMillis() + timeoutMillis;

    synchronized (lock) {
        long remaining = timeoutMillis;
        while (!ready) {
            if (remaining <= 0) {
                return false;
            }
            lock.wait(remaining);
            remaining = deadline - System.currentTimeMillis();
        }
        return true;
    }
}
```

这里重新计算 `remaining` 很重要，因为：

- 线程可能被提前唤醒。
- 线程被唤醒后还要重新竞争锁。
- 条件可能仍然没满足，需要继续等待。

如果不重算剩余时间，整体超时语义就会失真。

### 4. 参数约束

`timeoutMillis` 不能为负数，否则会抛出 `IllegalArgumentException`。

此外要注意：

- `wait(0)` 不是“等待 0 毫秒后立刻返回”。
- 它的含义是“不考虑时间限制，直到被唤醒或被中断”。

这点和很多超时 API 不一样，不能望文生义。

## 七、Object.wait(long timeoutMillis, int nanos) 的作用、原理与用法

### 1. 作用

这个重载方法的作用和 `wait(long timeoutMillis)` 相同，只是提供了更细粒度的时间表达：

- 毫秒部分由 `timeoutMillis` 指定。
- 额外的纳秒补充量由 `nanos` 指定。

总等待时长可以理解为：

$$
等待时间 = timeoutMillis \times 1{,}000{,}000 + nanos
$$

单位是纳秒。

不过在工程实践里，这个重载的使用频率很低，因为：

- 操作系统调度精度通常达不到这么细。
- 大多数业务场景并不需要这么细的等待粒度。
- 更常见的是基于 `LockSupport`、`Condition` 或更上层并发工具完成超时控制。

另外要注意一个实现细节：在 HotSpot JVM 中，当 `nanos > 0` 时，实际等待时间会被向上取整到至少 1 毫秒。也就是说，`wait(0, 1)` 并不会真的只等 1 纳秒，而是至少等 1 毫秒。所以这个重载并不能提供真正的纳秒级精度。

### 2. 参数规则

这个方法的参数约束比前一个更严格：

- `timeoutMillis` 不能小于 0。
- `nanos` 必须在 0 到 999999 之间。

只要越界，就会抛出 `IllegalArgumentException`。

### 3. 原理

底层语义与前两个 `wait` 一致，只是等待上限更细化：

- 先进入当前对象等待集。
- 释放当前对象监视器。
- 被通知、被中断、超时或虚假唤醒后重新竞争锁。
- 真正拿回锁后方法返回。

### 4. 使用方式

如果确实要写这个重载，依然应该放到循环里，并且配合剩余时间重算：

```java
public boolean awaitReady(long timeoutMillis, int nanos) throws InterruptedException {
    long totalNanos = timeoutMillis * 1_000_000L + nanos;
    long deadline = System.nanoTime() + totalNanos;

    synchronized (lock) {
        long remainingNanos = totalNanos;
        while (!ready) {
            if (remainingNanos <= 0) {
                return false;
            }

            long millis = remainingNanos / 1_000_000L;
            int extraNanos = (int) (remainingNanos % 1_000_000L);
            lock.wait(millis, extraNanos);

            remainingNanos = deadline - System.nanoTime();
        }
        return true;
    }
}
```

这里使用 `System.nanoTime()` 比 `System.currentTimeMillis()` 更合适，因为它适合测量时间间隔，不受系统时钟回拨影响。

## 八、Object.notify() 的作用、原理与用法

### 1. 作用

`notify()` 的作用是：

- 从当前对象的等待集中随机选择一个等待线程。
- 将它从等待状态变成“可参与锁竞争”的状态。

请注意，`notify()` 不是：

- 立刻执行被唤醒线程。
- 指定唤醒某个特定线程。
- 保证被唤醒线程下一秒就拿到锁。

JVM 只保证“唤醒一个”，但不保证“唤醒哪一个”，也不保证“它一定最先重新获得锁”。

### 2. 原理

`notify()` 执行时，当前线程仍然持有这把锁，所以被唤醒线程此时还不能继续执行。典型过程如下：

1. 线程 B 持有 `lock` 监视器。
2. B 修改共享条件。
3. B 调用 `lock.notify()`。
4. 某个等待线程 A 被从等待集中移出。
5. 但 A 还得继续等，因为 B 还没有退出同步块。
6. B 退出 `synchronized(lock)` 后释放监视器。
7. A 与其他竞争线程一起争抢 `lock`。
8. A 拿到锁以后，才从 `wait()` 返回并继续执行。

这也是为什么建议把“修改共享状态”和“notify”放在同一个同步块里。

### 3. 典型用法

当且仅当你非常确定“等待集中的线程都在等同一种条件，而且唤醒一个就够”时，可以使用 `notify()`。

例如一个单生产者、单消费者的简化模型：

```java
class OneSlotBuffer {
    private final Object lock = new Object();
    private String value;

    public void put(String newValue) throws InterruptedException {
        synchronized (lock) {
            while (value != null) {
                lock.wait();
            }
            value = newValue;
            lock.notify();
        }
    }

    public String take() throws InterruptedException {
        synchronized (lock) {
            while (value == null) {
                lock.wait();
            }
            String result = value;
            value = null;
            lock.notify();
            return result;
        }
    }
}
```

但这个例子只是为了说明机制。在复杂场景里，`notify()` 很容易用错。

### 4. 常见风险

如果多个线程等待的不是同一种条件，`notify()` 可能唤醒“错误类型”的线程，导致：

- 被唤醒线程发现条件仍不满足，又继续等待。
- 真正应该被唤醒的线程却还在沉睡。
- 整体吞吐下降，甚至出现长期卡住。

所以只要条件类型稍复杂，优先考虑 `notifyAll()`。

## 九、Object.notifyAll() 的作用、原理与用法

### 1. 作用

`notifyAll()` 的作用是：

- 唤醒当前对象等待集中的所有线程。
- 让它们都回到可竞争该对象监视器的状态。

同样要注意：

- 它不是让所有线程同时执行。
- 它只是让所有等待线程都有机会重新竞争这把锁。

### 2. 原理

调用 `notifyAll()` 后：

- 所有等待线程都会从等待集转移出来。
- 但它们仍要在当前线程释放锁后，依次竞争监视器。
- 最终只有拿到锁的线程才会先从 `wait()` 返回。

所以 `notifyAll()` 的真实效果通常不是“并行运行”，而是“批量重新检查条件”。

### 3. 为什么实际开发里通常更推荐 notifyAll()

因为它更稳妥。

当多个线程可能等待不同条件时，`notifyAll()` 虽然会带来更多无效唤醒，但它能显著降低错误唤醒导致的逻辑卡死风险。

例如一个缓冲区里同时可能有：

- 等待“非空”的消费者。
- 等待“未满”的生产者。

如果你只用 `notify()`，很可能唤醒到同类线程，结果条件仍然不满足；而 `notifyAll()` 会让所有等待线程都重新判断自己的条件，由真正满足条件的线程继续执行。

### 4. 代价是什么

`notifyAll()` 的代价主要是：

- 会唤醒更多线程。
- 增加锁竞争。
- 在高并发下可能造成无效上下文切换。

但在多数业务代码里，正确性优先级高于这一点性能代价。只有在你清楚证明 `notify()` 没有遗漏风险时，才值得用 `notify()` 做更激进的优化。

## 十、wait 和 sleep 的本质区别

这两个方法非常容易被混淆，但它们完全不是一回事。

`Thread.sleep(...)`：

- 定义在 `Thread` 上。
- 作用是让当前线程暂停一段时间。
- 不依赖任何对象监视器。
- 不会释放已经持有的锁。

`Object.wait(...)`：

- 定义在 `Object` 上。
- 作用是等待某个共享条件变化。
- 必须持有对应对象监视器。
- 会释放当前对象的监视器。

一句话概括：

- `sleep` 解决的是“线程暂时不想运行”。
- `wait` 解决的是“条件还没满足，先释放锁等通知”。

如果你在需要线程协作的场景里用 `sleep` 代替 `wait`，通常会出现两个问题：

- 锁白白被占着，其他线程无法推进条件。
- 等待时长只能靠猜，代码要么延迟过长，要么轮询过于频繁。

## 十一、正确示例：用 wait/notifyAll 实现一个简单阻塞队列

下面给一个足够小、但语义完整的例子：

```java
import java.util.ArrayDeque;
import java.util.Queue;

public class SimpleBlockingQueue<E> {
    private final Object lock = new Object();
    private final Queue<E> queue = new ArrayDeque<>();
    private final int capacity;

    public SimpleBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(E element) throws InterruptedException {
        synchronized (lock) {
            while (queue.size() == capacity) {
                lock.wait();
            }

            queue.offer(element);
            lock.notifyAll();
        }
    }

    public E take() throws InterruptedException {
        synchronized (lock) {
            while (queue.isEmpty()) {
                lock.wait();
            }

            E element = queue.poll();
            lock.notifyAll();
            return element;
        }
    }
}
```

这个例子里有几个关键点：

- 所有共享状态都在同一把 `lock` 下保护。
- 条件检查使用 `while` 而不是 `if`。
- 状态修改后立即通知。
- 使用 `notifyAll()`，因为生产者和消费者等待的是不同条件。

这已经是 `wait/notify` 这套原语最经典、也最规范的使用方式。

## 十二、常见误区与线上风险

### 1. 没有在 synchronized 里调用

这是最直接的问题，会立即抛出 `IllegalMonitorStateException`。

### 2. 用 if 包裹 wait

这会让代码暴露在虚假唤醒、错误通知和条件竞争之下。

### 3. 只调用 notify，不清楚等待线程在等什么

如果等待集中混有不同类型线程，`notify()` 很容易制造长期卡顿。

### 4. 修改条件和通知不在同一个同步块里

这会破坏条件检查与通知之间的原子性，可能造成丢失通知。

例如：

```java
ready = true;
synchronized (lock) {
    lock.notifyAll();
}
```

这种写法存在竞态条件：

1. 线程 A 调用 `awaitReady()`，检查 `ready` 为 false，准备调用 `wait()`。
2. 此时线程 B 执行 `markReady()`，将 `ready` 设为 true，然后进入同步块调用 `notifyAll()`。
3. 但此时线程 A 还没有进入等待集，所以 `notifyAll()` 没有唤醒任何线程。
4. 线程 A 随后调用 `wait()`，进入等待状态。
5. 结果：线程 A 永远不会被唤醒，即使条件已经满足。

正确写法是：

```java
synchronized (lock) {
    ready = true;
    lock.notifyAll();
}
```

这样条件修改和通知在同一个同步块内完成，保证了原子性。

### 5. 在持有多把锁时调用 wait

`wait()` 只释放当前对象的监视器，不会自动释放其他锁。多锁嵌套下很容易把阻塞链路复杂化。

### 6. 把 wait/notify 当成高层并发工具的首选

现代 Java 开发里，很多场景更适合使用：

- `BlockingQueue`
- `CountDownLatch`
- `CyclicBarrier`
- `Semaphore`
- `ReentrantLock` + `Condition`
- `CompletableFuture`

这些工具语义更明确、可读性更好、踩坑更少。

## 十三、从 JVM 角度看，这套方法到底依赖什么

如果站在 JVM 运行时角度看，`wait/notify` 依赖的是对象监视器机制。

可以把核心过程概括成下面几步：

1. 线程通过 `synchronized` 竞争对象监视器。
2. 获得监视器后，线程可以安全检查共享条件。
3. 条件不满足时，线程调用 `wait`，进入该对象的等待集。
4. JVM 让该线程释放该对象监视器并挂起。
5. 另一个线程拿到同一监视器，修改共享状态。
6. 它调用 `notify` 或 `notifyAll`，把等待线程转移到可竞争状态。
7. 当前线程退出同步块后，监视器被释放。
8. 被唤醒线程重新竞争监视器，拿到锁后恢复执行。

这也是为什么说它们是“基于监视器的条件等待原语”，而不是一般意义上的线程暂停 API。

## 十四、这五个方法分别什么时候用

可以直接记下面这张表。

| 方法 | 主要作用 | 典型场景 | 关键注意点 |
| --- | --- | --- | --- |
| `wait()` | 无限期等待条件成立 | 只接受通知或中断唤醒 | 必须配合 `while` |
| `wait(long timeoutMillis)` | 最多等待指定毫秒数 | 需要超时返回的条件等待 | 需要重算剩余时间 |
| `wait(long timeoutMillis, int nanos)` | 更细粒度超时等待 | 极少数需要更细时间表达的场景 | `nanos` 范围是 0 到 999999 |
| `notify()` | 唤醒一个等待线程 | 明确只需唤醒一个且条件单一 | 唤醒目标不可控 |
| `notifyAll()` | 唤醒全部等待线程 | 多条件等待、复杂协作场景 | 更稳妥，但竞争更大 |

## 十五、实际开发建议

如果是日常业务开发，建议按下面的顺序做选择：

1. 能用现成并发容器或同步工具，就不要自己手写 `wait/notify`。
2. 必须手写时，优先使用单一锁对象保护共享状态。
3. 条件等待一律写成 `while`。
4. 多条件或不确定场景优先用 `notifyAll()`。
5. 对中断要有清晰策略，不要随手吞掉 `InterruptedException`。
6. 带超时等待时，一定自己维护剩余时间，而不是只调用一次超时等待。

## 十六、小结

`Object.wait()`、`Object.wait(long timeoutMillis)`、`Object.wait(long timeoutMillis, int nanos)`、`Object.notify()`、`Object.notifyAll()` 的本质，都是围绕“对象监视器 + 条件等待”展开的。

可以把它们浓缩成下面几句话：

- `wait` 是“条件不满足时，释放锁并进入等待”。
- `notify` 是“唤醒一个等待者去重新竞争锁”。
- `notifyAll` 是“唤醒所有等待者去重新检查条件”。
- `wait` 返回不代表条件成立，只代表应该重新检查条件。
- 正确模式永远是“同一把锁保护条件 + while 循环等待 + 状态变化后通知”。

如果只是理解 JVM 和 Java 并发基础，这套方法必须掌握；如果是写生产代码，则更重要的是知道它们的边界，并在合适的时候优先选择更高层的并发工具。