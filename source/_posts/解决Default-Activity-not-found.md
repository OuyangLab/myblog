---
title: 解决Default Activity not found
date: 2020-07-06 21:36:35
tags:
---

今天把Android studio开发工具版本升级到4.0后，拉取项目工程代码运行的时候一直报这个错误：Error running :
Default Activity not found
网上各种方法，什么重启AS啊，清缓存啊，啥都不管用。最后的解决方案是手动把位于C盘目录下的C:\Users\aaron\.AndroidStudio4.0\system\caches文件夹删掉，再重启AS问题才得以解决。