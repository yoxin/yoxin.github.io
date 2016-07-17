---
title: Android面试题（二）——IPC机制
s: android-interview-questions2-IPC
date: 2016-07-11 10:41:56
tags: android
---
## 引言
***
  - IPC是Inter-Process Communication的缩写，含义是**进程间通信和跨进程通信**，是指**两个进程直接进行数据交换的过程**。 
  - **Binder机制是Android 采用的独特的进程间通信机制**。基于OpenBinder框架的一个驱动，用于提供Android平台的进程间通信。
  - **Messenger**、**ContentProvider**、**AIDL**底层实现都是**Binder**。
<!--more-->

## 面试题
***
  1. ### Android IPC有哪些方式？优缺点和适用场景？
    - **Bundle**：在Bundle中附加数据并通过Intent传输
    - **文件共享**：两个进程通过读写一个文件来交换数据
    - **AIDL**：Android Interface Definition Language
    - **Messenger**：基于消息的进程间通信
    - **ContentProvider**：：专门用于不同应用间的数据共享
    - **Socket**：使用TCP和UDP协议进行网络通信
![IPC方式的优缺点和适用场景](http://upload-images.jianshu.io/upload_images/1944615-90ad5871e7034239.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**注：RPC——远程过程调用**

  2. ### Binder的系统架构
    - **Service Manager**
Service Manager主要负责Android系统中所有的服务，当客户端要与服务端进行通信时，首先就会通过Service Manager来查询和取得所需要交互的服务。当然，每个服务也都需要向Service Manager注册自己提供的服务，以便能够供服务端进行查询和获取。
    - **服务（Service）**
这里的服务即上面所说的服务端，通常也是Android的系统服务，通过Service Manager可以查询和获取某个Server。
    - **客户端**
这里的客户端一般是指Android系统上面的应用服务。它可以请求Service中的服务，比如Activity。
    - **服务代理**
服务代理是指在客户端应用程序中生成的Server代理（proxy）。从应用程序的角度来看，服务代理和本地对象没有差别，都可以调用其方法，方法都是同步的，并且返回相应的结果。服务代理也是Binder机制的核心模块。
    - **Binder驱动**
用于实现Binder的设备驱动，主要负责组织Binder的服务节点，调用Binder相关的处理线程，完成实际的Binder传输等，他位于Binder结构的最底层（即Linux内核层）。
它们之间的结构关系如下：
![Android系统Binder机制中的四个组件Client、Server、Service Manager和Binder驱动程序的关系](http://upload-images.jianshu.io/upload_images/1944615-0a546c6de18e3afc.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
      1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中
      2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server
      3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信
      4. Client和Server之间的进程间通信通过Binder驱动程序间接实现
      5. Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力

  3. ### Binder的工作流程
    1. 客户端首先获取服务器端的代理对象。所谓的代理对象实际上就是在客户端建立一个服务端的“引用”，该代理对象具有服务端的功能，使其在客户端访问服务端的方法就像访问本地方法一样。
    2. 客户端通过调用服务器代理对象的方式向服务器端发送请求。
    3. 代理对象将用户请求通过Binder驱动发送到服务器进程。
    4. 服务器进程处理用户请求，并通过Binder驱动返回处理结果给客户端的服务器代理对象。
    5. 客户端收到服务端的返回结果。 
![Binder的工作机制.png](http://upload-images.jianshu.io/upload_images/1944615-3c92d9d160957e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  4. ### 实现一个Messenger的步骤
答：基于消息的进程间通信方式：
![Messanger通信.png](http://upload-images.jianshu.io/upload_images/1944615-ec958fb44965258d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    - 服务端进程
需要在服务端创建一个Service来处理客户端的连接请求，同时创建一个Handler并通过它来创建一个Messenger对象，然后在Service的onBind中返回这个Messenger对象底层的Binder即可。
    - 客户端进程
客户端进程中，首先绑定服务端的Service，绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息，发送消息的类型为Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还需要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。
![Messenger的工作原理.png](http://upload-images.jianshu.io/upload_images/1944615-7b6a97fd0049a437.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  5. ### AIDL的工作流程
答：AIDL文件的本质是系统为我们提供的一种快速实现Binder的工具而已，所以AIDL的工作流程可以绕到Binder进行说明。（结合第一和第二问进行回答）

  6. ### AIDL的使用方法
    - **服务端**
服务端首先要创建一个远程Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。
    - **客户端**
首先绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转化成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。
    - **AIDL文件支持的数据类型**
      - 基本数据类型；
      - String和CharSequence；
      - List：只支持ArrayList，里面每个元素都必须被AIDL支持；
      - Map：只支持HashMap，里面每个元素都必须被AIDL支持，包括key和value；
      - Parcelable：所有实现了Parcelable接口的对象；
      - AIDL：所有AIDL接口本身都可以在AIDL文件中使用。
    - Parcelable和AIDL对象无论是否和当前AIDL文件位于同一个包内，都要显式import进来。
    - 服务端实现需要注意并发处理，可以借助Copy-On-Write容器
    - 服务端实现监听时，监听存储容器使用RemoteCallbackList，系统专门提供用来删除跨进程listener的接口
    - AIDL的包结构在服务端和客户端要保持一致，否则出错，因为客户端要反序列化服务端中和AIDL接口相关的所有类，如果类的完整路径不一样的话，就无法成功反序列化。
    - AIDL调用服务端方法后，会挂起等待，如果服务端进行执行大量耗时操作，会导致客户端ANR。解决方法：客户端调用放在非UI线程即可。

  7. ### Intent传递数据时，下列的数据类型不可以被传递的是（）
    A. Serializable
    B. File
    C. Parcelable
    D. Thread
    - 答案：D
    - 解释：Intent传递Bundle数据，**Bundle支持的数据类型有四种**：
      1. 8种基本数据类型及其数组
      2. String（String实现了Serializable）/CharSequence实例类型的数据及其数组
      3. 实现了Parcelable的对象及其数组  (自己来做, 操作较复杂, 但速度快)
      4. 实现了   Serializable   的对象及其数组(系统来做, 操作简单, 但速度慢) 

## 知识点：
***
1. Android IPC方式
2. Binder机制的原理
3. Messenger和AIDL的使用和工作原理

## 推荐
***
[我的个人博客](yoxin.github.io)
[Android面试题（一）——Activity的生命周期和启动模式
](http://www.jianshu.com/p/e279b3137157)

## 参考资料
***

[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)
[Android 基于Message的进程间通信 Messenger完全解析](http://blog.csdn.net/lmj623565791/article/details/47017485)
[远程过程调用协议](http://baike.baidu.com/link?url=PriROtglDLDAnwUlFMN2i80-FPB_k-4PCHO-dEZCYUaI7bmyFCEgQ_IRKuPA5gUN0zrjJYPcwtYyK6iG29wFpwPWPRg3WvBKZloj_CQlApy)
[聊聊并发-Java中的Copy-On-Write容器](http://ifeve.com/java-copy-on-write/)
《Android开发艺术探索》
《Android技术内幕系统卷》
《深入理解Android卷1》
《Android系统源代码情景分析》
