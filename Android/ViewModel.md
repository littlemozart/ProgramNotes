# Jetpack —— ViewModel 篇

`ViewModel` 是 Jetpack 的一员，简单地顾名思义，它就是 MVVM 模式中的 VM 层。当然它的作用远不止如此。先列几个 `ViewModel` 主要解决的痛点：

+ 让数据可在发生屏幕旋转等配置更改后继续留存

如果没有在 manifest.xml 文件配置 `android:configChanges="orientation"` 这个属性旋转屏幕时， Activity 会重建，这样会导致数据丢失。虽然可以通过 `onSaveInstanceState()` 方法从 `onCreate()` 中的 `savedInstanceState` 恢复其数据，但该方法仅适合可以序列化再反序列化的少量数据，而不适合数量可能较大的数据，如用户列表或位图。

+ 使数据在 `Activity` 与 `Fragment` 或 `Fragment` 与 `Fragment` 之间共享更简单



+ 承担 MVVM 中的 MV 层，让数据和试图解耦

诸如 Activity 和 Fragment 之类的界面控制器主要用于显示界面数据、对用户操作做出响应或处理操作系统通信（如权限请求）。如果要求界面控制器也负责从数据库或网络加载数据，那么会使类越发膨胀，违背单一职责原则。所以从界面控制器逻辑中分离出视图数据，实现解耦，也方便测试。

## 引用

下面使官方说明需要在 gradle 文件中加入相关依赖。

```
dependencies {
    // 最新版本号
    def lifecycle_version = "2.2.0"

    // ViewModel
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
}
```

## 示例

先自定义一个 `ViewModel`
```
class MyViewModel : ViewModel() {
    private val users: MutableLiveData<List<User>> by lazy {
        MutableLiveData().also {
            loadUsers()
        }
    }

    fun getUsers(): LiveData<List<User>> {
        return users
    }

    private fun loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```

在 Activity 中使用
```
class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // 使用 'ViewModelProvider' 创建
        val model = ViewModelProvider(this).get(MyViewModel::class.java)
        // 或使用 'by viewModels()' 创建，需要依赖 activity-ktx
        // val model: MyViewModel by viewModels()
        model.getUsers().observe(this, Observer<List<User>>{ users ->
            // 更新试视图
        })
    }
}
```

该 Activity 如果重建，它接收的 MyViewModel 实例与第一个 Activity 创建的实例相同。直到 Activity 销毁掉时，ViewModel 的 onCleared() 方法会被调用，以及时清理资源。

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

        // Use the 'by activityViewModels()' Kotlin property delegate
        // from the fragment-ktx artifact
        private val model: SharedViewModel by activityViewModels()

        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
            itemSelector.setOnClickListener { item ->
                // Update the UI
            }
        }
    }

    class DetailFragment : Fragment() {

        // Use the 'by activityViewModels()' Kotlin property delegate
        // from the fragment-ktx artifact
        private val model: SharedViewModel by activityViewModels()

        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
            model.selected.observe(viewLifecycleOwner, Observer<Item> { item ->
                // Update the UI
            })
        }
    }
```
