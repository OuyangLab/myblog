---
title: Android屏幕适配方案
date: 2020-03-27 17:19:01
index_img: /images/pingmu/pingmu.jpg
tags: Android
---
##### 布局适配

- 使用RelativeLayout，即使屏幕大小改变，但控件的相对位置不变。
- 使用限定符，对不同设备大小屏幕，可以使用尺寸限定符（layout-large）创建布局文件，大号设备布局用layout-large布局，默认用layout；还可以用最小宽度(Smallest-width)限定符

##### 布局组件适配

- 使用wrap_content、 match_parent和weight来控制视图组件的宽度和高度
- 还有可以设置minHeight，minWidth

##### 图片资源适配

- 使用.9图片资源，自动拉伸位图。.9.png，会根据控件的大小自动拉伸你想要的部分。

##### 布局控件适配

- 使用密度无关像素dp来指定控件大小，density-independent pixel 可以保证不同屏幕像素密度设备显示相同的效果.