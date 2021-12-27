[TOC]

# 修改记录

| 版本 | 修改日期   | 作者   | 修改内容 |
| ---- | ---------- | ------ | -------- |
| V1.0 | 2020.10.31 | 顾林成 | 创建     |



# 搜索设计概要



## 一、描述



```
* 搜索分字段匹配搜索和FTS 分词搜索
* 文件名搜索的优先级最低，因为速度最慢（谷歌支持文件名搜索，不支持重命名）,文件名作为脏数据，不予检索

* 同时，相册不需要重命名，重命名由文件管理器实现，重命名会触发媒体库更新，比较耗时。相册提供信息备注，和照片格式信息

* 分词搜索应用于文件描述，用户自定义信息，本地朋友圈

* 字段匹配用于文件夹名字，分类名字，日期，地点，等数据较少的文字匹配。

* 全局搜索只从数据库中检索，同时增加当前页面搜索，从内存数据中查找。
```





大数据量，不要频繁destory。需要viewpager + navigation

由于以下原因，bottomnavigationview 点击会重新创建fragment



What you are describing is the default behavior of the navigation component. when navigating **down**, the fragment you navigated from is not destroyed, only when navigating **up**.

personally I don't understand why you would want to notify the viewModel the fragment is destroyed, but if you want to run a certain piece of code when you are navigating to another destination, you could use NavController.OnDestinationChangedListener in your main activity (or in your fragment, but don't forget to remove the listener when it is destroyed), and do some action according to your start and end destinations.



谷歌的single activity 原则。



将viewpager作为一个fragment导航节点。用activity 中的navi graph 做导航。



改变默认长按行为