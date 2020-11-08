# Jetpack —— ViewModel 篇

## 前言

`ViewModel` 是 Jetpack 的核心组件之一。简单地顾名思义，它充当 MVVM 模式中的 VM 层。当然它的作用远不止如此。

本文主要按以下几点展开讨论

+ `ViewModel` 解决痛点
+ `ViewModel` 依赖引用
+ `ViewModel` 用法示例
+ `ViewModel` 源码浅析

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

    // 推荐 - activity-ktx 和 fragment-ktx 可代替上面两个
    implementation 'androidx.activity:activity-ktx:1.1.0'
    implementation 'androidx.fragment:fragment-ktx:1.2.5'
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
     * 因为 AppCompatActivity 和 viewModels 默认的 ViewModelProvider.Factory 都是
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

在 Lifecycle 的 2.1.0 版本后加入了 `viewModelScope` 以更方便地支持在 `ViewModel` 中使用协程；另外 `ViewModel` 还可以结合 `databinding` 和 `` 使用，具体可以参考，本文不再赘述。

## 源码浅析

在分析源码之前，我们先思考这几个问题

1. `ViewModel` 是怎么在 `Fragment` 之间共享数据的？
2. `ViewModel` 如何保存和恢复数据的？
3. `ViewModel` 在配置改变和进程被杀死这两种情况下的恢复机制有什么区别？

带着这些问题我们逐步深入源码。首先从 `ViewModel` 的创建入开始。`ViewModel` 的创建均是由 `ViewModelProvider` 提供的，所以先看下 `ViewModelProvider` 的构造函数。
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
从构造函数中可以看出需要传入 `ViewModelStore` / `ViewModelStoreOwner` 和 `Factory` 。其中 `Factory` 是一个接口，如下
```
public interface Factory {
    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```
这里我们先了解 `ViewModel` 实例是由 `Factory` 的实现类创建的，在源码中有 `AndroidViewModelFactory` 和 `SavedStateModelFactory` 这些实现类，基本都是通过反射来生成，具体细节暂不深入。

重点关注第一个参数 `ViewModelStore` / `ViewModelStoreOwner` ，其中 `ViewModelStoreOwner` 也是个接口，如下
```
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```
它提供了 `ViewModelStore` 的获取方法，也即间接提供了 `ViewModelStore` 。`ComponentActivity` 和 `Fragment` 就实现了 `ViewModelStoreOwner` 这个接口：
```
// ComponentActivity 是 FragmentActivity 的基类，而 FragmentActivity 是常用的 AppCompatActivity 的基类
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
    ...

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }

    ...
}

// Fragment
public class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner,
        ViewModelStoreOwner, SavedStateRegistryOwner {
    ...

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (mFragmentManager == null) {
            throw new IllegalStateException("Can't access ViewModels from detached fragment");
        }
        return mFragmentManager.getViewModelStore(this);
    }

    ...
}
```
那么 `ViewModelStore` 是个什么呢？为什创建 `ViewModel` 需要它呢？不妨进一步看下它的源码
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
它维护的是一个 `HashMap<String, ViewModel>` ，可以推断出我们 Activity 和 Fragment 创建和使用的 `ViewModel` 是存在这个 map 里面的。

既然是 `HashMap` 结构，那么在存取的时候就需要一个 key ，这个 key 是在哪里传进去的呢？让我们把焦点转移到 `ViewModelProvider` 获取 `ViewModel` 的 get 方法
```
public class ViewModelProvider {

    private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";
    ...

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }

    ...
```
可知 key 正是在 `get` 方法传进去的，当调用没有 key 传参的方法时，系统会自动帮你生成一个默认 key ，规则是 `DEFAULT_KEY + ':' + ViewModel 类名` 。

我们通常使用的是不传 key 的方法，这里有一个地方需要注意的地方。因为上文提到 `ViewModelStore` 内部是一个 `HashMap` ，所以当我们使用默认 key 生成同类型的 `ViewModel` 时候，获取到的会是同一个 `ViewModel` 。

举个例子，比如你在一个 Activity 中声明了两个类型一样的 `ViewModel` 并通过 `ViewModelProvider(this).get(xxx.class)` 的方式初始化，你会发现两个 `ViewModel` 的地址是一样的。由此，我们可以推断出 `ViewModel` 可以在 Fragment 之间共享数据，前提是他们归属同一个 Activity 的原因。

当我们需要在 Fragment 之间共享 `ViewModel` 时，在初始化会使用它们的 Activity 作为入参来创建 `ViewModelProvider(requireActivity()).get(xxx.class)` ， 这样得到的是就同一个 `ViewModel` 。至此，问题一得到解决。

**`ComponentActivity.NonConfigurationInstances`**

**`ComponentActivity::onRetainNonConfigurationInstance()`**