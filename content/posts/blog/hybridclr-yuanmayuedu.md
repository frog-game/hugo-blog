---
title: "Hybridclr-源码阅读"
date: 2022-12-07T01:30:29+08:00
lastmod: 2022-12-07T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
- 
tags:
- Unity
description: "Hybridclr-源码阅读"
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
    image: "posts/blog/hybridclr-yuanmayuedu/image-20221204163813761.png" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## 一些前提知识点

![image-20221204160751223](image-20221204160751223.png)

### 代码术语区别

- `IL`:微软平台上的一门中间语言，我们常写的`C#`代码在编译器中都会自动转换成`IL`,中间语言是编译使用高级 .NET 语言编写的代码后获得的结果。 对使用其中一种语言编写的代码进行编译后，即可获得 IL 所生成的二进制代码

  ![image-20221204002617483](image-20221204002617483.png)

- 托管代码：`托管代码`就是执行过程交由运行时管理的代码。 在这种情况下，相关的运行时称为公共语言运行时 (CLR)，不管使用的是哪种实现（例如 Mono、.NET Framework 或 .NET Core/.NET 5+）。 CLR 负责提取托管代码、将其编译成机器代码，然后执行它。 除此之外，运行时还提供多个重要服务，例如自动内存管理、安全边界、类型安全，把托管代码理解成上面的IL中间语言也行
- 非托管代码：`非托管代码`（Unmanaged Code）是指直接编译成目标计算机的机器码，这些代码只能运行在编译出这些代码的计算机上，或者是其他相同处理器或者几乎一样处理器的计算机上。`非托管代码`不能享受公共语言运行库所提供的一些服务，例如内存管理、安全管理等。`非托管代码`（Unmanaged Code）不由CLR公共语言运行库执行，而是由操作系统直接执行的代码,如果非托管代码需要进行内存管理等服务，就必须显式地调用操作系统的接口，通常非托管代码调用Windows SDK所提供的API来实现内存管理。
- 原生代码:`native code`是本地`cpu`的目标执行代码, 不是`IL`, 所以速度很快, 它的执行不依赖某个虚拟机或者解释器，编译后可直接依附操作系统运行，不需要经过虚拟机之类的东西
- 程序集:程序集（Assembly）的文件负责封装中间语言，程序集中包含了描述所创建的方法、类以及属性的所有元数据

### 编译器

- c#编译器: 将`c#`编译为`IL` `[C#---->CIL]`
- Mono Runtime编译器:将`IL`转换成原生码，然后让`Mono`运行时去执行,这样其实也达到了`c#`跨平台的效果

### 平台编译

**build**

**`编译GCC的平台`**

**host**

**`运行GCC的平台`**

**target**

**`GCC编译产生的应用程序的运行平台`**

- native compiler:三者全部相同（build = host = target）就是`原生编译` 我们在PC上装的Ubuntu或者Fedora里面带的GCC，就是native compiler
- cross compile:如果build = host，但是target跟前两者不同就是 `交叉编译`开发手机应用程序的编译器，通常运行在PC或Mac上，但是编译出来的程序无法直接在PC或Mac上执行

### 编译方式

- 即时编译[**`Just in time, JIT`**]: 就是在程序运行的时候把CIL的byte code 转成目标平台的原生码,也就是Mono Runtime编译器干的活
- 提前编译[**`Ahead of time,AOT`**]:程序运行前将exe或者dll里面的CIL的byte code部分转成目标平台的原生码,并且存储起来，好加快速度,但是程序中还是会有部分的CIL的byte code需要JIT编译
- 完全静态编译[**`Full ahead of time,Full-AOT`**]:就是将所有源码编译成目标平台所需要的原生码

### IOS不支持JIT编译的原因

- IOS并非把JIT禁止了，主要还是IOS封存了内存的[`可执行权限`]，变相的封锁了JIT编译方式

## IL2CPP

就是上面的`2.1编译方式`中的AOT提前编译

### 分为下面两部分

1. AOT编译器

​		把IL中间语言转成CPP文件的作用

2. 运行时库

​		主要是做垃圾回收,线程文件的获取，还有对托管数据的原生代码进行修改

### 为啥要转换成CPP

1. 运行速度快,这个毋庸置疑
2. mono 为了跨平台，他是通过VM来实现的，也就是说有几个平台，就需要实现几个VM,这种方法耗时耗力,而且为了实现各个平台的支持和移植，势必要把代码进行修改，然后出现bug在进行修复，来来回回时间和精力花费不少，所以为啥不利用现成各个平台的C++编译器执行了,这也是IL2CPP的核心思想
3. 虽然最后代码都变成的静态的C++但是内存管理这块还是需要遵循c#的标准和方式,这也是为什么最后还需要一个 IL2CPP VM的原因 这个时候VM做得主要还是GC的管理,线程的创建等等一些辅助服务性的工作

### 工作原理就直接上官网的图了

![image-20221203100515921](image-20221203100515921.png)



![image-20221204163051027](image-20221204163051027.png)

### 为什么IL2CPP不支持热更新

因为IL2CPP是一个纯静态的AOT运行时，然后不支持运行时加载DLL，所以是不支持热更新

然后hybridclr扩充了IL2CPP的代码,使他从AOT运行方式变成了AOT+Interpreter的混合方式从而可以动态的加载dll实现热更新

## Hybridclr的原理

dll不过是元数据和代码的集合,aot与 热更新dll的区别只不过一个函数以aot代码方式执行，一个以解释方式执行,最后都会直接在虚拟机层面将aot和热更新dll统一对待

为什么Hybridclr能做到如此统一和彻底，因为元数据不过是数据，不管aot还是热更新是没有本质区别的

而托管代码执行，依赖的不过是代码和数据

Hybridclr分两个工程

1. Hybridclr工程进行了源码的编译解析，在这里可以理解成这个工程做得主要事情是解释器的工作,此解释器是在IL2CPP VM扩充的，不解的可以看下图

2. 第二个工程是il2cpp_plus,这个工程会把泛型方法、泛型数据类型、以及其他的一些支持添加到IL2CPP里面,也可以简单的理解成为主要是为了给IL2CPP扩展功能用的，从而能让他动态的加载DLL

   ![image-20221204163813761](image-20221204163813761.png)

## 源码解析

   ### Hybridclr工程

这个是代码工程结构 后面会持续更新暂时只写了engine.cpp的源码分析

![image-20221207121054575](image-20221207121054575.png)

   **`阅读前提`**

1. MethodBridge: 方法桥接,这里的method其实就是c#里面的方法,也就是函数

2. ARM64:CPU的ARM架构 主流的手机/平板品牌，绝大数是采用ARM架构

3. 栈帧大概意思如下,hybridclr 也差不多是这种结构申请的空间和数据

   ```c#
   public class TestFrames {
       public static void main(String[] args) {
           method1(10);
       }
    
       private static void method1(int x){
           int y = x + 1;
           Object m = method2();
           System.out.println(m);
       }
    
       private static Object method2(){
           Object n = new Object();
           return n;
       }
   }
   ```

   ![image-20221204194419081](image-20221204194419081.png)

4. unity底层托管堆使用的是BoehmGC算法

   BoehmGC算法，是用的mark-sweep（标记-清扫）算法,具体和Java的gc算法类型,这块由于篇幅问题就不详细写了,太多内容一句话说不清楚,以后有时间整理一篇详细文章出来

5. `Module` 是 .dll 或 .exe 类型的可移植可执行文件，这些文件由一个或多个类和接口组成。单个模块可包含多个命名空间，而一个命名空间可跨越多个模块。
   作为一个单元部署的一个或多个模块组成程序集。 

6. Il2CppImage 这个结构体是程序集镜像,可以通过它来获取命名空间,class,方法,函数指针地址等等

   ```c++
   typedef struct Il2CppImage
   {
       const char* name;//名字
       const char *nameNoExt;//扩展名字
       Il2CppAssembly* assembly;//程序集指针
   
       TypeDefinitionIndex typeStart;//方法类型偏移位置开始
       uint32_t typeCount;//方法总数
   
       TypeDefinitionIndex exportedTypeStart;//导出类型偏移位置开始
       uint32_t exportedTypeCount;//导出类型总数
   
       CustomAttributeIndex customAttributeStart;//自定义属性偏移位置开始
       uint32_t customAttributeCount;//自定义属性总数
   
       MethodIndex entryPointIndex;//方法入口点索引
   
   #ifdef __cplusplus
       mutable
   #endif
       Il2CppNameToTypeDefinitionIndexHashTable * nameToClassHashTable;//name对应的class的hashTable
   
       const Il2CppCodeGenModule* codeGenModule;//Module指针
   
       uint32_t token;//通过他可以得到函数指针地址
       uint8_t dynamic;//没看到使用,估计是用来验证是不是动态lib使用的
   } Il2CppImage;
   ```
   
7. MethodInfo

   ```c++
   typedef struct MethodInfo
   {
       Il2CppMethodPointer methodPointer;//指向普通执行函数
       InvokerMethod invoker_method;//指向委托函数
       const char* name;//名字
       Il2CppClass *klass;//函数所属类指针
       const Il2CppType *return_type;//返回值类型
       const ParameterInfo* parameters;//参数信息
   
       union//generic instance method 
       {
           const Il2CppRGCTXData* rgctx_data; /* is_inflated is true and is_generic is false, i.e. a generic instance method */
           const Il2CppMethodDefinition* methodDefinition;//方法定义
       };
   
      
       union//uninflated generic method 
       {
           const Il2CppGenericMethod* genericMethod; /* is_inflated is true */
           const Il2CppGenericContainer* genericContainer; /* is_inflated is false and is_generic is true */
       };
   
       uint32_t token;
       uint16_t flags;
       uint16_t iflags;
       uint16_t slot;
       uint8_t parameters_count;
       uint8_t is_generic : 1; /* true if method is a generic method definition */
       uint8_t is_inflated : 1; /* true if declaring_type is a generic instance or if method is a generic instance*/
       uint8_t wrapper_type : 1; /* always zero (MONO_WRAPPER_NONE) needed for the debugger */
       uint8_t is_marshaled_from_native : 1; /* a fake MethodInfo wrapping a native function pointer */
   } MethodInfo;
   ```

## 下载

### Hybridclr 工程

- 下面是我正在阅读的Hybridclr作者初始源码版本

```sh
git clone  https://gitee.com/focus-creative-games/hybridclr.git  -b main
git reset --hard 0540b31aa739fd275d8cfcd861cb41568d4a982c
```

`执行上面的命令就能下载到我正在阅读的指定的分支,指定的commit版本`

- 下面是我加上的对应的注释版本

```sh
git clone https://github.com/frog-game/hybridclr-0540b31aa739fd275d8cfcd861cb41568d4a982c.git
```

`执行上面的命令就能下载到我加上的对应的注释版本`

### il2cpp_plus工程

- 下面是我正在阅读的il2cpp_plus作者初始源码版本

```sh
git clone -b v2019-1.0.0-rc --depth=1 https://github.com/focus-creative-games/il2cpp_plus.git
```

`执行上面的命令就能下载到我正在阅读的指定的tag版本`



