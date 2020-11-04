# Jetpack —— ViewModel 篇

## 前言

`ViewModel` 是 Jetpack 的核心组件之一。简单地顾名思义，它充当 MVVM 模式中的 VM 层。当然它的作用远不止如此。

本文主要按以下几点展开讨论

+ `ViewModel` 解决的痛点
+ `ViewModel` 依赖
+ `ViewModel` 示例
+ `ViewModel` 原理浅析

## `ViewModel` 主要解决的痛点：

**数据持久化**

如果没有在 manifest 文件中为 Activity 配置 `configChanges` , 则当发生配置改变如旋转屏幕或更换系统语言等操作，会导致 Activity 重建，所以视图绑定的数据就丢失了。这个问题普通的 ViewModel 就可以解决。

另外一种情况是当应用退到后台，由于资源紧张内存不足导致进程被杀，这时用户再回到该应用，Activity 也会重建。这个需要用到带 `SavedStateHandle` 的 ViewModel ，下文会有示例。

**Fragment 间共享数据**

通常 Fragment 之间共享数据需要借助共同的父 Activity ，以接口的方式调用。这样无疑会加大 Fragment 和 Activity 耦合度。 当然 `ViewModel` 用于 Fragment 之间数据共享也是基于共同的 Activity ，但是 `ViewModel` 框架已经为我们做了很多工作，我们可以不再需要写接口。

**数据和视图解耦**

## 引用

在 gradle 文件中加入相关依赖。

```
dependencies {
    // 最新版本号
    def lifecycle_version = "2.2.0"

    // ViewModel
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"

    // 可选 - Saved state module for ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:$lifecycle_version"

    // 推荐 - activity-ktx 可代替上面两个
    implementation 'androidx.activity:activity-ktx:1.1.0'
}
```

## 示例

先自定义一个普通的 `ViewModel` 。
```
class MyViewModel : ViewModel() {
    private val _count = MutableLiveData<Int>()

    val count = _count

    fun add() {
        count.value?.plus(1)
    }
}
```

再定义一个带 `SavedStateHandle` 的 `ViewModel` 。
```
class MySavedStateViewModel(private val savedStateHandle: SavedStateHandle) : ViewModel() {
    private val key = "key"

    val count = savedStateHandle.getLiveData<Int>(key)

    fun add() {
        savedStateHandle[key] = (count.value ?: 0) + 1
    }
}
```

在 Activity 中使用
```
class MyActivity : AppCompatActivity() {

    /**
     * 这里初始化用了 activity-ktx 的委托属性，原始的初始化需要使用 ViewModelProvider
     * 因为 AppCompatActivity 和 viewModels 默认的ViewModelProvider.Factory 都是
     * SavedStateViewModelFactory，所以创建 SavedStateViewModel 不用自己再传
     */
    private val viewModel by viewModels<MyViewModel>()
    private val savedStateViewModel by viewModels<MySavedStateViewModel>()
    private var count = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        viewModel.count.observe(this) {
            text1.text = "$it"
        }
        savedStateViewModel.count.observe(this) {
            text2.text = "$it"
        }
        fab.setOnClickListener {
            viewModel.add()
            savedStateViewModel.add()
            add()
        }
    }

    private fun add() {
        text0.text = "${++count}"
    }
}
```

上面示例的代码分别用三个 `TextView` 来显示计数，为了模拟应用在后台会杀死的情境，可以在开发者选项里面打开 “不保留活动” 这个开关。下面三个截图分别对应启动后点击四次增加按钮后、旋转屏幕后和退出应用再回来的情况。

![image](../assets/viewmodel.png)

在 Fragment 之间共享数据
```
class SharedViewModel : ViewModel() {
    val selected = MutableLiveData<Item>()

    fun select(item: Item) {
        selected.value = item
    }
}

class MasterFragment : Fragment() {

    private lateinit var itemSelector: Selector

    private val model: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        itemSelector.setOnClickListener { item ->
            // Update the UI
        }
    }
}

class DetailFragment : Fragment() {

    private val model: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        model.selected.observe(viewLifecycleOwner, Observer<Item> { item ->
            // Update the UI
        })
    }
}
```

另外在 Lifecycle 的 2.1.0 版本后加入了 `viewModelScope` 以更方便地支持在 `ViewModel` 中使用协程，且该协程作用域会在 `clear()` 中被取消，避免内存泄漏。示例如下：
```
class MyViewModel() : ViewModel() {

    fun initialize() {
        viewModelScope.launch {
            processBitmap()
        }
    }

    suspend fun processBitmap() = withContext(Dispatchers.Default) {
        // 在这里做耗时操作
    }

}
```

## 原理浅析

**`ViewModelProvider`**

从上面示例代码中可以看到 `ViewModel` 的创建均是由 `ViewModelProvider` 提供的，所以先从 `ViewModelProvider` 切入。
```
public class ViewModelProvider {
    ...

    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }
    
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
        this(owner.getViewModelStore(), factory);
    }

    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }

    ...
}
```
**`ViewModelStore`**
```
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
**`ViewModelStoreOwner`**
```
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```
**`ViewModel`**
**`ComponentActivity.NonConfigurationInstances`**
**`ComponentActivity::onRetainNonConfigurationInstance()`**