---
title: 为什么 Java 8 移除永久代并引入元空间
summary: 从规范边界、容量管理和类卸载机制解释 HotSpot 使用元空间替代永久代的原因及影响。
created: 2026-09-01
updated: 2026-09-02
tags: Java, JVM, 元空间, 永久代
cover: /img/jvm/why-java-8-replaced-permgen-with-metaspace-cover.webp
---

Java 8 在 HotSpot 虚拟机中移除了永久代（Permanent Generation，简称 PermGen），并使用元空间（Metaspace）承载类元数据。这个变化没有删除 JVM 规范中的方法区，而是替换了 HotSpot 对方法区相关数据的具体实现。它解决的核心问题是永久代容量难以预估和调节，同时降低了垃圾收集器与一块特殊堆区域之间的耦合。

## 方法区不等于永久代

《Java 虚拟机规范》将方法区定义为线程共享的运行时数据区，用于保存每个类的结构，例如运行时常量池、字段和方法信息，以及方法和构造器的代码。规范同时明确：方法区在物理内存中的位置、是否连续、是否执行垃圾收集，都由 JVM 实现自行决定。

因此，这三个概念不能混用：

- **方法区**是 JVM 规范定义的逻辑区域；
- **永久代**是 Java 8 之前 HotSpot 实现方法区相关存储的一种方式；
- **元空间**是 Java 8 起 HotSpot 存放类元数据的实现。

Java 8 移除的是永久代这项 HotSpot 实现，不是方法区本身。其他 JVM 也不必采用永久代或元空间。

## 永久代承担了什么

在早期 HotSpot 中，类的内部表示被分配在永久代，包括类结构、方法信息和运行时常量池等元数据。永久代由垃圾收集器管理，但与存放普通 Java 对象的年轻代、老年代分开计量，并通过以下参数单独控制：

```text
-XX:PermSize=<size>
-XX:MaxPermSize=<size>
```

永久代的内容并非在 Java 8 才一次性迁出。永久代移除工作在 JDK 7 已经开始，例如 JDK 7 将字符串常量池中的 `String` 对象移到了普通 Java 堆。到 JDK 8，剩余的类元数据转移到本地内存，永久代实现才被完整移除。

## 为什么要移除永久代

### 容量上限难以匹配实际类规模

永久代的最大容量由 `-XX:MaxPermSize` 限制。应用需要多少空间，主要取决于加载类的数量、类结构复杂度、动态生成类的数量以及类能否卸载。这些因素很难仅根据 Java 堆容量推算。

容量设置过小会产生 `java.lang.OutOfMemoryError: PermGen space`；设置过大又会预留应用未必需要的空间。框架大量使用动态代理、字节码生成，或者应用服务器反复部署应用时，这种估算尤其困难。OpenJDK 的 JEP 122 因而把“移除永久代容量调优需求”列为直接目标。

### 让永久代自身动态扩展并不简单

保留永久代并允许其无限扩展看似也能减少调参，但永久代不是一个孤立的字节数组。JEP 122 指出，永久代扩展还要求卡表、块偏移表等配套数据结构一起扩展；为了维持高效访问，它还需要保持近似连续的空间模型。这会增加实现复杂度，并可能留下不可用的地址空间片段。

将类元数据移到本地内存后，HotSpot 可以按需向操作系统申请空间，不再受一块预先规划的永久代边界约束。

### 类元数据的生命周期更适合按类加载器管理

类元数据的生命周期与定义它的类加载器相关。Java 8 的 HotSpot 从操作系统申请内存，再将其划分为块；每个块与特定类加载器关联，该加载器加载的类从这些块中分配元数据。类加载器及其类满足卸载条件后，相关内存块可以整体回收或复用。

这种组织方式直接表达了“类加载器拥有一组类元数据”的关系。永久代代码则广泛影响垃圾收集器、运行时和编译器。移除这块特殊区域减少了这些组件对永久代布局的依赖，也为后续垃圾收集器改进降低了约束。

### HotSpot 与 JRockit 的实现收敛

JEP 122 还明确记录了一个工程背景：移除永久代是 HotSpot 与 JRockit 收敛工作的一部分。JRockit 没有永久代，也不要求用户配置永久代容量。因此，这次变更不仅针对某一种异常，也是在统一 JVM 实现经验和运维模型。

## 元空间如何管理内存

元空间使用 Java 堆之外的本地内存保存类元数据。它按需申请和提交内存，并以类加载器为单位组织元数据块。这里的“本地内存”表示内存计入 JVM 进程占用，而不受 `-Xmx` 的 Java 堆上限直接约束。

当已提交的元空间达到一个高水位线时，HotSpot 可以触发垃圾收集，尝试卸载无用的类和类加载器。回收完成后，JVM 会根据释放空间的比例调整高水位线。`-XX:MetaspaceSize` 设置的是这个高水位线的初始值，不是启动时一次性分配的元空间大小。

在 64 位 HotSpot 开启压缩类指针时，类元数据还会使用压缩类空间（Compressed Class Space）。HotSpot 将它与其他类元数据所在区域分开管理，其容量由 `-XX:CompressedClassSpaceSize` 控制；`MaxMetaspaceSize` 则约束两部分已提交内存的总和。排查内存问题时，需要分别观察两部分数据。

## 永久代与元空间的差异

| 对比项 | 永久代 | 元空间 |
| --- | --- | --- |
| HotSpot 版本 | JDK 7 及以前 | JDK 8 起 |
| 主要内容 | 类元数据 | 类元数据 |
| 内存来源 | HotSpot 管理的永久代区域 | Java 堆之外的本地内存 |
| 默认容量边界 | 受永久代最大容量约束 | 默认不设置类元数据硬上限，仍受进程可用内存约束 |
| 主要上限参数 | `-XX:MaxPermSize` | `-XX:MaxMetaspaceSize` |
| 回收粒度 | 依赖永久代及对应 GC 实现 | 类卸载后回收或复用类加载器关联的内存块 |
| 典型溢出信息 | `PermGen space` | `Metaspace` 或 `Compressed class space` |

元空间的主要收益是弹性分配和更低的容量调优负担，不是减少每个类所需的元数据。JEP 122 甚至将“减少类元数据内存需求”列为非目标。因此，不能仅凭迁移到元空间推导出类元数据占用一定下降。

## 参数和监控方式的变化

升级到 Java 8 后，应从启动脚本中移除永久代参数，并按需要使用元空间参数：

```text
# 类元数据可使用的本地内存上限
-XX:MaxMetaspaceSize=256m

# 触发元数据相关 GC 的初始高水位线
-XX:MetaspaceSize=64m
```

这两组参数不能机械地一一替换。`MetaspaceSize` 控制初始 GC 高水位线，不等价于预分配容量；`MaxMetaspaceSize` 才是元空间上限。Oracle JDK 8 默认不限制类元数据可使用的本地内存，但操作系统、容器或进程地址空间仍会形成实际边界。生产环境是否设置显式上限，应结合进程总内存预算决定，而不是照搬旧的 `MaxPermSize` 数值。

可以先用 `jstat` 观察元空间、压缩类空间和类卸载趋势：

```bash
jstat -gc <pid> 1000
jstat -class <pid> 1000
```

`jstat -gc` 输出中的 `MC`、`MU` 分别表示元空间容量和已使用量，`CCSC`、`CCSU` 表示压缩类空间容量和已使用量；`jstat -class` 可以观察已加载和已卸载类数量。

如需分析 JVM 内部的本地内存，应在启动时开启 Native Memory Tracking，再使用 `jcmd` 查看 `Class` 等类别的保留量和提交量：

```text
-XX:NativeMemoryTracking=summary
```

```bash
jcmd <pid> VM.native_memory summary
```

Native Memory Tracking 会带来额外的性能和内存开销，不应在未评估成本时长期启用详细模式。

## 元空间仍然可能溢出

使用本地内存只取消了永久代的固定边界，并没有让类元数据变成无限资源。以下情况仍可能导致元空间持续增长：

- 动态代理、表达式引擎或字节码增强工具持续生成新类；
- 应用重复部署后，旧类加载器仍被线程、静态引用或缓存持有；
- 设置的 `MaxMetaspaceSize` 低于应用稳定运行所需容量；
- 进程或容器的可用本地内存耗尽；
- 压缩类空间达到 `CompressedClassSpaceSize` 上限。

当类元数据超过 `MaxMetaspaceSize` 时，HotSpot 通常抛出 `java.lang.OutOfMemoryError: Metaspace`。压缩类空间耗尽时则可能出现 `java.lang.OutOfMemoryError: Compressed class space`。如果没有设置元空间上限，持续增长还可能先耗尽进程本地内存。

排查时不应直接把上限调大。若 `MU` 和已加载类数量持续上升，而类卸载数量长期不变，应进一步检查类生成是否失控，或旧类加载器是否仍然可达。只有确认应用的稳定类规模本来就需要更多空间时，扩大上限才是容量调整，而不是掩盖类加载器泄漏。

## 结论

Java 8 移除永久代，本质上是 HotSpot 更换了类元数据的存储和容量管理方式。方法区仍是 JVM 规范的一部分；变化的是其实现从固定边界、需要单独调节的永久代，转为按类加载器组织、从本地内存按需申请的元空间。

这项设计减少了永久代容量估算和特殊区域维护成本，但没有消除内存上限、类卸载失败或类加载器泄漏。理解元空间时，需要同时关注类元数据的本地内存占用、类加载器生命周期以及进程的总体内存预算。

## 参考资料

- [JEP 122：Remove the Permanent Generation](https://openjdk.org/jeps/122)
- [Java 虚拟机规范：Method Area](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4)
- [Oracle JDK 8 GC 调优指南：Class Metadata](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/considerations.html)
- [Oracle JDK 7 虚拟机增强：字符串常量池迁移](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/enhancements-7.html)
- [Oracle JDK 8 故障排查指南：Native Memory Tracking](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html)
- [Oracle JDK 8 工具文档：jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)
