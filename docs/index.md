# 相册开发文档


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


​    

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


## 详细