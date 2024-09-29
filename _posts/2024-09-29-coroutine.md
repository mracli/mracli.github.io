---
title: 从异步回调到C++20中的协程
description: 虽然C++20有了协程基本件，但还需要自己装修
author: momo
categories:
- C++
- 协程
tags:
- C++
- 协程
layout: post
pin: false
math: true
date: 2024-09-29 15:43 +0800
---
## 协程

解释协程 `coroutine` 前需要了解例程 `routine`, 即一系列连续的操作, 例程执行可构成调用关系，被调用例程总是在调用例程之前返回，且在单一执行流(单线程环境下)中**例程不可由用户主动暂停**。

当然实际情况下函数与例程的表现是等价的，后面我们都以例程表示。

协程则是例程的推广形式，可以通过**保留执行状态**(现在我们都称其为协程栈)**实现手动暂停和恢复进度**，提供更强的灵活性。与线程相比，使用协程在io密集型任务重可有效降低资源调度成本，并减少内核态-用户态切换的开销。

### 协程特性

- 连续多次调用协程，仍然保持局部变量值
- 可暂停可恢复
- 支持对称和非对称的控制流切换
- 是一种 `一等对象`[^firstclassobject] `first-class object`, 也即可被当做一般的对象使用，同时无其他任何限制，在C++中 `lambda表达式`返回的对象 和 重载`()`操作符的对象可被认为是一种一等对象.
- 有栈(支持嵌套调用)或无栈(不支持，或者需要保存调用协程来实现嵌套调用)

### 应用-事件驱动模型

`事件驱动模型` 是一种编程范式，其程序流受事件影响，事件则来自多个无关事件源，在该模型中有一个事件分配器，等待外部的源在将来的任何时候触发回调函数，整个模型则可分为 `事件选择` 和 `事件处理`. 可以看到这样的结构组件之间是松耦合的，该模型的使用场景则包括: UI应用(用户点击和组件槽发送信号)、WebServer(外部用户通过网络请求本地文件)、游戏画面渲染(帧事件驱动)

![](assets/img/c++20coroutine/event-driven model.png)

为什么要使用这样的模型，不能使用线程实现吗？

- 多线程需要考虑更多的多线程安全问题
- 内存需求高
- 创建维护线程状态昂贵
- 线程上线文切换成本高

相对的使用协程则可以：

- 单一控制流，不需要考虑多线程安全问题
- 上下文切换不需要切换到内核态

### 异步实现

更早的时期，为了实现事件循环的异步操作，需要引入回调函数，并将执行控制流分为多个部分，按照整个控制流中所有可能引起例程挂起的 `关键点` 进行切割，举例：

```c++
/* 
  psuedo code
  This is a webserver, as a server end program, we should listen all 
  clients' requests, plus waiting for any un completed i/o.
 */
int sockfd = create_bind_listen_socket();


// hung up / blocked if no one request for anything from here
int connfd = accept(sockfd)
// generate a thread to process this connfd as below described: 
void thread_handle(int connfd):
    // repeat do read/write stuff until connection state timeout,
    // reset/closed by peer or closed by self
    while(!stop_token):
        // blocked if no content yet
        ssize_t recvsz = read(connfd)

        // blocked if send buffer full already
        ssize_t sendsz = write(connfd);
```

对于线程来说，执行上述的例程中的函数，一旦阻塞，则会被迫挂起，同时被操作系统调度，该过程上下文切换成本非常高，使用 `回调函数` 则把这样的执行例程拆分为多个部分：

```c++
/*                                                   
    wrapper for this routine, and read/write could be paused/reenterd,
    cuz we save all local variables in io_context

    +-------+         +--------+        +---------+           
    |       |         |        |        |         |           
    | start +----+--->|  read  +------->|  write  +--------+  
    |       |    |    |        |        |         |        |  
    +-------+    |    +--------+        +---------+        |  
                 |                                         |  
                 +------------------<---------------------+  
 */
struct session : std::enable_shared_from_this /* shared this session to callback pool but no need to leak address of io_context */ 
{
    // io_context means all fd resources here are non-block type
    session(io_context ctx) : socket_(ctx) {}

    operator socket() { return socket_; }

    void start() 
    {
        do_read();
    }

    void do_read() {
        // async read operation
        ssize_t res = nonblock_read();
        // check if read failed with the EAGAIN error, 
        // which means we should check later if this operation is done 
        // We can achieve this by epoll
        if(res){
            // only focus on normal EAGAIN situation, like this
            register_do_read_operation_later();
        }
        else do_write();
    }

    void do_write() {
        // async write operation
        ssize_t res = nonblock_write();
        // check if write failed with the EAGAIN error, 
        // which means we should check later if this operation is done 
        // We can achieve this by epoll
        if(res){
            // only focus on normal EAGAIN situation, like this
            register_do_write_operation_later();
        }
        this->do_read();
    }
    socket socket_;
    static size_t max_length = 1024;
    char data_[max_length];
}

```

可以看到，使用回调函数的方式实现异步，会导致原始例程需要被拆分为多个小的回调函数，并且这个过程中会引入新的作用域和错误回调问题，大幅降低代码可读性和编码稳定性，更重要的，会**提高我们的心智成本**。

在 boost 中则有提供类似于协程的异步实现，使得实际编码风格可以接近例程。

```c++
void session(boost::asio::io_service& io_service){
    // construct TCP-socket from io_service
    boost::asio::ip::tcp::socket socket(io_service);

    try{
        for(;;){
            // local data-buffer
            char data[max_length];

            boost::system::error_code ec;

            // read asynchronous data from socket
            // execution context will be suspended until
            // some bytes are read from socket
            std::size_t length=socket.async_read_some(
                    boost::asio::buffer(data),
                    boost::asio::yield[ec]);
            if (ec==boost::asio::error::eof)
                break; //connection closed cleanly by peer
            else if(ec)
                throw boost::system::system_error(ec); //some other error

            // write some bytes asynchronously
            boost::asio::async_write(
                    socket,
                    boost::asio::buffer(data,length),
                    boost::asio::yield[ec]);
            if (ec==boost::asio::error::eof)
                break; //connection closed cleanly by peer
            else if(ec)
                throw boost::system::system_error(ec); //some other error
        }
    } catch(std::exception const& e){
        std::cerr<<"Exception: "<<e.what()<<"\n";
    }
}
```


## C++20中的协程


在 C++20 新增三个关键字 `co_yield`, `co_eturn`, `co_await`, 当一个函数类似的代码块中有任意所述的关键字，那么就认为该代码块为协程，可在关键字处 `暂停` 并返回调用者的执行位置。但是，这些功能并未直接实现，C++20仅是暴露对应的接口和规范，我们仍然需要自行实现对应的接口或者使用第三方库实现。

### 协程额外的概念

一个协程总是带有: Awaiter(决定协程被挂起时的行为), Awaitable(可以提供Awaiter对象), Promise(定义协程在不同状态下的操作)，至于硬性实现，请参考cppref。

[协程支持](https://zh.cppreference.com/w/cpp/coroutine)

[std::coroutine_handle](https://zh.cppreference.com/w/cpp/coroutine/coroutine_handle)

### Awaiter

对于 Awaiter 对象来说，其成员函数应用时机为：

- await_ready()    调用co_await 时用于判断子协程是否就绪
- await_suspend()  await_ready()判断未就绪即将子协程挂起，会调用 await_suspend()
- await_resume()   co_await 返回结果时，会调用 await_resume()

![](assets/img/c++20coroutine/AwaitableObject.png)

### 等价操作

对于 `co_await` 来说，调用该方法等价于: `co_await promise().yield_value()`

## 参考
[^firstclassobject]: [一等对象](https://stackoverflow.com/questions/245192/what-are-first-class-objects)