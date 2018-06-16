---
layout:     post
title:      Redis缓存架构
subtitle:   Redis缓存架构之Redis分布式环境搭建
date:       2018-06-16
author:     Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Redis
---

> 正所谓前人栽树，后人乘凉。
> 
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板
> 
> [我的的博客](http://qiubaiying.top)

# 前言
从 Jekyll 到 GitHub Pages 中间踩了许多坑，终于把我的个人博客[BY Blog](http://qiubaiying.top)搭建出来了。。。

本教程针对的是不懂技术又想搭建个人博客的小白，操作简单暴力且快速。当然懂技术那就更好了。

看看看博客的主页样式：

[![](http://upload-images.jianshu.io/upload_images/2178672-51a2fe6fbe24d1cd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](http://qiubaiying.github.io/)

在手机上的布局：

[![](http://upload-images.jianshu.io/upload_images/2178672-d58bb45f9faedb70.jpg)](http://qiubaiying.github.io/)

废话不多说了，开始进入正文。

# 快速开始

### 从注册一个Github账号开始

我采用的搭建博客的方式是使用 [GitHub Pages](https://pages.github.com/) + [jekyll](http://jekyll.com.cn/) 的方式。

要使用 GitHub Pages，首先你要注册一个[GitHub](https://github.com/)账号，GitHub 是全球最大的同性交友网站(吐槽下程序员~)，你值得拥有。

![](http://upload-images.jianshu.io/upload_images/2178672-e65e5cda50f38cef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)












