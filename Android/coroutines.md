# Kotlin 异步编程之 —— Coroutines

其实，协程在编程语言中并不是什么新鲜概念。
像 go, python 也有协程的概念，只不过 API 不尽相同。

## 为什么使用协程？

### 1. 轻量

> 协程就像非常轻量级的线程。线程是由系统调度的，线程切换或线程阻塞的开销都比较大。
> 而协程依赖于线程，但是协程挂起时不需要阻塞线程，几乎是无代价的，协程是由开发者控制的。
> 所以协程也像用户态的线程，非常轻量级，一个线程中可以创建任意个协程。

协程开发人员 Roman 是这样描述协程的。如何理解非常轻的意思呢？下面用两段代码做个对比。

```
val c = AtomicLong()

for (i in 1..1_000_000L)
    thread(start = true) {
        c.addAndGet(i)
    }

println(c.get())
```
这里运行了一百万个线程并为每个都增加了一个共同的计数器。
在我的机器上一运行风扇就狂转，一分钟都运行不完，甚至有可能会因为内存不足而出现异常。

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

相较之下，线程有以下缺点：

+ 比较重，切换上下文开销大。

+ 可启动的线程数受底层操作系统的限制。

+ 一些平台中并不支持线程，如 JavaScript 。

+ Debug，避免竞争条件等麻烦。

### 2. 以同步代码块风格编写异步程序

众所周知，如何避免进程阻塞是程序员都会遇到的问题，这种情况下就需要考虑异步编程。下面简单对比一些主流异步设计：

- 回调

    核心思想是将一个函数作为参数传递给另一个函数，并在处理完成后调用该函数。
    感觉是一个更优雅的解决方案，但又有几个问题：

    [1] 回调地狱

    [2] 错误处理复杂

- Futures, Promise

    背后想法是当我们发起调用的时候，返回一个可被操作的对象。主要有以下几个缺点：

    [1] 与回调类似，编程模型从自上而下的命令式方法转变为具有链式调用的组合模型，需要对我们的编程方式进行一系列更改。

    [2] 返回类型不是我们直接需要的类型，而是一个必须被内省的新类型 `Promise` / `Futures` 。

    [3] 异常处理会很复杂。错误的传播和链接并不总是直截了当的。

- 响应式扩展（如Rx）

    响应式 (Rx) 被移植到各种平台，包括 JavaScript（RxJS）。
    Rx 背后的想法是“可观察流”，我们现在将数据视为流（无限量的数据），并且可以观察到这些流。
    另外它提供了大量的操作符，允许我们对数据进行操作。

    在方法上它与 Futures 非常相似，但是 Futures 可被视为一个离散元素，而 Rx 则返回的是一个流。
    与前面类似，它也引入了一种全新的编程模型的方式，著名的表述是：

    > 一切都是流，并且它是可被观察的

    这意味着处理问题的方式不同，并且在编写同步代码时从我们使用的方式发生了相当大的转变。
    但与 `Futures` 相比的一个好处是，它被移植到很多平台，通常我们可以找到一致的 API 体验，无论我们使用 C＃、Java、JavaScript、Swift，还是 Rx 可用的任何其他语言。

    此外，Rx 确实引入了一种更好的错误处理方法。

- 协程

    背后的想法是一种可以被挂起的函数，即可以在某个时刻暂停执行并在稍后恢复的思想。

    协程的一个好处是，当涉及到开发人员时，编写非阻塞代码与编写阻塞代码基本相同。
    编程模型本身并没有真正改变。以下面的代码为例
    
    ```
    fun postItem(item: Item) {
        launch {
            val token = preparePost()
            val result = doPost(token, item)
            showResult(result)
        }
    }
    
    suspend fun preparePost(): Token {
        // 发起请求并挂起该协程
        return suspendCoroutine { /* ... */ } 
    }
    
    suspend fun doPost(token: Token, item: Item): Result {
        // 发起post请求
        return result
    }
    ```
   
    `preparePost` 和 `doPost` 就是所谓的 `可挂起的函数`，因为它们含有 `suspend` 前缀。
    此段代码将启动长时间运行的操作，而不会阻塞主线程。

## 引入协程

首先在 Project 的 build.gradle 中添加 kotlin 插件

```
buildscript {
  // kotlin版本必须在1.3以上，写文时最新为1.3.72
  ext.kotlin_version = '1.3.72'
  dependencies {
    classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version" // Kotlin Gradle Plugin
  }
}
```
然后在 app 的 build.gradle 中添加一下依赖
```
dependencies {
  def coroutines_version = 1.3.6
  implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutines_version"
  implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutines_version"
  implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
}
```

## 协程简析

在我们使用 `协程` 之前，有必要先了解一下这几个概念：

- `CoroutineContext`

    是一个接口，所有可以被称为协程上下文的元素都实现了这个接口，这些元素包括了 `Dispatchers` 以及 `Job` 等等。

- `CoroutineScope`

    提供 `CoroutineContext` 的接口，翻译为协程的作用域。协程都是跑在 `CoroutineScope` 里面的。

- `CoroutineBuilders`

    并不是一个类，它是 `CoroutineScope` 的一些扩展函数，用于定义和启动协程，用的最多的是 `launch` 和 `async` ，详见下文。
- `CoroutineDispatchers`

    `CoroutineContext` 包含一个 `CoroutineDispatcher` ，它确定了哪些线程或与线程相对应的协程执行。协程调度器可以将协程限制在一个特定的线程执行，或将它分派到一个线程池。
    协程调度器有以下四种：
    
    [1] `Dispatchers.Default`
    
    默认调度器，使用共享的后台线程池。 当协程在 `GlobalScope` 中启动时，使用的就是该调度器。所以 `launch(Dispatchers.Default) { ... }` 与 `GlobalScope.launch { ... }` 使用相同的调度器。
    
    [2] `Dispatchers.Main`
    
    主线程调度器，指定在主线程 （android 即 UI 线程）中运行。
    
    [3] `Dispatchers.IO`
    
    I/O调度器，将任务指定在 I/O 线程中执行。和 `Dispatchers.Default` 共享后台线程池。
    
    [4] `Dispatchers.Unconfined`

    是一个特殊的调度器，调度器在调用它的线程启动了一个协程，但它仅仅只是运行到第一个挂起点。挂起后，它恢复线程中的协程，而这完全由被调用的挂起函数来决定。非受限的调度器非常适用于执行不消耗 CPU 时间的任务，以及不更新局限于特定线程的任何共享数据（如UI）的协程。
    
- `Job` & `Deferred`
 
    协程作用域里的任务。 `Deferred` 继承 `Job`，它们的区别是 `Job` 完成时是没有返回值的，而 `Deferred` 可指定返回类型。
    它们有以下状态：

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

## `launch`

### 第一个协程程序

```
fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(500) // 非阻塞的等待 500 毫秒
        println("world")
    }
    println("hello") // 协程已在等待时主线程还在继续
    Thread.sleep(1000) // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```

这里我们在 `GlobalScope` 中启动了一个新的协程，这意味着新协程的生命周期只受整个应用程序的生命周期限制。

### runBlocking

```
fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(500)
        println("world")
    }
    println("hello")
    runBlocking {     // 这个表达式阻塞了主线程
        delay(1000)  // 延迟 1 秒来保证 JVM 的存活
    } 
}
```

调用了 `runBlocking` 的主线程会一直阻塞直到 `runBlocking` 内部的协程执行完毕。
它也作为用来启动顶层主协程的适配器

```
fun main() = runBlocking {
    GlobalScope.launch {
        delay(500)
        println("world")
    }
    println("hello")
    delay(1000)
}
```

### 等待协程执行完

延迟一段时间来等待另一个协程运行并不是一个好的选择。让我们显式（以非阻塞方式）等待所启动的后台 `Job` 执行结束：

```
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(500)
        println("world")
    }
    println("hello")
    job.join()
}
```

### 结构化的并发

上面提到 `join` 来等待协程结束，需要用一个变量来持有协程的引用。
除此之外，有一个更好的解决办法。我们可以在代码中使用结构化并发。 
我们可以在执行操作所在的指定作用域内启动协程， 而不是像通常使用线程（线程总是全局的）那样在 `GlobalScope` 中启动。

```
fun main() = runBlocking {
    launch { // 在 runBlocking 作用域中启动一个新协程
        delay(500)
        println("world")
    }
    println("hello")
}
```

外部协程（示例中的 `runBlocking` ）直到在其作用域中启动的所有协程都执行完毕后才会结束。

## `async`

### 顺序调用

假设我们在不同的地方定义了两个进行某种调用远程服务或者进行计算的挂起函数：

```
suspend fun getNumber1(): Int {
    delay(1000) // 模拟耗时操作
    return 1
}

suspend fun getNumber2(): Int {
    delay(1000) // 模拟耗时操作
    return 2 
}
```

下面用一个函数测试顺序调用花费时间：

```
fun main() = runBlocking {
    val time = measureTimeMillis {
        val num1 = getNumber1()
        val num2 = getNumber2()
        println("The answer is ${num1 + num2}")
    }
    println("Completed in $time ms")
}
```

结果是显而易见的。实际上，如果我们要根据第一个函数的结果来决定是否我们需要调用第二个函数或者决定如何调用它时，才会这样做。

### 并发调用

如果 `getNumber1` 与 `getNumber2` 之间没有依赖，并且我们想更快的得到结果，这时 `async` 就派上用场了。
在概念上，`async` 就类似于 `launch` 。它启动了一个单独的协程，不同之处在于 `launch` 返回一个 `Job` 并且不附带任何结果值，而 `async` 返回一个 `Deferred` —— 一个轻量级的非阻塞 `Future`，
你可以使用 `.await()` 在一个延期的值上得到它的最终结果。 因为 `Deferred` 继承 `Job` ，所以同样可以用 `.cancel()` 取消它。

```
fun main() = runBlocking {
    val time = measureTimeMillis {
        val def1 = async { getNumber1() }
        val def2 = async { getNumber2() }
        println("The answer is ${def1.await() + def2.await()}")
    }
    println("Completed in $time ms")
}
```

运行时间大概是上面的一半，因为两个协程并发执行。

### 懒性启动

`async` 可以通过将 `start` 参数设置为 `CoroutineStart.LAZY` 而变为惰性的。 在这个模式下，只有结果通过 `await` 获取的时候协程才会启动，或者在 `Job` 的 `start` 函数调用的时候。运行下面的示例：

```
fun main() = runBlocking {
    val time = measureTimeMillis {
        val def1 = async(start = CoroutineStart.LAZY) { getNumber1() }
        val def2 = async(start = CoroutineStart.LAZY) { getNumber2() }
        // 执行一些计算
        def1.start() // 启动第一个
        def2.start() // 启动第二个
        println("The answer is ${def1.await() + def2.await()}")
    }
    println("Completed in $time ms")
}
```

注意，如果我们只是在 `println` 中调用 `await`，而没有在单独的协程中调用 `start` ，这将会导致顺序行为，直到 `await` 启动该协程执行并等待至它结束，这并不是惰性的预期用例。

### `coroutineScope` 函数

由于 `async` 被定义为了 `CoroutineScope` 上的扩展，我们需要将它写在作用域内，而 `coroutineScope` 函数提供了这种操作：

```
suspend fun concurrentSum(): Int = coroutineScope {
    val def1 = async { getNumber1() }
    val def2 = async { getNumber2() }
    def1.await() + def2.await()
}
```

这是一种结构化并发的写法，通过 `coroutineScope` 封装成一个挂起函数。

###  `withContext` 函数

与 `coroutineScope` 类似可用来创建挂起函数，不同的是，`withContext` 需要指定协程调度器。

```
suspend fun concurrentSum2(): Int = withContext(Dispatchers.Default) {
    val def1 = async { getNumber1() }
    val def2 = async { getNumber2() }
    def1.await() + def2.await()
}
```

当我们需要发起网络请求时，可以使用 `withContext(Dispatchers.IO)` 来执行它。

## 并发问题

首先看一个例子

```
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 启动的协程数量
    val k = 1000 // 每个协程重复执行同一动作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程的作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}

fun main() = runBlocking {
    var counter = 0
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("counter = $counter")
}
```

这段代码最后打印出什么结果？

它不太可能打印出 `Counter = 100000` ，因为一百个协程在多个线程中同时递增计数器但没有做并发处理。

### Volatile 可以解决问题吗？

```
@Volatile
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

用 `Volatile` 修饰后运行速度似乎变更慢了，而且仍然没有得到 `Counter = 100000` 这个结果，可见 `Volatile` 也无济于事。因为 `Volatile` 虽然保证了可见性，但是无法保证原子性。

### 采用线程安全的数据结构

```
fun main() = runBlocking {
    val counter = AtomicInteger()
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
```

这次我们得到了 `Counter = 100000` 的正确结果了。

线程安全的数据结果适用于普通计数器、集合、队列和其他标准数据结构以及它们的基本操作。然而，它并不容易被扩展来应对复杂状态、或一些没有现成的线程安全实现的复杂操作。

### `Mutex`

在使用线程时，我们通常会用 `synchronized` 或一些锁来保证代码块永远不会同时执行，以此保护共享状态的所有修改。在协程中的替代品叫做 `Mutex` 。它具有 lock 和 unlock 方法， 可以隔离关键的部分。关键的区别在于 `Mutex.lock()` 是一个挂起函数，它不会阻塞线程。

在实际开发中，通常会用其扩展函数 `withLock` 来简化 `lock()` 和 `unlock()` 的操作。

```
fun main() = runBlocking {
    val mutex = Mutex()
    var counter = 0
    withContext(Dispatchers.Default) {
        massiveRun {
            // 用锁保护每次自增
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

## 协程在 LifeCycle / ViewModel 中的使用

在 Android 中，使用协程需要指定其 `CoroutineScope` 。 JetPack 中可以方便的使用 LifeCycle 组件的扩展属性 `LifecycleOwner.lifecycleScope` 和
ViewModel 组件的扩展属性 `ViewModel.viewModelScope` 来执行协程。

首先，需要在 gradle 文件添加以下扩展依赖：
```
dependencies {
    // for LifeCycleScope
    impletementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.2.0'
    // for ViewModelScope
    impletementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0'
}
```

然后可在 LifeCycle/ViewModel 组件中使用。

```
class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.lifecycleScope.launch {
            
        }
    }
}

class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            // Coroutine that will be canceled when the ViewModel is cleared.
        }
    }
}
```

因为这些扩展属性实现时都在销毁时调用了 `cancel`，所以我们不必担心内存泄漏问题。

## 协程在 Retrofit 中的使用

Retrofit 在 2.6.0+ 的版本开始支持 `suspend fun` 的形式，并且由于内部已经异步处理，所以不用我们指定 `Dispatchers.IO` 来执行。
简单的示例如下：

```
// 数据类
data class DataResponse(
    ...
)

// 数据接口类
interface DataApi {
    @GET("api/get")
    suspend fun getData(): DataResponse
}

// 在 ViewModel 中使用
class DataViewModel : ViewModel() {
    private val api by lazy {
        Retrofit.Builder().baseUrl("xxx")
            .addConverterFactory(MoshiConverterFactory.create()) // 添加JSON解析器
            .build()
            .create(DataResponse::class.java)
    }
    
    private val dataModel by lazy {
        MutableLiveData<DataResponse>()
    }

    init {
        viewModelScope.launch {
            val data = api.getData()
            dataModel.postValue(data)
        }
    }
}
```
