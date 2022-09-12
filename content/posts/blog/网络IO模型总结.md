## 网络编程流程

![image-20220912104923737](image/网络IO模型总结.assets/image-20220912104923737.png)

## 堵塞IO

![image-20220912141757964](image/网络IO模型总结.assets/image-20220912141757964.png)

## 非堵塞IO

![image-20220912141809146](image/网络IO模型总结.assets/image-20220912141809146.png)

## 信号驱动IO

![image-20220912142709884](image/网络IO模型总结.assets/image-20220912142709884.png)

## 异步io模型

![image-20220912143406827](image/网络IO模型总结.assets/image-20220912143406827.png)

## 多路复用

![image-20220912141923315](image/网络IO模型总结.assets/image-20220912141923315.png)

## 单reactor

![image-20220912115933887](image/网络IO模型总结.assets/image-20220912115933887.png)

代表作：redis 内存数据库

注意：`redis 6.0 以后是多线程`

## 单reactor 多进程模型

![image-20220912133954912](image/网络IO模型总结.assets/image-20220912133954912.png)

代表：nginx

## 单reactor模型 + 任务队列 + 线程池

![image-20220912103644712](image/网络IO模型总结.assets/image-20220912103644712.png)

代表作:skynet

## 主从 reactor

![image-20220912125034816](image/网络IO模型总结.assets/image-20220912125034816.png)

代表作：netty

## 多reactor + 多线程

![image-20220912140325111](image/网络IO模型总结.assets/image-20220912140325111.png)

代表作：memcache

## 多reactor + 多线程 +协程池

![image-20220912104016515](image/网络IO模型总结.assets/image-20220912104016515.png)



