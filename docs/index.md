# Gallery Developer Docs


## Project layout

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
