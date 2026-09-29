---
title: JVM 在什么时机会卸载类
summary: 从定义类加载器的可达性出发，说明类取得卸载资格、JVM 实际执行卸载以及排查类加载器泄漏的方法。
created: 2026-08-07
updated: 2026-08-12
tags: JVM, 类卸载, 类加载器, GC, Metaspace
cover: /img/jvm/jvm-class-unloading-cover.webp
---

决定一个类能否被卸载的，不是它还有没有存活实例，而是**定义它的类加载器是否可以被垃圾收集器回收**。本文以 Java SE 25 规范与 HotSpot JVM 为实现范围展开说明。

## 先给出结论：类什么时候会被卸载

[Java 语言规范 §12.7](https://docs.oracle.com/javase/specs/jls/se25/html/jls-12.html#jls-12.7) 给出的正式条件是：一个类或接口可以被卸载，当且仅当定义它的类加载器可以被垃圾收集器回收。由 Bootstrap ClassLoader 加载的类不得卸载。

这个定义实际上包含两个不同的时刻，需要分开理解：

1. **取得卸载资格**：定义类加载器已经不可达，因此它定义的类也不可能再被程序使用。
2. **实际完成卸载**：JVM 在之后的某次垃圾回收过程中识别并卸载这些类。

所以，"最后一个实例被回收"并不是类被卸载的时刻。实例消失至多消除了一条通往加载器的引用路径；只要定义类加载器仍然可达，类就不能被卸载。反过来，类加载器不可达也只是说明类"允许"被卸载，规范并不要求 JVM 立刻执行 GC 或立刻完成卸载。

```mermaid
flowchart LR
    A[业务停止使用一组类] --> B[断开实例、Class 对象与加载器的外部引用]
B --> C[定义类加载器不可达]
C --> D[这些类取得卸载资格]
D --> E[后续 GC 检查类卸载]
E --> F[类被卸载并释放对应元数据]
```

## 为什么类卸载取决于类加载器

同名类由不同的定义类加载器加载后，在 JVM 中是两个不同的运行时类型。定义类加载器不仅负责"产生"类，还划定了类的生命周期边界。

试想：如果 JVM 在定义类加载器仍可能被使用时卸载其中的类，那么加载器之后完全可能再次加载同名类。新旧 `Class` 对象的身份不同，静态字段的状态、静态初始化的副作用以及本地状态都无法透明恢复。因此，规范禁止卸载由仍可能可达的加载器定义的类。

这也解释了为什么类卸载总是表现为"一组类共同退出"：一个自定义类加载器通常会定义多个类，只有加载器本身可回收时，这一组类才同时具备卸载资格。HotSpot 为每个类加载器分配承载类元数据的 Metaspace 内存块，卸载某个加载器的全部类之后，对应内存块可以被复用或返还给操作系统。需要说明的是，Metaspace 的具体管理方式属于 HotSpot 实现细节，并不是 Java 语言规范层面的保证。

## 从对象死亡到类可以卸载

判断一个类能否被卸载，正确做法是从 GC Roots 出发，沿着引用方向检查定义类加载器是否仍然可达，而不是机械地核对几个孤立条件。

### 实例不再被使用

存活实例依赖其运行时类，运行时类又关联定义类加载器。因此，只要仍有可达实例，加载器便不能回收。

但实例全部消失仍然不够。程序可以直接保存 `Class<?>`、类加载器，也可以通过其他对象间接保存它们，这些引用同样会延长加载器的生命周期。

### `Class` 对象不再被外部保留

反射缓存、序列化缓存、依赖注入容器或全局集合都可能保存 `Class<?>`。一个可达的 `Class` 对象会让其代表的类保持活动状态，进而阻止定义类加载器被回收。

这里的关键是"被外部保留"。类内部的静态字段并不会凭空成为 GC Root；只有当某条来自 GC Roots 的路径能够到达该类、其实例或加载器时，整条对象图才会继续存活。

### 定义类加载器不再可达

即使没有存活实例，也没有显式的 `Class<?>` 缓存，应用仍可能直接保存类加载器本身。比如插件管理器删除了插件对象，却没有把对应的 `ClassLoader` 从加载器注册表中移除，此时该加载器定义的所有类都无法卸载。

因此，把"实例、`Class` 对象、类加载器都不可达"当作排查清单是可行的，但规范层面的决定性条件始终只有一条：定义类加载器可回收。前两项的真正意义，在于解释加载器为什么仍然不可回收。

## 具备卸载资格后，JVM 何时真正卸载

类卸载是垃圾回收的一部分，而不是对象引用刚被清除时立即执行的同步操作。[HotSpot GC 调优指南](https://docs.oracle.com/en/java/javase/25/gctuning/other-considerations.html) 明确说明，Java 类随垃圾回收而卸载，对应的类元数据也在类卸载时释放。

实际时机取决于三个因素：

- 类卸载功能本身是否开启。HotSpot 的 `-Xnoclassgc` 会禁用类的垃圾回收。
- 当前垃圾收集器在哪种回收周期中处理类卸载。一次普通的年轻代回收，并不保证会检查类加载器是否死亡。
- JVM 的回收策略是否决定启动相应的回收周期。Metaspace 达到动态调整的高水位可能诱发 GC，但不可达加载器的出现本身并不要求 JVM 立刻回收。

顺带一提，`System.gc()` 只能请求 JVM 执行垃圾回收，不能当作生产代码中的卸载协议。HotSpot 还可以通过 `-XX:+DisableExplicitGC` 忽略这个请求。把"调用一次就立即卸载"当作预期，等于把实现策略误当成了程序语义。

## 为什么常见应用很少看到类卸载

Bootstrap ClassLoader 加载的类按规范不能卸载；Platform ClassLoader 和 App ClassLoader 在常规应用中通常也会存活到 JVM 结束，因此由它们定义的类很少取得卸载资格。后两者不是因为规范禁止，而是因为加载器长期可达。

类卸载更常见于主动建立了"短生命周期类加载器边界"的系统，例如：

- 应用服务器重新部署应用；
- 插件系统卸载插件；
- 脚本引擎替换脚本模块；
- 动态生成大量类并周期性废弃对应加载器。

反过来，仅仅持续生成新类而复用同一个长期存活的类加载器，不会形成可回收的边界。要让旧类真正退出生命周期，必须连同定义它们的加载器一起退出。

## 哪些引用会阻止类卸载

下面这些场景的共同结果，都是从长生命周期的 GC Root 或对象出发，建立了一条通往自定义类加载器的路径：

| 来源 | 典型引用链 |
| --- | --- |
| 存活线程 | 线程正在执行插件代码，或者线程的上下文类加载器指向插件加载器 |
| `ThreadLocal` | 长生命周期线程持有的 `ThreadLocalMap` 保存了插件类实例 |
| 父加载器中的静态缓存 | 系统类或框架类的全局集合保存子加载器定义的 `Class`、实例或加载器 |
| 注册机制 | 监听器、JDBC 驱动、MBean、日志组件或回调注册后未注销 |
| 后台任务 | 定时任务、线程池任务或异步回调仍保存插件对象 |
| 本地代码 | JNI 全局引用保存类、实例或类加载器 |

引用方向决定了是否形成泄漏。例如，插件类的静态字段指向一个系统对象，并不会单独让插件加载器变成可达；反方向由系统级缓存指向插件对象，才会形成从长生命周期对象到插件加载器的保留链。

## 用最小示例观察类卸载

下面的示例让 `URLClassLoader` 单独加载一个类。父加载器选择系统类加载器的父加载器，避免 App ClassLoader 从应用 classpath 提前加载 `demo.Plugin`。

```java
// Plugin.java
package demo;

public class Plugin {
    public String name() {
        return "temporary-plugin";
    }
}
```

```java
// ClassUnloadDemo.java
import java.lang.ref.WeakReference;
import java.net.URL;
import java.net.URLClassLoader;
import java.nio.file.Path;
import java.nio.file.Paths;

public class ClassUnloadDemo {
    public static void main(String[] args) throws Exception {
        Path pluginDirectory = Paths.get(args[0]);
        WeakReference<ClassLoader> loaderReference = load(pluginDirectory);

        for (int i = 0; i < 20 && loaderReference.get() != null; i++) {
            System.gc();
            Thread.sleep(100);
        }

        System.out.println("类加载器已回收：" + (loaderReference.get() == null));
    }

    private static WeakReference<ClassLoader> load(Path directory) throws Exception {
        URL[] urls = {directory.toUri().toURL()};
        URLClassLoader loader = new URLClassLoader(
                urls,
                ClassLoader.getSystemClassLoader().getParent()
        );

        try {
            Class<?> pluginClass = Class.forName("demo.Plugin", true, loader);
            Object plugin = pluginClass.getDeclaredConstructor().newInstance();
            System.out.println(pluginClass.getMethod("name").invoke(plugin));
            return new WeakReference<ClassLoader>(loader);
        } finally {
            loader.close();
        }
    }
}
```

在 JDK 9 及以上版本中编译并运行：

```bash
mkdir -p plugin-classes demo-classes
javac -d plugin-classes Plugin.java
javac -d demo-classes ClassUnloadDemo.java
java -Xlog:class+unload=info -cp demo-classes ClassUnloadDemo plugin-classes
```

几个要点：

- `URLClassLoader.close()` 只关闭它打开的文件或 JAR 资源，并不会主动卸载类。
- 真正让加载器具备回收条件的是：`load` 方法返回后，不再存在指向加载器、`Class` 对象和实例的强引用。
- 如果发生卸载，HotSpot 的统一日志会出现包含 `unloading class demo.Plugin` 的记录；`WeakReference` 变为 `null` 说明加载器已被回收，日志则直接证明类卸载事件确实发生。
- 由于 `System.gc()` 只是请求而非命令，单次运行没有观察到卸载，并不能反证卸载条件不成立。
- 官方 [`java` 命令文档](https://docs.oracle.com/en/java/javase/25/docs/specs/man/java.html) 给出的类卸载日志选项是 `-Xlog:class+unload=info`，需要更多细节时可改为 `trace`。

## 排查类不能卸载的方法

排查时要先确定待卸载类的定义类加载器，而不是只盯着类名。可以在代码中临时输出：

```java
System.out.println(targetClass.getClassLoader());
```

如果加载器本应退出却长期存在，使用堆转储工具查找从 GC Roots 到该加载器的引用路径。排查顺序可以按引用链的常见来源收敛：

1. 检查仍在运行的线程、线程上下文类加载器和线程栈。
2. 检查线程池中的 `ThreadLocal`、待执行任务和异步回调。
3. 检查由父加载器加载的框架类、单例和静态集合。
4. 检查监听器、驱动、MBean、定时任务等注册项是否已经注销。
5. 检查 JNI 全局引用或本地库持有的回调对象。
6. 确认没有使用 `-Xnoclassgc`，并通过 `-Xlog:class+unload=info` 观察实际卸载事件。

最后提醒一点：Metaspace 使用量没有立即下降，不能单独证明类没有被卸载。HotSpot 会复用已经释放的元数据内存块，已提交内存是否马上返还给操作系统，还受其内存管理策略影响。判断类卸载，应优先依据卸载日志、加载类统计和类加载器的可达性，而不是只盯着进程内存曲线。
