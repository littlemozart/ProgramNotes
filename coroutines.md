# Kotlin 异步编程之 —— Coroutines

其实，协程在编程语言中并不是什么新鲜概念。像 go, python 也有协程的概念，只不过 API 不尽相同。

## 为什么使用协程？

### 1. 轻量

> 协程就像非常轻量级的线程。线程是由系统调度的，线程切换或线程阻塞的开销都比较大。而协程依赖于线程，但是协程挂起时不需要阻塞线程，几乎是无代价的，协程是由开发者控制的。所以协程也像用户态的线程，非常轻量级，一个线程中可以创建任意个协程。

协程开发人员 Roman 是这样描述协程的。如何理解非常轻的意思呢？下面简单举个栗子

```
val c = AtomicLong()

for (i in 1..1_000_000L)
    thread(start = true) {
        c.addAndGet(i)
    }

println(c.get())
```
这里运行了一百万个线程并为每个都增加了一个共同的计数器。在我的机器上一运行风扇就狂转，可能一分钟都运行不完 😂

再对比一下协程:
```
val c = AtomicLong()

for (i in 1..1_000_000L)
    GlobalScope.launch {
        c.addAndGet(i)
    }

println(c.get())
```
这段代码在 1 秒左右可以跑完。

### 2. 代码风格更接近同步代码块

异步编程是程序员必定会遇到的问题，我们有很多途径来解决这个问题：

- 线程

线程应该目前最常见的避免应用程序阻塞的方法。但是线程有一些明显的缺陷：

—— 比较重，切换上下文开销大。

—— 可启动的线程数受底层操作系统的限制。

—— 一些平台中并不支持线程，如 JavaScript 。

—— Debug，避免竞争条件等麻烦。

- 回调

核心思想是将一个函数作为参数传递给另一个函数，并在处理完成后调用该函数。感觉是一个更优雅的解决方案，但又有几个问题：

—— 回调地狱

—— 错误处理复杂

- Futures, Promise

背后想法是当我们发起调用的时候，返回一个可被操作的对象。主要有以下几个缺点：

—— 与回调类似，编程模型从自上而下的命令式方法转变为具有链式调用的组合模型，需要对我们的编程方式进行一系列更改。

—— 返回类型远离不是我们直接需要的类型，而是返回一个必须被内省的新类型 `Promise` / `Futures` 。

—— 异常处理会很复杂。错误的传播和链接并不总是直截了当的。

- 响应式扩展（如Rx）

- 协程

## 协程实体 `Job` & `Deferred`
 
 A job has the following states:

| **State**                        | [isActive] | [isCompleted] | [isCancelled] |
| -------------------------------- | ---------- | ------------- | ------------- |
| _New_ (optional initial state)   | `false`    | `false`       | `false`       |
| _Active_ (default initial state) | `true`     | `false`       | `false`       |
| _Completing_ (transient state)   | `true`     | `false`       | `false`       |
| _Cancelling_ (transient state)   | `false`    | `false`       | `true`        |
| _Cancelled_ (final state)        | `false`    | `true`        | `true`        |
| _Completed_ (final state)        | `false`    | `true`        | `false`       |

```
                                      wait children
+-----+ start  +--------+ complete   +-------------+  finish  +-----------+
| New | -----> | Active | ---------> | Completing  | -------> | Completed |
+-----+        +--------+            +-------------+          +-----------+
                  |  cancel / fail       |
                  |     +----------------+
                  |     |
                  V     V
              +------------+                           finish  +-----------+
              | Cancelling | --------------------------------> | Cancelled |
              +------------+                                   +-----------+
```
 
## 挂起函数 `suspend fun`

### suspend fun

### coroutineScope

### withContext

## async

## LifeCycle中使用协程