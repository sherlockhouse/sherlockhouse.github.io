[TOC]

# 修改记录

| 版本 | 修改日期 | 作者 | 修改内容 |
|--------|--------|--------|--------|
| V1.0| 2022.01.12 | 顾林成 | 创建 |

# 相册开发文档
 链接 https://sherlockhouse.github.io/

##  一 . 目录结构

    app/
    ├── assets               # tensorflow lite 模型文件等
    ├── java
    │   └── core             #主要的抽象框架
    │   ├── customview       #自定义view
    │   ├── db               #自定义数据库
    │   ├── di               #全局依赖注入
    │   ├── domain           #领域
    │   ├── models           #MVVM - model
    │   ├── repository       #repository pattern 
    │   ├── ui               #MVVM - view & viewmodel
    │   └── utils            #工具类和扩展函数
    
    google/      # 原生平台可复用的代码
    matisse/     # 知乎开源的图片选择库

![ff](https://fernandocejas.com/assets/img/blog/android_architecture_reloaded_layers.png)






## 二 . 功能模块

* 界面导航
  
   >采用ViewPager2 + Recyclerview + Fragment 代替原生OpenGL实现
   
* 缩略图解码
  
  >使用谷歌的Glide
  
* 图片高清预览
  
  >subsampling-scale-image-view。只支持jpg和png
  
* 智能分类
  
  > tensorflowlite. 使用谷歌开源的预训练模型,模型文件在asset目录
* 编辑
  > 原生的编辑功能,目录 /app/src/main/java/com/android/gallery3d/filtershow/
  > 修改了知乎开源的matisse 图片选择模块



##  三 . 详细设计

### 1. 主题
在Android.manifest中,配置. 配置后,当前activity会应用系统样式.同时自定义样式失效，无法使用compat和material主题.

因此只用于非重要的activity。主activity控件主题效果完全自己控制。

```xml
   <meta-data
     android:name="com.freeme.app.theme"
     android:value="freeme:style/Theme.Freeme.NoActionBar.DayNight" />
```
### 2. 媒体数据结构

图库数据主要包括相册目录和媒体文件
```java
//单个照片
data class DetailMediaFile(
    var id: Long,
    var name: String?,
    var dateAdded: Long?,
    var dateTaken: Long?,
    var dateModified: Long?,
    val uri: Uri,
    var path: String?,
    var parentPath: String?,
    var size: Long?,
    var type: Int?,
    var videoDuration: Int?,
    var isFavorite: Boolean = false,
    var deletedTS: Long?,
    var bucketId: Long?,
)

//相片目录
data class DetailMediaSet(
    val name: String,
    val id: Int,
    val dateAdded: Long,
    val dateTaken: Long,
    val uri: Uri,
    var count: Int,
    var order: Int,
)
```
数据对象实现BaseListMediaFileItem接口，用于在recyclerview中配置不同的viewholder，以及使用DiffUtil

```java
open class BaseListMediaFileItem(val viewType: Int) {
    fun isSame(other: BaseListMediaFileItem): Boolean {
        if (this is MediaFileItem && other is MediaFileItem) {
            return this.detailMedia.uri == other.detailMedia.uri
        } else if (this is MediaFileHeaderItem && other is MediaFileHeaderItem) {
            return this.text == other.text
        }
        return false
    }

    var gridPosition: Int = 0

    enum class Type {
        HEADER,
        DATA
    }

    data class MediaFileHeaderItem(val text:String) : BaseListMediaFileItem(Type.HEADER.ordinal)
}

```
### 3. 全局数据管理对象

```
MediaDataManager为全局单例。因为相册对应的多个界面，包括fragment和activity，其核心数据都是mediastore中取出的媒体对象。因此需要一个全局管理mediadata的对象，避免反复查询mediastore，和互相传递。
```

```java
class MediaDataManager @Inject constructor() {

    // 媒体文件对象
    private val _mediaFilesLocal: MutableLiveData<List<MediaFileLocal>> =
        MutableLiveData(emptyList())
    val mediaFilesLocal: LiveData<List<MediaFileLocal>> get() = _mediaFilesLocal

    fun loadImage(medias : List<MediaFileLocal>) {
        _mediaFilesLocal.postValue(medias)
    }
    
	//图集对象
    val _HashMapMedias: MutableLiveData<WeakHashMap<String, ArrayList<BaseListMediaFileItem>>> = MutableLiveData(WeakHashMap())


}
```




### 4. 界面导航
主界面采用viewpager2.

相册主界面为复杂界面，而且数据量大，需要实时更新。所以采用viewpager2进行缓存。

采用了viewpager2后对navigation 导航组件就不太友好。因此主界面不再使用导航组件。

导航组件可用于其他非主要界面，比如设置等。

因为搜索也是复杂界面，需要频繁调用。因此也要作为一页page，来利用viewpager的缓存特性。但目前的UI没有搜索tab。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">


    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?android:actionBarSize" />


    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/main_view_pager"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />


    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_nav"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:menu="@menu/bottom_navigation_menu" />

</LinearLayout>

```



```java
 override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        setSupportActionBar(binding.toolbar)

        binding.mainViewPager.isUserInputEnabled = false
        binding.mainViewPager.offscreenPageLimit = 3


        val config = mutableListOf<Pair<Int, Supplier<out Fragment>>>()
        binding.bottomNav.menu.forEachIndexed { index, item ->
            config.add(when (item.itemId) {
                R.id.timeline -> R.string.tab_photos to Supplier {
                    TimeLineFragment()
                }
                R.id.search -> R.string.menu_search to Supplier {
                    SearchFragment()
                }
                R.id.mediaSet -> R.string.tab_albums to Supplier {
                    MediaSetFragment()
                }
                else -> R.string.tab_photos to Supplier {
                    TimeLineFragment()
                }
            })
        }
        binding.bottomNav.setupWithViewPager2(
            binding.mainViewPager,
            supportFragmentManager,
            lifecycle,
            config
        )
        binding.toolbar.setupWithViewPager2(binding.mainViewPager, config)
    }


```




### 5. 媒体单张预览
照片单张预览入口为MediaPagerActivity.java

通过 MediaPagerAdapter按照照片类型进入对应的预览fragment
```java

    override fun createFragment(position: Int): Fragment {
        when(mediaFiles[position].detailMedia.name) {
            "jpg/png" -> return ImageFragment.newInstance(position)
            "video" -> return VideoFragment.newInstance(position)
            "gif" -> return GifFragment.newInstance(position)
            else -> return FastImageFragment.newInstance(position)
        }
    }
```

* ImageFragment 采用imagesapling scale进行解码预览
* GifFragment 用于gif预览,采用glide解码
* VideoFragment 用于视频预览,目前相册未内置视频播放，跳转到外部视频播放
* FastImageFragment 除上述类型外的其他类型,以及未知类型的预览, 采用imagezoom进行预览



## 四 . 关于领域层 Domain Layer

![android architecture](https://fernandocejas.com/assets/img/blog/android_architecture_reloaded_mvvm_app.png)



https://fernandocejas.com/blog/engineering/2021-01-23-writing-first-class-features-bdd-gherkin/

