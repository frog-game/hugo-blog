---
title: "HybridCLR IOS打包热更新"
date: 2022-12-19T01:30:29+08:00
lastmod: 2022-12-19T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
- 
tags:
- HybridCLR
description: "HybridCLR IOS打包热更新"
weight:
draft: false # 是否为草稿
comments: true
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "posts/tech/Hybridclr-ios-dabao/image-20221207103651120.png" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## 使用的HybridCLR版本,unity版本

![image-20221207103651120](image-20221207103651120.png)

![image-20221207103712812](image-20221207103712812.png)

![image-20221207113033012](image-20221207113033012.png)

## XAsset打包过程[这个过程每个公司,每个项目都不一样,但是原理差不多,下面是我这里的步骤]

![image-20221207104252719](image-20221207104252719.png)

![image-20221207104411454](image-20221207104411454.png)

![image-20221207111900037](image-20221207111900037.png)

![image-20221207131703794](image-20221207131703794.png)

![image-20221207131815388](image-20221207131815388.png)

## HybridCLR打包过程

![image-20221207101150191](image-20221207101150191.png)

![image-20221207101256646](image-20221207101256646.png)

![image-20221207101426935](image-20221207101426935.png)

![image-20221207104001828](image-20221207104001828.png)

![image-20221207102956410](image-20221207102956410.png)



![image-20221207103214670](image-20221207103214670.png)

![image-20221207103506102](image-20221207103506102.png)

## Xcode上的要完成的一些前提

**`IOS XCode下记得编译libil2cpp.a`**[马赛克是自己的一些个人信息,进行了涂抹不用关心]

![image-20221207114145009](image-20221207114145009.png)

执行后就生成

![image-20221207114401827](image-20221207114401827.png)

然后按下面步骤进行替换

![image-20221207114917225](image-20221207114917225.png)

![image-20221207114558466](image-20221207114558466.png)

![image-20221207114828073](image-20221207114828073.png)

然后就可以愉快的打IOS包了 如果XCode有其他报错,就自己根据项目解决吧。这个就不截图了

## 热更新流程

![image-20221219111132004](image-20221219111132004.png)

![image-20221219111350314](image-20221219111350314.png)

![image-20221219111502477](image-20221219111502477.png)

**`这一步要注意去git查看是否真的生成了需要的内容只有打勾的地方才说明生成成功了`**

![image-20221219111533209](image-20221219111533209.png)

![image-20221219111828610](image-20221219111828610.png)

![image-20221219111938878](image-20221219111938878.png)