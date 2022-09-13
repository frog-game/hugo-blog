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
-
description: ""
weight:
slug: ""
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
    网络IO模型总结: "" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## 1. 网络编程流程

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912104923737.png)

## 2. 堵塞IO

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets//image-20220912141757964.png)

## 3. 非堵塞IO

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912141809146.png)

## 4. 信号驱动IO

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912142709884.png)

## 5. 异步io模型

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912143406827.png)

## 6. 多路复用

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912141923315.png)

## 7. 单reactor

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912115933887.png)

代表作：redis 内存数据库

注意：`redis 6.0 以后是多线程`

## 8. 单reactor 多进程模型

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912133954912.png)

代表：nginx

## 9. 单reactor模型 + 任务队列 + 线程池

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912103644712.png)

代表作:skynet

## 10. 主从 reactor

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912125034816.png)

代表作：netty

## 11. 多reactor + 多线程

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912140325111.png)

代表作：memcache

## 12. 多reactor + 多线程 +协程池

![](https://raw.githubusercontent.com/frog-game/typora-notes/main/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/%E7%BD%91%E7%BB%9C/image/%E7%BD%91%E7%BB%9CIO%E6%A8%A1%E5%9E%8B%E6%80%BB%E7%BB%93.assets/image-20220912104016515.png)



