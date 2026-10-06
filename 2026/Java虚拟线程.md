# Java 虚拟线程

## 概述

JDK 21（LTS）正式引入了**虚拟线程（Virtual Threads）**，作为 `java.lang.Thread` 的一种
轻量实现。它让「一请求一线程」这种直观写法在高并发下重新变得划算，而不是一上来就上
异步框架或响应式编程。

核心区别：

| 维度 | 平台线程 Platform Thread | 虚拟线程 Virtual Thread |
|------|--------------------------|--------------------------|
| 载体 | 1:1 映射 OS 线程 | 由 JVM 调度，可多个复用同一载体线程 |
| 开销 | 约 1MB 栈 + 系统资源 | 极小，可轻松创建上百万个 |
| 创建成本 | 高 | 微秒级、极廉价 |
| 阻塞 | 真阻塞 OS 线程 | 阻塞时自动让出载体线程 |
| 池化 | 需要 | **不需要**，用完即弃 |

## 创建虚拟线程

```java
// 方式一：静态工厂
Thread vThread = Thread.ofVirtual().start(() -> {
    System.out.println("hello virtual thread");
});

// 方式二：用构建器命名 / 设置未捕获异常处理器
Thread t = Thread.ofVirtual()
        .name("worker-", 0)
        .uncaughtExceptionHandler((th, ex) -> log.error("vthread error", ex))
        .start(() -> doWork());
```

## 配合执行器

虚拟线程不应该丢进传统线程池（那会抹平它的优势）。用 JDK 提供的新执行器：

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> fetchUrl(i))
    );
} // try-with-resources 会自动等待所有任务结束
```

`newVirtualThreadPerTaskExecutor` 每提交一个任务就新建一个虚拟线程，任务结束即回收，
**天然无需池化**。

## 一个对比示例

用虚拟线程改写阻塞式 HTTP 调用，代码结构不变，吞吐却大幅提升：

```java
// 传统写法（平台线程池，线程数有限，阻塞等待浪费资源）
try (var pool = Executors.newFixedThreadPool(200)) {
    urls.forEach(u -> pool.submit(() -> download(u)));
}

// 虚拟线程写法（去掉池，逻辑照旧）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    urls.forEach(u -> executor.submit(() -> download(u)));
}
```

阻塞点（数据库连接、HTTP、文件 IO）会让虚拟线程「挂起」，JVM 把载体线程让给别的
虚拟线程用，所以整体吞吐随并发数几乎线性上升。

## 适用与不适用

**适合**：IO 密集型、请求处理、任务数量大且生命周期短、想简化异步代码的场景。

**不适合**：

- 纯 CPU 密集计算：虚拟线程不会加速计算本身，反而有调度开销
- 需要固定线程身份 / 线程局部状态强依赖的场景
- 大量 `synchronized` 块内的长阻塞：早期版本存在 **pinning（钉住）** 问题

## 注意事项

1. **不要池化虚拟线程**，也别往 `ThreadPoolExecutor` 里塞虚拟线程。
2. 避免把昂贵的对象放进 `ThreadLocal` 长期持有，虚拟线程多、生命周期短，容易造成堆积。
3. `synchronized` 里避免长时间阻塞 IO；优先用 `ReentrantLock` 或 IO 友好结构。
4. 用 `-Djdk.tracePinnedThreads=full` 可诊断被钉住的虚拟线程。

## 小结

虚拟线程不是「更快的线程」，而是「更便宜、能大规模创建、阻塞不心疼」的线程。
大多数 `new Thread(...)` / 线程池的阻塞式代码，替换成虚拟线程就能在结构不变的情况下
显著提升 IO 并发能力。
