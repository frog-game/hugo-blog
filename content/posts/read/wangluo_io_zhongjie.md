---
title: "网络IO模型总结"
date: 2022-09-13T01:30:29+08:00
lastmod: 2022-09-13T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
-
tags:
- 网络底层
description: "网络IO模型, reactor 图示"
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
    image: "posts/read/wangluo_io_zhongjie/image-20220912104016515.png" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

##  网络编程流程

![](image-20220912104923737.png)

## 堵塞IO

![](image-20220912141757964.png)

##  非堵塞IO

![](image-20220912141809146.png)

## 信号驱动IO

![](image-20220912142709884.png)

## 异步io模型

![](image-20220912143406827.png)

## 多路复用

![](image-20220912141923315.png)

## 单reactor

![](image-20220912115933887.png)

代表作：redis 内存数据库

注意：`redis 6.0 以后是多线程`

## 单reactor 多进程模型

![](image-20220912133954912.png)

代表：nginx

## 单reactor模型 + 任务队列 + 线程池

![](image-20220912103644712.png)

代表作:skynet

## 主从 reactor

![](image-20220912125034816.png)

代表作：netty

##  多reactor + 多线程

![](image-20220912140325111.png)

代表作：memcache

##  多reactor + 多线程 +协程池

![](image-20220912104016515.png)



