# 相册开发文档
 链接 https://sherlockhouse.github.io/

## 目录结构

    app/         # 主应用
        com.
            android.*/ # 谷歌原生的相册代码
            arch.*/    #相册通用的框架代码
            cmcc.*/    # 由于需要修改包名,部分代码必须放到包名同名目录
            mediatek.*/# MTK 原生相册代码目录
        renderscript/ # MTK 原生的图片编辑库
        assets/       # tensorflow lite 模型文件等




    base/		 # 项目通用的基础框架
    imageZoom/   # github开源的图片查看放大库
    matisse/     # 知乎开源的图片选择库
    mtkgallery2/ # mtk平台原生的资源文件




    # network 网络相关的库,项目中没有用到,保留
    Network/
    NetworkLite/




## 功能模块

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



## 详细设计

### 媒体数据结构

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
### 数据库

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


### 界面导航
采用谷歌导航组件, 以nav_graph_albumset.xml为例


### 照片单张预览
