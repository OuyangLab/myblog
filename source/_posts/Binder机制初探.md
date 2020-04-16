---
title: Binder机制初探
date: 2020-03-27 17:23:43
index_img: /images/binder/12.jpg
tags:
---

整个文章目录如下图：

![alt](/Ouyang/images/binder/ha.png)

#### **1.Binder概述** 

- - Binder中文名“粘合剂”，粘合了两个不同的进程。那Binder到底是什么呢？
    - 从机制角度来说，Binder是一种Android实现跨进程通信（IPC）的方式
    - 从组成结构来说，Binder是一种虚拟的物理设备驱动
    - 从代码实现角度来说，Binder是一个类，实现了IBinder接口
    
  - Android 系统是基于 Linux 内核的，Linux 已经提供了管道、消息队列、共享内存和 Socket 等 IPC 机制。那为什么 Android 还要提供 Binder 来实现 IPC 呢？主要是基于**性能**、**稳定性**和**安全性**几方面的原因。

##### **性能**

​    从进程A将数据拷贝到进程B，传统的Socket主要用在跨网络的进程间通信和本机上进程间的低速通信，消息队列和管道采用存储-转发的方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存无需拷贝，但难以控制。Binder仅需要一次数据拷贝，性能上仅次于共享内存。

| IPC方式              | 数据拷贝次数 |
| -------------------- | ------------ |
| 共享内存             | 0            |
| Binder               | 1            |
| Socket/管道/消息队列 | 2            |

##### **稳定性**

​    Binder 基于 C/S 架构，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度讲，Binder 机制是优于内存共享的。

##### **安全性**

​    首先传统的 IPC 接收方无法获得对方可靠的进程用户ID/进程ID（UID/PID），从而无法鉴别对方身份。Android 为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志。传统的 IPC 只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。其次传统的 IPC 访问接入点是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。同时 Binder 既支持实名 Binder，又支持匿名 Binder，安全性高。

​    基于以上选择Binder的优势：

| 优势   | 描述                                                     |
| ------ | -------------------------------------------------------- |
| 性能   | 一次数据拷贝，性能上仅次于共享内存                       |
| 稳定性 | 基于 C/S 架构，职责明确、架构清晰，因此稳定性好          |
| 安全性 | 为每个 APP 分配 UID，进程的 UID 是鉴别进程身份的重要标志 |

 

#### **2. 传统Linux进程通信原理**

##### 进程隔离

- 操作系统中，进程与进程间内存是不共享的。两个进程就像两个平行的世界，A 进程没法直接访问 B 进程的数据，这就是进程隔离的通俗解释。A 进程和 B 进程之间要进行数据交互就得采用特殊的通信机制：进程间通信（IPC）

##### 进程空间划分

- 一个进程空间分为用户空间（User Space）和内核空间（Kernel Space）
- 进程间的用户空间数据不可共享
- 进程间的内核空间数据可共享
- 进程内用户空间与内核空间交互需要通过系统调用

- 系统调用

```
copy_from_user() //将数据从用户空间拷贝到内核空间
copy_to_user() //将数据从内核空间拷贝到用户空间
```

![alt](/Ouyang/images/binder/1.png)

##### Linux传统IPC通信

- 传统的 IPC 方式中，进程之间是如何实现通信的？消息发送方将要发送的数据存放在内存缓存区中，通过系统调用进入内核态。然后内核程序在内核空间分配内存，开辟一块内核缓存区，调用 copy_from_user() 函数将数据从用户空间的内存缓存区拷贝到内核空间的内核缓存区中。同样的，接收方进程在接收数据时在自己的用户空间开辟一块内存缓存区，然后内核程序调用 copy_to_user() 函数将数据从内核缓存区拷贝到接收进程的内存缓存区。这样数据发送方进程和数据接收方进程就完成了一次数据传输，我们称完成了一次进程间通信。如下图：

![alt](/Ouyang/images/binder/2.png)

- - - 可以看到传统的IPC通信需要两次数据拷贝，性能低下
    - 接收数据的缓存需要接收方提供，但接收方不知道缓存有多大，一般尽量开辟一个很大的缓存空间，有点浪费资源。

 

#### **3. Binder跨进程通信原理**

##### 内存映射

- Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。
- 简单示意图如下：
  - 假设进程1、2的虚拟内存区域同时映射到同1个共享对象；
  - 当进程1对其虚拟内存区域进行写操作时，也会映射到进程2中的虚拟内存区域。

![alt](/Ouyang/images/binder/3.png)

##### 实现过程

- 调用系统下的系统调用函数：mmap()
- 该函数的作用是创建虚拟内存区域 + 与共享对象建立映射关系。
- 函数原型

```
/**
  * 函数原型
  */
void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);

/**
  * 具体使用（用户进程调用mmap（））
  * 下述代码开辟了一片大小 = MAP_SIZE的接收缓存区 & 关联到共享对象中（即建立映射）
  */
  mmap(NULL, MAP_SIZE, PROT_READ, MAP_PRIVATE, fd, 0);
```

- - - - -  步骤1：创建虚拟内存区域
           步骤2：实现地址映射关系，即：进程的虚拟地址空间 ->> 共享对象
    
    Binder驱动 
    
    - 在 Android 系统中，运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块。
    - 综合概述如下图所示

![alt](/Ouyang/images/binder/4.png)

##### Binder通信机制模型

- 基于C/S模式
  - Client进程：使用服务的进程
  - Server进程：提供服务的进程
  - ServiceManager进程：管理Service注册与查询
  - Binder驱动：一种虚拟设备，连接Client进程、Server进程和ServiceManager进程的桥梁；通过内存映射传递进程间数据。

![alt](/Ouyang/images/binder/5.png)

 

 

 

 

 

 

 

 

 

##### Binder代理模式

- Client进程想要 Server进程中某个对象（Object）是如何实现的呢？毕竟它们分属不同的进程，Client进程没法直接使用 Server进程中的 Object。
- Client进程请求Server进程Object对象时候，Binder驱动会对数据做一层转换，返回一个代理对象ProxyObject，这个ProxyObject具有和Object对象一样的方法，不具备Server进程中的Object对象那些方法的能力，但请求的时候只需要把请求参数交给驱动即可。
- 当 Binder 驱动接收到 Client 进程的消息后，发现这是个 ProxyObject 就去查询自己维护的表单，一查发现这是 Server 进程 Object 的代理对象。于是就会去通知Server进程调用Object 的方法，并要求Server进程把返回结果发给自己。当驱动拿到Server进程的返回结果后就会转发给 Client进程。
- 示意图如下：

![alt](/Ouyang/images/binder/6.png)

##### Binder IPC通信过程

-  Binder 驱动在内核空间创建一个**数据接收缓存区**
- 接着在内核空间开辟一块**内核缓存区**，建立**内核缓存区**和**内核中数据接收缓存区**之间的映射关系，以及**内核中数据接收缓存区**和**接收进程用户空间地址**的映射关系；
- 发送方进程通过系统调用 copy_from_user() 将数据 copy 到内核中的**内核缓存区**，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。
- 如下图所示：

![alt](/Ouyang/images/binder/7.png)

#### **4. Binder机制在Android实践(AIDL)**

这里Server进程是开启了一个继承Service的类，来模拟进程间通信的。

##### 注册服务

- 这里我们用AS工具新建一个AIDL文件，IBookMananger.aidl，接口里面定义一个addBook()、getBookList()方法，其中Book类实现了Parcelable接口。

```
interface IBookMananger {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

            void addBook(inout Book book);

            List<Book> getBookList();
}
```

- - 创建一个和Book同名的aidl文件，Book.aidl

```
// Book.aidl
package com.example.aidldemo.bean;

parcelable Book;
```

- - 然后Make Project ，SDK为自动为我们生成对应的IBookManager类，目录在generatedJava

![alt](/Ouyang/images/binder/8.png)

##### 获取服务

- Client通过binderService绑定Server进程注册的Server进程
- 调用Server进程的onBind()得到创建的Binder对象的代理对象Proxy
- Client进程通过调用onServiceConnected()获得Server进程创建的代理对象Proxy。

```
	private ServiceConnection mServiceConnection = new ServiceConnection()
	{
		@Override
		public void onServiceConnected(ComponentName name, IBinder binder)
		{
                          //此处返回的是一个代理对象Proxy
			//通过服务端onBind方法返回的binder对象得到IBookManager的实例，得到实例就可以调用它的方法了
			mIBookMananger = IBookMananger.Stub.asInterface(binder);
		}

		@Override
		public void onServiceDisconnected(ComponentName name)
		{
			mIBookMananger = null;
		}
	};
	public static com.example.aidldemo.IBookMananger asInterface(android.os.IBinder obj)
	{
		if ((obj == null))
		{
			return null;
		}
		android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                //如果是同一个进程调用则返回IBookManager对象，否则跨进程调用则返回Proxy对象，即 
                  Binder类的代理对象。
		if (((iin != null) && (iin instanceof com.example.aidldemo.IBookMananger)))
		{
			return ((com.example.aidldemo.IBookMananger) iin);
		}
		return new com.example.aidldemo.IBookMananger.Stub.Proxy(obj);
	}
```

##### 使用服务

- Client进程 将参数（Book对象）发送到Server进程
- Server进程根据Client进程要求调用目标方法（即addBook函数）
- Server进程 将目标方法的结果（即加法后的结果）返回给Client进程

```
	//1. Client进程 将需要传送的数据写入到Parcel对象中
	@Override public void addBook(com.example.aidldemo.bean.Book book) throws android.os.RemoteException
	{
		android.os.Parcel _data = android.os.Parcel.obtain();
		android.os.Parcel _reply = android.os.Parcel.obtain();
		try {
			_data.writeInterfaceToken(DESCRIPTOR);
			if ((book!=null)) {
				_data.writeInt(1);
				book.writeToParcel(_data, 0);
			}
			else {
				_data.writeInt(0);
			}
	// 2. 通过 调用代理对象的transact（）将上述数据发送到Binder驱动，并且以_reply结果返回给Client进程
			mRemote.transact(IBookMananger.Stub.TRANSACTION_addBook, _data, _reply, 0);
			_reply.readException();
			if ((0!=_reply.readInt())) {
				book.readFromParcel(_reply);
			}
		}
		finally {
			_reply.recycle();
			_data.recycle();
		}
	}
	// 注：在发送数据后，Client进程的该线程会暂时被挂起
       // 所以不要在Server进程执行的耗时操作
			@Override
			public java.util.List<com.example.aidldemo.bean.Book> getBookList() throws android.os.RemoteException
			{
				android.os.Parcel _data = android.os.Parcel.obtain();
				android.os.Parcel _reply = android.os.Parcel.obtain();
				java.util.List<com.example.aidldemo.bean.Book> _result;
				try
				{
					// 1. Binder驱动根据代理对象沿原路将结果返回并通知Client进程获取返回结果，client进程会拿到这个result展示在UI
					// 2. 通过代理对象 接收结果（之前被挂起的线程被唤醒）
					_data.writeInterfaceToken(DESCRIPTOR);
					mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
					_reply.readException();
					_result = _reply.createTypedArrayList(com.example.aidldemo.bean.Book.CREATOR);
				}
				finally
				{
					_reply.recycle();
					_data.recycle();
				}
				return _result;
			}
```

​    通过addBook将数据发送给Server进程进行计算，并将结果返回给Client端，toast添加book的数量。Demo示意图如下：

​    ![alt](/Ouyang/images/binder/9.png)

总结，整个时序逻辑如下：

![alt](/Ouyang/images/binder/10.png)

PS：受水平有限，只是对Binder一个很浅的初步认识总结。

 

 

 

