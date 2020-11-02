# Jetpack —— ViewModel 篇

`ViewModel` 是 Jetpack 的核心组件之一。简单地顾名思义，可以把它当成 MVVM 模式中的 VM 层。当然它的作用远不止如此。先列几个 `ViewModel` 主要解决的痛点：

+ 数据持久化

+ Fragment 之间共享数据

+ 数据和视图解耦

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
