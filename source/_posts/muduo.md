---
title: muduo
date: 2025-10-14 10:59:42
categories: proj
---

# 项目总结

## 描述

模拟muduo网络库，用C++11的新特性实现了非阻塞IO复用的高并发TCP服务器模型核心（TCPServer），用C++11的新特性取代了Boost，同时提升了Buffer等组件的性能，实现了0第三方库的依赖，仅需linux内核支持。

## 实现

- 整体采用non-blocking+IO-multiplexing+loop线程的设计框架，其中线程模型采用one loop per thread的多线程服务端网络编程模型，结合Reactor模型进行实现。
  其中主Reactor（1线程）专职于accept()，同时采用EPOLLEXCLUSIVE避免惊群。子Reactor（N默认等于CPU核心数），轮询处理已连接的客户端数据。
- 去掉了Muduo库中的Boost依赖，使用C++11的标准：
    1. 使用了智能指针：unique，shared，weak对Poller，Channel等的内存资源释放，对TcpConnection建立和Channel绑定等
    2. 使用atomic原子操作类型对状态量进行保护，用unique_lock替代lock_guard
- 缓冲区参照Netty设计，使用prepend，read和write三个指针的设计，划分缓冲区数据
- 日志系统采用同步输出方式，使用snprintf进行INFO、DEBUG、ERROR、FATAL四个等级的格式化输出...
- 使用开发的网络库实现了EchoServer

# 概念介绍

## Reactor模型

Reactor模型是一种事件驱动的并发编程模型，主要用于处理大量并发IO操作。

### 基本工作流程

1. 注册感兴趣的事件（读就绪，写就绪事件）
2. Reactor（事件分发器）监听所有注册的事件
3. 事件发生时，Reactor分发给相应的Handler（事件处理器）
4. 事件处理器执行具体的业务逻辑

### 主要组件

- Reactor

  事件分发器，使用了多路IO复用技术比如select，poll，epoll

- Handlers

  事件处理器，比如Acceptor，ReadHandler

- Demultiplexer

  多路分发器，比如epoll

## 阻塞和非阻塞IO

### 定义

- 阻塞IO

  当进程发起IO操作时，进程会被挂起，直到操作完成才返回

  在等待期间，进程不能执行其他任务

- 非阻塞IO

  当进程发起IO操作时，立即返回，不管操作是否完成

  进程需要主动检查操作是否完成

### 具体例子

- 阻塞

  ```cpp
  // 阻塞模式下的 read 操作
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  // ... 连接设置 ...
  
  char buffer[1024];
  int n = read(sockfd, buffer, sizeof(buffer));  // 阻塞直到有数据或出错
  // 只有当有数据可读时，read 才会返回
  ```

- 非阻塞

  ```cpp
  // 设置为非阻塞模式
  int flags = fcntl(sockfd, F_GETFL, 0);
  fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
  
  char buffer[1024];
  int n = read(sockfd, buffer, sizeof(buffer));
  if (n < 0) {
      if (errno == EAGAIN || errno == EWOULDBLOCK) {
          // 暂时没有数据可读，需要稍后重试
          // 程序可以去做其他事情
      } else {
          // 真正的错误
      }
  } else {
      // 成功读取 n 字节数据
  }
  ```

## 同步和异步IO

### 定义

- 同步IO

  应用程序主动等待IO操作完成

  IO操作期间，应用程序被阻塞或需要主动检查

- 异步IO

  应用程序发起IO操作后立即返回

  由内核在IO操作完成后通知应用程序

# 问题

## 项目问题

### Reactor模型是如何体现的

Reactor模型的核心组成就是reactor，handlers和demultiplexer，其中：

Reactor对应的是Eventloop

handlers对应的是channel类，分装了文件描述符和处理回调函数

demultiplexer对应的就是epoller，监听所有的channel

### 工作流程

- Acceptor（维护的是主Reactor）监听新连接，当新连接到达时，Acceptor创建一个Tcpconnection对象，轮询注册到一个SubReactor的EpollPoller中。
- Subloop通过Epoll监听套接字是否有事件触发，如果有，EpollPoller返回活跃的Channel，EventLoop主动调用Channel的HandleEvent去执行对应回调。
- Channel回调函数是用户设置的，实现了业务逻辑的解耦。

Muduo属于多Reactor多线程设计，一个主Reactor+N个子Reactor。工作流程中，主Reactor通过轮询把新连接分配给子Reactor中，子Reactor通过EpollPoller监听事件并触发Channel回调，实现高并发处理。

### 为什么选择one loop per thread模型

这个模型是一种常见的并发网络编程模式，他有四个优点：
       1. 简化并发编程，每个线程独立运行自己的事件循环。
       2. 减少锁竞争，事件处理在单个线程内完成
       3. 不同连接由不同线程处理，一个连接的问题不会影响其他连接
       4. 每个线程可以绑定不同的CPU核心



### 使用这个网络库可以做什么？

TCP长连接服务，比如即时通讯、聊天服务器、金融交易系统、代理服务器



### 实现一个服务器需要考虑哪些？

1. 线程模型的设计

   one loop per thread

   如何实现跨线程调用？

   跨线程调用的安全方案为：外部线程通过`runInLoop()`方法提交任务，任务被加入队列并唤醒事件循环.EventLoop在下次迭代中处理队列任务。使用`wakeupfd`实现跨线程唤醒，即发送一个单字节数据。

2. 协议处理

   PLC HTTP处理

3. 性能优化

   负载均衡算法

4. 连接生命周期

   muduo库中的TcpConnection继承自enable_shared_from_this，使用shared_ptr进行生命周期的管理。

   TcpServer这里创建的是shared_ptr的TCPconnection，拥有主要的所有权；连接建立时，Channel只持有弱引用，观察还是否存在。用户回调只会获得临时的shared_ptr（如果存在即tie_.lock()临时提升，提升后出了作用域引用计数则-1）。

   为什么要这么做？因为tcpconnector的回调传给channel，但是channel调用的时候tcpconnector可能已经被destroy了。

### 如何分配子Reactor，轮询算法用到了什么负载均衡算法

1. 轮询分配 按顺序依次分配
2. 加权分配 根据子Reactor的处理能力分配不同权重（CPU不同核心访问内存速度不一定一样）
3. 最少连接数 分配给当前连接数最少的线程

### 为什么使用多线程而不是多进程？  
线程更轻量，性能和资源开销更小；数据共享（连接状态和上下文）和通信效率更高；更适配reactor模型。

## 核心模块实现细节

### Channel

#### Channel如何与Eventloop和Epollpoller交互？

Epollpoller和eventloop里都维护了channel，eventloop轮询时调用epollwait，每一轮返回有事件的fd，通过维护的map，和fd，获取到对应的channel，eventloop直接触发对应channel的回调handleevent

#### 如何处理不同的事件类型？（读写错误等等）

Channel首先通过enableRead、enableWrite等方法处理可读可写事件，设置为感兴趣事件，返回时通过revent确定具体事件类型，在handleevent函数中，选择对应的分支选择对应的函数进行处理



### Epollpoller

#### epoll对比poll和select的优势是什么？

1. 时间复杂度低，只返回就绪的fd
2. 连接较多时，性能更加稳定
3. 监控的fd上限仅受系统内存限制

#### epoll的LT和ET你选择了哪种，优势是什么？

  LT，只要fd处于就绪状态就会持续通知，开发简易性优先的场景，不容易遗漏事件。但是要注意在数据量大的时候主动开关epollout。



### EventLoop

#### loop工作流程是什么？

1. 事件收集 通过系统调用（比如epoll_wait）收集所有就绪的IO事件，返回后填充活跃Channel列表
2. 事件分发 遍历每个活跃的Channel，调用每个Channel的handleEvent方法，Channel根据具体事件调用注册的回调
3. 异步任务 执行其他线程提交到本线程的任务

#### 如何实现跨线程调用？（如何保证线程安全？）

首先介绍一下Muduo核心的线程模型：One Loop Per Thread，即每个EventLoop对象严格绑定一个特定的线程。

跨线程调用的安全方案为：外部线程通过runInLoop()方法提交任务，任务被加入队列并唤醒事件循环，EventLoop在下次迭代中处理队列任务。使用wakeupfd实现跨线程唤醒，即发送一个单字节数据。

### TcpConnection

#### 连接的生命周期如何管理？

muduo库中的TcpConnection继承自enable_shared_from_this，使用shared_ptr进行生命周期的管理。

TcpServer这里创建的是shared_ptr的TCPconnection，拥有主要的所有权；连接建立时，Channel只持有弱引用，观察还是否存在。用户回调只会获得临时的shared_ptr（如果存在即tie_.lock()临时提升，提升后出了作用域引用计数则-1）。

#### 为何使用shared_ptr?

1. tcpserver拥有shared_ptr是为了保证服务运行期间持续存活，避免意外提前释放。
2. 与之配合的weak_ptr能有效地避免在调用回调时已经被释放。

### Buffer设计

#### 为什么采用三个指针，各自的作用是什么？

BUffer的三个指针分别是readerIndex/writerIndex/以及首尾指针。

读指针标记已接收但是还没被应用层消费的数据，写指针标记可写内存区域的起始位置。

在读缓冲区，读指针是待处理数据的起始位置，写指针是新数据追加位置；而在写缓冲区，读指针是待发送数据的起始位置，写指针是新数据追加位置。

#### 如何实现高效的数据读写？

 1. 减少内存拷贝的优化
     直接暴露内部指针给用户，避免数据拷贝。
     使用readv/writev，单次系统调用填充多个缓冲区，当主缓冲区空间不足时，自动使用栈上的缓冲区。
 2. 动态扩容
     如果可以先优化Buffer内的内存分配，如果还是不够再扩容，可以考虑指数增长。

## C++11特性应用

1. 你用了哪些智能指针 为什么在这些场景下用这些特定的智能指针？

    unique_ptr、shared_ptr以及weak_ptr都用到了。
    
    unique_ptr用到的地方比较多，比如tcpconnector的channel、socket；tcpserver里的acceptor；eventloop里的poller、wake_channel等
    shared_ptr和weak_ptr成对使用，tcpserver新建tcpconnection时用的是shared_ptr，但是channel使用weakptr去接收tcpconnection，因为Channel持有tcpconnection的引用，但是tcpconnection拥有channel的所有权，会出现循环引用

2. atomic和mutex的选择标准是什么？

    优先选择atomic的情况：1. 简单的数据类型 2. 仅需保证单个变量的原子性 3. 没有依赖关系 4. 高频操作
    局限在不支持多变量同步或者符合变量

    优先选择mutex：1. 复杂数据结构，比如map vector 2. 多变量操作 3. 可以配合条件变量，执行复杂逻辑
    局限在可能引发死锁

3. C++11的移动语义在你的项目中有什么应用？

    回调函数的赋值操作使用到了move

4. 当TcpConnection被多线程持有，shared_from_this在析构中调用是否安全？

    不安全，此时引用计数可能已经变成0了。在析构中如果一定需要用，需要加一个weak_ptr的判断，不过根本原则是在析构函数中禁止调用。

## 项目经验

1. 在实现过程中遇到的最大挑战，如何解决的？

    多线程下TcpConnection对象在IO线程和业务线程间共享，会不会出现这么一个问题：IO线程在发送数据的时候，业务线程想要delete conn这样会导致崩溃。
    所以考虑使用shared_ptr和weak_ptr配合使用。即在进行操作时（无论是handleclose韩式handleread等操作），线程都使用weakptr去接收Tcpconnection，如果此时还存在则升级到shared_ptr进行操作，操作结束后引用计数自动-1，避免了对象提前被析构的问题。所有的销毁操作都延迟到IO线程中的事件循环中去执行。

2. 如果让你继续优化这个项目，你会从哪些方面着手？

    日志库的添加；现在只有IO线程和简单的工作线程，考虑添加其他复杂的工作线程。

3. 从muduo源码中你能学到哪些优秀的代码设计？

    Reactor模式的高效实现；多Reactor线程模型；对象生命周期的控制；Buffer的设计；跨线程的任务提交；接口设计和函数命名

## 网络编程基础

1. TCP服务器的基本工作流程

    创建监听套接字->创建事件监听器->事件循环->处理新连接->处理IO->关闭连接

2. 非阻塞IO相比阻塞IO的优势和劣势

    优点：1. 高并发 2. 资源利用率高 3. 响应速度更快 4. 避免线程阻塞
    缺点：1. 编程复杂度高 2. 调试困难 3. 持续重试读写导致的CPU空转风险  
