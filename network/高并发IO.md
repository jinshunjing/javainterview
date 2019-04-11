# 高并发I/O基础
https://www.jianshu.com/writer#/notebooks/33672046/notes/41681996

##准备知识

### 用户态与内核态
- 操作系统将虚拟空间分成用户空间与内核空间。用户进程不能访问内核空间。只有系统调用才可以访问内核空间。
- 操作系统会将IO数据缓存在文件系统的页缓存，也就是说，数据会先被拷贝到内核的缓存区，然后才拷贝到用户空间。

### I/O的两个阶段
- 等待IO事件就绪
- 拷贝数据：将数据从内核缓冲区拷贝到用户缓存区

### 五种网络I/O模型
|模型|描述|
|------|------|
|阻塞式I/O|等待数据与拷贝数据都阻塞|
|非阻塞式I/O|轮询数据是否就绪|
|I/O多路复用|事件驱动：select, poll, epoll|
|信号驱动I/O|使用信号处理函数来处理数据的拷贝|
|异步I/O|等待数据与拷贝数据都不阻塞|

## I/O多路复用

### select
检查的文件描述符列表作为参数传入，数量限制在1024

### poll
去除了数量限制，但是依然要遍历所有传入的文件描述符，并且从用户态拷贝到内核态

## epoll

### 数据结构
- 利用mmap技术避免了拷贝和遍历操作
- 红黑树存储注册的文件描述符
- 就绪链表存储就绪的事件

### 流程
- epoll_create: 建立一个epoll对象
- epoll_ctl: 向epoll对象添加socket链接，并注册一个回调函数ep_poll_callback，当驱动器有事件就绪时插入就绪列表
- epoll_wait: 收集就绪列表的就绪事件，调用ep_poll返回文件描述符，并且通过ep_send_events向用户空间发送就绪事件

## Reactor模式

### 简单模式
- Reactor/Initiation Dispatcher: 分发器，管理Handler的注册，从Synchronous Event Demultiplexer获取事件，然后分发给Handler处理
- Synchronous Event Demultiplexer: 同步事件分离器/同步事件多路复用，等待事件的发生。比如Java NIO里的Selector
- Event Handler: 请求的处理
- Handle：表示一种资源，比如socket文件描述符

### 单Reactor线程模式
- Acceptor: 注册一个Acceptor事件处理器，用于处理ACCEPT事件
- Worker Thread Pool: 添加一个工作者线程池，将非IO操作交给线程池处理

### 多Reactor线程模式
- 一个mainReactor处理ACCEPT事件
- 多个subReactor处理其他事件

### Netty实现多Reactor线程模式
|Reactor|Netty|
|------|------|
|Initiation Dispatcher|NioEventLoop|
|Synchronous Event Demultiplexer|Selector|
|Event Handler|ChannelHandler|
|mainReactor|bossGroup中的某个NioEventLoop|
|subReactor|workerGroup中的某个NioEventLoop|
|acceptor|ServerBootstrapAcceptor|
|Thread pool|用户自定义的线程池|

### Proactor模式（貌似有两种方案？）
- 通过异步操作来非阻塞的拷贝数据
- Proactor: 调用Asynchronous Operation Processor
- Asynchronous Operation Processor: 调用Asynchronous Operation发起异步IO，并获取IO完成通知
- Asynchronous Operation: 异步操作
- Completion Handler: 回调
- Handle: 资源


