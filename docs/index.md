[TOC]

# 修改记录

| 版本 | 修改日期 | 作者 | 修改内容 |
|--------|--------|--------|--------|
| V1.0| 2020.10.31 | 顾林成 | 创建 |

# 相册开发文档
 链接 https://sherlockhouse.github.io/

##  一 . 目录结构

    app/
    ├── assets                # tensorflow lite 模型文件等
    ├── java
    │   └── com
    │       ├── android       # 谷歌原生的相册代码
    │       ├── arch          #相册通用的框架代码
    │       ├── cmcc          # 由于需要修改包名,部分代码必须放到包名同名目录
    │       └── mediatek      # MTK 原生相册代码目录
    └── rs                    #MTK 原生的图片编辑库

    base/        # 项目通用的基础框架
    imageZoom/   # github开源的图片查看放大库
    matisse/     # 知乎开源的图片选择库
    mtkgallery2/ # mtk平台原生的资源文件




    # network 网络相关的库,项目中没有用到,保留
    Network/
    NetworkLite/




## 二 . 功能模块

* 界面导航
   >采用谷歌导航组件 + ViewPager2 https://developer.android.com/guide/navigation
   >来处理界面间的跳转.数据的异步加载结合RxJava

* 缩略图解码

  >使用谷歌的Glide

* 图片高清预览
  >开源项目   subsampling-scale-image-view
  >https://github.com/davemorrissey/subsampling-scale-image-view

* 智能分类
  > tensorflowlite. 使用谷歌开源的预训练模型,模型文件在asset目录,项目相关目录
  > app/src/main/java/com/android/gallery3d/discover
* 编辑
  > 原生的编辑功能,目录 /app/src/main/java/com/android/gallery3d/filtershow/
  > 修改了知乎开源的matisse 图片选择模块



##  三 . 详细设计

### 1. 主题
在Android.manifest中,配置. 配置后,当前activity会应用系统样式.同时自定义样式失效.

```xml
            <meta-data
                android:name="com.cmcc.app.theme"
                android:value="cmcc:style/Theme.Cmcc.Light" />
```
### 2. 媒体数据结构

图库数据主要包括相册目录和照片
```java
//单张照片
public class MediaItem implements TimelineItem, CursorHandler, Parcelable {
    private String path = null;
    private String title = null;
    private long dateModified = -1;
    private String mimeType = MimeTypeUtils.UNKNOWN_MIME_TYPE;
    private int orientation = 0;
    private int id;
    private int bucketid;
    public double latitude = INVALID_LATLNG;
    public double longitude = INVALID_LATLNG;
    public int width;
    public int height;
    public int mFileFlag;
    public long mJpegSize = 0;

    private String uriString = null;
    private String photoPreviewUri = null;
}

//相片目录
public class Album implements CursorHandler, Parcelable {

    public static final long ALL_MEDIA_ALBUM_ID = 8000;
    private String name, path;
    private long id = -1, dateModified;
    private int count = -1;
    private String onlinePath;

    private boolean selected = false;
    public AlbumSettings settings = null;
    private MediaItem lastMediaItem = null;
}
```
数据对象继承 CursorHandler,由CPHelper类从系统媒体库中获取照片数据信息.

```java
public class CPHelper {

    private static Observable<Album> getAlbums(Context context, ArrayList<String> excludedAlbums, SortingMode sortingMode, SortingOrder sortingOrder) {
        //.....
    }

    public static Observable<MediaItem> getMedia(Context context, Album album) {

        if (album.getId() == -1) {
            return getMediaFromStorage(context, album);
        } else if (album.getId() == Album.ALL_MEDIA_ALBUM_ID) {
            return getAllMediaFromMediaStore(context, album.settings.getSortingMode(), album.settings.getSortingOrder());
        } else {
            return getMediaFromMediaStore(context, album, album.settings.getSortingMode(), album.settings.getSortingOrder());
        }
    }

}

```
### 3. 数据库

智能分类功能,"discover.db", 用于存储分类信息.

```java
public class DiscoverHelper extends SQLiteOpenHelper {
    private static final String DB_NAME = "discover.db";
    private static final int DB_VERSION = 1;

    @Override
    public void onCreate(SQLiteDatabase db) {
        String thing = "CREATE TABLE IF NOT EXISTS " + DiscoverStore.Things.Media.TABLE_NAME
                + "("
                + DiscoverStore.Things.Columns._ID + " INTEGER PRIMARY KEY AUTOINCREMENT,"
                + DiscoverStore.Things.Columns.IMAGE_ID + " INTEGER,"
                + DiscoverStore.Things.Columns.CLASSIFICATION + " INTEGER DEFAULT -2"
                + ")";
     }
}
```

表thing 用于记录智能分类后的相册数据,

```java
DiscoverStore.Things.Columns._ID  : 唯一ID
DiscoverStore.Things.Columns.IMAGE_ID : 对应系统媒体数据库中的 ID,用于查找对应图片
DiscoverStore.Things.Columns.CLASSIFICATION : 分类ID,由于归类
```


### 4. 界面导航
采用谷歌导航组件, 以nav_graph_albumset.xml为例

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/nav_graph_albumset"
    app:startDestination="@id/AlbumSetFragment">

    <fragment
        android:id="@+id/AlbumSetFragment"
        android:name="com.arch.gallery.fragments.albums.AlbumSetFragment"
        android:label="@string/tab_photos"
        tools:layout="@layout/fragment_albumset">

        <action
            android:id="@+id/action_AlbumSetFragment_to_AlbumFragment"
            app:destination="@id/AlbumFragment" />

    </fragment>

    <fragment
        android:id="@+id/AlbumFragment"
        android:name="com.arch.gallery.fragments.albums.AlbumFragment"
        android:label="Album"
        tools:layout="@layout/fragment_album">

        <action
            android:id="@+id/action_AlbumFragment_to_AlbumSetFragment"
            app:destination="@id/AlbumSetFragment" />
        <argument
            android:name="album"
            app:argType="com.arch.gallerybase.data.Album" />
    </fragment>

</navigation>

```

上诉xml文件中定义了导航关系,通过action 指定导航的目的地,这个可以直接从android studio里可视化的设置.

https://developer.android.com/guide/navigation/navigation-navigate

```java
 // 在AlbumsetFragment中调用,就可以跳转到AlbumFragment
        NavHostFragment.findNavController(this)
                .navigate(AlbumSetFragmentDirections.Companion
                        .actionAlbumSetFragmentToAlbumFragment(adapter.get(position)));
```

https://developer.android.com/guide/navigation/navigation-custom-back#java
返回参考上述链接,按如下代码设置,不然会被activity拦截back按键
```java
//从AlbumFragment返回
 @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        setHasOptionsMenu(true);
        callback = new OnBackPressedCallback(true /* enabled by default */) {
            @Override
            public void handleOnBackPressed() {
                // Handle the back button event
                if (!isAdded()) {
                    return;
                }
                findNavController(requireActivity(), R.id.nav_host_fragment_albumset).popBackStack();
            }
        };
        requireActivity().getOnBackPressedDispatcher().addCallback(this, callback);
    }
```


### 5. 照片单张预览
照片单张预览入口为SingleMediaActivity.java

通过 MediaPagerAdapter 按照照片类型进入对应的预览fragment
```java
    private MediaPagerAdapter adapter;

    @Override
    public Fragment createFragment(int position) {
        MediaItem mediaItem = this.mediaItems.get(position);
        if (mediaItem.getMimeType() == null) {
            return FastImageFragment.newInstance(mediaItem);
        }

        if (mediaItem.getMimeType().equals(MediaItem.MIME_TYPE_JPEG) ||
                mediaItem.getMimeType().equals(MediaItem.MIME_TYPE_PNG)) {
            return ImageFragment.newInstance(mediaItem);
        }
        if (mediaItem.isVideo()) {
            return VideoFragment.newInstance(mediaItem);
        } else if (mediaItem.isGif()) {
            return GifFragment.newInstance(mediaItem);
        } else  {
            return FastImageFragment.newInstance(mediaItem);
        }
    }
```

* ImageFragment 只能用于jpeg和png 的预览,采用imagesapling scale进行解码预览
* GifFragment 用于gif预览,采用glide解码
* VideoFragment 用于视频预览,相册未内置视频播放
* FastImageFragment 除上述类型外的其他类型,以及未知类型的预览, 采用imagezoom进行预览

