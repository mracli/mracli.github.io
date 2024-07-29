---
title: 套接字Socket
description: Linux网络编程基础
author: momo
categories:
- Linux
tags:
- 笔记
layout: post
pin: false
math: true
date: 2024-05-29 12:23 +0800
---
## 定义

`Socket` 指二元组 $(ip, port)$, 即Socket基础中的Socket地址，所有的接口定义都在 `<sys/socket.h>` 头文件中，该API可实现功能：
: 创建，命名和监听socket，读取和设置socket选项；以及接受，发起连接，读写数据，获取地址信息和检测带外标记，

`Socket` 有三种地址类型 `sockaddr` ：Unix(`sockaddr_un`), TCP/IPv4(`sockaddr_in`), TCP/IPv6(`sockaddr_in6`)

在socket地址中，所有的ip地址都保存为对应的 `网络字节序` ，一般在记录日志时才会将ip地址转换为对应的点分十进制字符串表示[^pt_presentation]。在 `<arpa/inet.h>` 保存网络字节序地址和点分十进制字符串的转换函数，需要注意的是其中 `inet_ntoa` 函数返回结果为静态地址，不能保证线程安全（不可重入）。

## Socket使用

### 创建

引入 `<sys/socket.h>` 后，可使用 `socket()` 函数创建，对应的参数和返回类型查询手册可得：
: `int socket(int domain, int type, int protocol);`

`domain` 表示对应的协议族，`type` 表示使用的传输层协议，`protocol` 不使用，直接设置为0.

### 命名

将socket与socket地址绑定的操作称为 `给socket命名`。一般在服务端中需要命名socket，而在客户端中只需要匿名的方式获得系统自动分配的socket地址即可。命名则使用 `bind()` 函数实现，定义为：
: `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

### 监听

服务端命名socket后，并不能立即接收客户端连接，需要使用系统调用 `listen()` 创建一个监听队列来存放待处理的客户端连接，这个队列我们也称作全连接队列，特指已经完成三次握手后的连接，同时该文件描述符对应的可读事件绑定为全连接队列内容非空，这样我们可以通过系统调用 select, poll 或者 epoll 监听该 fd 对应的 EPOLLIN 实现监听是否有客户端连接。
: `int listen(int sockfd, int backlog)`

`sockfd` 指定被监听的socket，而 `backlog` 提示内核监听队列的最大长度，如果超过该长度 (backlog指定的长度) 服务端则不会受理新的客户端连接，同时客户端会受到拒绝信息 `ECONNREFUSED` 

### 接受连接

`accept()` 系统调用可从监听队列中接受一个连接：
: `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`

`sockfd` 是指监听socket，成功时 `accept()` 返回连接的socket

### 发起连接

客户端通过系统调用 `connect()` 与服务端建立连接：
: `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

### 关闭连接

关闭连接指关闭该连接对应的socket，通过系统调用 `close()` 完成：
: `int close(int fd);`

`close()` 调用减少socket的引用计数，当socket对应的引用计数为0时该连接关闭。

如果需要事实上关闭连接，则通过系统调用 `shutdown()` 完成：
: `int shutdown(int sockfd, int howto);`

其中 `howto` 参数决定shutdown的行为，分为写关闭，读关闭和完全关闭。写关闭控制进程不可再向sockfd写操作，同时会在关闭连接前将发送缓冲区的数据全部发送；读关闭则会丢弃接收缓冲区中的所有数据。

### 数据读写

根据传输层协议分别提供专用的系统调用，对于TCP流，读 `recv()` 写 `send()`, 对于UDP流，读 `recvfrom()` 写 `sendto()`.

其中TCP部分：
: `ssize_t recv(int sockfd, void *buf, size_t len, int flags);`
: `ssize_t send(int sockfd, const void *buf, size_t len, int flags);`

两者都指定了需要读取或写入的sockfd，buf指读或写缓冲区，len为缓冲区大小，flags为标识符，表示额外的控制，并且两者都返回实际读和写的大小。

对于UDP部分：
: `ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);`
: `ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);`

相比TCP，UDP是无连接的通信方式，所以每次发送或接收都需要指定对应的socket地址，可以看参数，对应的有 `*_addr` 以及 `addrlen`.

当UDP部分的数据包socket执行connect后，可不指定 `socket_addr` 和 `addrlen`, 但传输层仍然会使用UDP的传输方式，只是自动的从sockfd中获得对应的socket地址和长度。

同时可以使用通用的系统调用 `sendmsg()` 和 `recvmsg()`. 该调用可以同时使用TCP流数据和UDP数据包。
: `ssize_t recvmsg(int sockfd, struct msghdr* msg, int flags);`
: `ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);`

其中 `struct msghdr` 结构体如下：

```c
struct msghdr {
   void         *msg_name;       /* Optional address */
   socklen_t     msg_namelen;    /* Size of address */
   struct iovec *msg_iov;        /* Scatter/gather array */
   size_t        msg_iovlen;     /* # elements in msg_iov */
   void         *msg_control;    /* Ancillary data, see below */
   size_t        msg_controllen; /* Ancillary data buffer len */
   int           msg_flags;      /* Flags (unused) */
};
```

其中 `msg_name` 为socket地址，当操作的是TCP流数据时，不需要指定该变量，之间传入 `NULL` 即可，对于UDP数据包则需要指定对应的socket地址。

### 带外数据

`带外数据` 又称为 `加速数据`，为紧急数据，可以通过系统调用 `sockatmark()` 判断 sockfd 是否处于带外标记，下一次读取到的数据是否为带外数据。如果是，则可以通过 `MSG_OOB` 标志的 `recv` 系统调用来接收带外数据。

### 地址信息函数

可以根据sockfd获取对应的本端地址以及远端地址：

本端地址
: `int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`
: `int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`

需要注意的是，该系统调用成功时返回 `0` 失败返回 `-1`.

### socket描述符设置

利用系统调用获取或设置描述符设置：
: `int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);`
: `int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);`

需要注意的是，该系统调用成功时返回 `0` 失败返回 `-1`.

### 网络信息获取

即获取socket地址中的IP地址和端口号，分别代表对应的主机名和服务。

#### 获取主机信息
其中获取主机名：
: `struct hostent *gethostbyname(const char *name);` 通过 `<netdb.h>`获取
: `struct hostent *gethostbyaddr(const void *addr, socklen_t len, int type);` 通过`<sys/socket.h>`获取

可分别根据主机名或IP地址来获得完整的主机网络信息 `hostent`，其结构包括主机名以及IP地址信息：

```c
struct hostent {
   char  *h_name;            /* official name of host */
   char **h_aliases;         /* alias list */
   int    h_addrtype;        /* host address type */
   int    h_length;          /* length of address */
   char **h_addr_list;       /* list of addresses */
}
```

#### 获取服务信息

获取服务信息，也即端口号和服务名，通过 `<netdb.h>` 获取：
: `struct servent *getservbyname(const char *name, const char *proto);`
: `struct servent *getservbyport(int port, const char *proto);`

可分别从服务名和端口号获得完整的服务信息 `servent`，其结构包括服务名和端口号：

```c
struct servent {
   char  *s_name;       /* official service name */
   char **s_aliases;    /* alias list */
   int    s_port;       /* port number */
   char  *s_proto;      /* protocol to use */
}
```

然后可已通过主机名和服务名获得对应地址信息：
: `int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res);`
: `void freeaddrinfo(struct addrinfo *res);`

需要注意，该系统调用会动态分配存储来存放结果，所以使用结束后需要手动释放结果。

```c
struct addrinfo {
   int              ai_flags;
   int              ai_family;
   int              ai_socktype;
   int              ai_protocol;
   socklen_t        ai_addrlen;
   struct sockaddr *ai_addr;
   char            *ai_canonname;
   struct addrinfo *ai_next;
};
```

#### 获取主机和服务信息

估计传入的socket地址，返回该socket地址对应的主机和服务信息：
: `int getnameinfo(const struct sockaddr *addr, socklen_t addrlen, char *host, socklen_t hostlen, char *serv, socklen_t servlen, int flags);`

### 一个包含简单的服务端与客户端连接流程

![](assets/img/LinuxSocket/tcp_socket.png){: .light .w-75 .shadow .rounded-10 w='1212' h='668' }

![](assets/img/LinuxSocket/tcp_socket_dark.png){: .dark .w-75 .shadow .rounded-10 w='1212' h='668' }

## Socket IO通信

### 管道创建

Linux 提供系统调用用于创建管道描述符，可以实现不同进程之间的**单向通信**：
: `int pipe(int pipefd[2])`，该系统调用来自 `<unistd.h>`.

其中pipefd[0] 描述符用于从对应文件读；pipefd[1] 用于向对应文件写。

同时 socket 提供了双向通信的管道：
: `int socketpair(int domain, int type, int protocol, int sv[2])`，该系统调用来自 `<sys/socket.h>`

但是该函数接口只能够接收 `domain` 为 AF_UNIX 的协议簇，意味只接收本机多个进程之间的通信。

### 复制描述符

如果需要重定向标准输入输出流，可以通过复制描述符实现：
: `int dup(int oldfd);` 创建一个尽可能小的描述符绑定至 `oldfd` 指向的文件
: `int dup2(int oldfd, int newfd);` 创建一个不小于 `newfd` 的描述符指向 `oldfd` 指向的文件

需要注意的是，复制描述符并不会继承原有描述符的属性。

### 文件描述符读写

: `ssize_t readv(int fd, const struct iovec *iov, int iovcnt)`
: `ssize_t writev(int fd, const struct iovec *iov, int iovcnt)`

该函数对来自 `<sys/uio.h>` 成功则返回对应的读/写字节数，失败则返回 `-1`并设置 `errno` 

### 文件传递

该方法称为零拷贝函数，不需要经过用户缓冲区，完全在内核中执行，实现从`文件`到`套接字`的传递：
: `ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count)`

该方法从 `in_fd` 所指文件至 `out_fd` 所指套接字的 `offset` 位置写入 `count` 字节。如果成功则返回实际传送的字节数量，否则返回 `-1` 同时设置 `errno`

### 内存申请

该函数可以申请一段内存空间，可用于作为进程间通信的共享内存，或者将文件映射其中：
: `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)`;
: `int munmap(void *addr, size_t length)`;

`addr` 指用户指定的内存起始地址，可以传递 `NULL` 不指定，则由内核决定，申请 `length`字节的空间，并根据 `prot` 决定该内存空间的该段空间被修改后程序的行为，包括该段内存空间是否共享，是否文件映射而来等；指定 `fd` 所指文件 `offset` 位置处开始映射内容，该函数成功则返回对应的内存地址，否则返回 `-1` 并设置 `errno`.

### 文件描述符之间传递数据

要求至少有一个管道文件描述符
: `ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out, size_t len, unsigned int flags);`

### 管道文件描述符之间传递数据

要求两个都是管道文件描述符：
: `ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags)`

### 修改文件描述符的属性

从库 `<fcntl.h>`中获取该函数，可用于控制文件描述符常见的属性：
: `int fcntl(int fd, int cmd, ...)`

第三个是可选参数，根据 `cmd` 决定实际是否需要以及需要的类型。

## 注释
[^pt_presentation]: 表示为形如："XX.XX.XX.XX" 或 "XX:XX:XX:XX:XX:XX" 的地址表示方法
