+++
date = '2025-08-21T01:07:12+08:00'
draft = false
title = '零拷贝技术'
+++

## 什么是零拷贝？

利用直接内存访问（Direct Memory Access，DMA），消除数据在内存中不必要的拷贝，即零拷贝技术（Zero-Copy）。

零拷贝不是没有任何复制，而是消除了消耗CPU和内存带宽的不必要的冗余复制。



## 为什么要有零拷贝？

传统IO过程：

用户进程--`read()`--->CPU(转化为内核态)--请求读取--->磁盘--多次缓冲区拷贝后转化为用户态--->`read()`返回---->用户进程

CPU被大量文件数据占用，多次切换上下文，期间干不了其他事情。并且，多次缓冲区的拷贝极大限制了文件传输性能。

于是Zero-Copy出现了。



## 优化思路

- 减少上下文切换的次数。传统文件IO（`read()`+`write()`）有4次上下文切换（`read`用户->内核->用户，`write`用户->内核->用户）。

- 减少各种缓冲区的拷贝次数。传统读取：磁盘控制器缓冲区->内核缓冲区->用户缓冲区，每次拷贝都在消耗时间。



## `mmap()`+`write()`

`read()`会把内核缓冲区的数据拷贝到用户缓冲区，为了减少拷贝，换成`mmap()`。

`mmap`(Memory Map)核心思想是利用虚拟内存管理机制，在进程虚拟地址空间与物理资源（比如文件）之间建立直接的映射关系。

~~~cpp
#include <sys/mman.h>

void *mmap(void addr[.length],
           size_t length,
           int prot,
           int flags,
           int fd,
           off_t offset);
~~~

使用
~~~cpp
mmap(nullptr, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
~~~

把内核缓冲区的数据直接映射到用户缓冲区，这样就避免了一次拷贝。

然后`write()`，把数据直接拷贝到socket缓冲区。由于数据还在内核缓冲区，这些都在内核态发生。

不过系统调用上下文切换的次数没变。



## `sendfile()`

~~~cpp
#include <sys/socket.h>

ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
~~~

read from `in_fd`，然后write to `out_fd`。offset是`in_fd`内部的偏移量，count是最大处理长度。返回实际处理长度。

- 直接减少了一次系统调用，减少上下文切换开销
- 直接把数据从内核缓冲区拷贝到socket缓冲区，不再经过用户态

这时有磁盘->内核缓冲区->socket缓冲区->网卡，共3次拷贝（CPU一次,从内核读缓冲区到socket写缓冲区）。

如果网卡还支持SD-DMA技术，数据可以绕过sockert缓冲区，从内核缓冲区直接发送到网卡。内核空间缓冲区中对应的数据描述信息（内存地址、地址偏移量）记录到socket缓冲区中，由 DMA 根据内存地址、地址偏移量将数据批量地从内核缓冲区拷贝到网卡设备中（DMA gather copy），这样就又省去了内核空间中仅剩的 1 次 CPU 拷贝操作。



## `splice()`

`sendfile`的局限性是，只能从本地文件的fd发送到socket fd，且在一定程度上依赖硬件支持。

~~~cpp
ssize_t splice(int fd_in,
               off_t *_Nullable off_in,
               int fd_out,
               off_t *_Nullable off_out,
               size_t len,
               unsigned int flags);
~~~

在内核空间的读缓冲区与socket缓冲区之间建立管道，但使用时数据并不真正经过管道，只是用来管理数据流，避免了CPU拷贝操作。

~~~cpp
// 接收文件示例
int pipefd[2];
pipe(pipefd);
ssize_t bytes_read;
size_t total_bytes = 0;
...
while (1) {
    // 从socket接收
    bytes_read = splice(sock_fd, nullptr, pipefd[1], nullptr, 65536, SPLICE_F_MOVE);
    // 输出到本地文件
    auto bytes_written = splice(pipefd[0], nullptr, local_fd, nullptr, bytes_read, SPLICE_F_MOVE);
    if (...) { ... }
    total_bytes += bytes_written;
}
~~~

发送文件同理。

过程与`sendfile`类似，但是`splice`比较自由，可以用在任意的fd上，前提是有管道做媒介。



## Linux零拷贝对比

| 操作                         | CPU拷贝 | DMA拷贝 | 系统调用       | 上下文切换 |
| ---------------------------- | ------- | ------- | -------------- | ---------- |
| `read()`+`write()`           | 2       | 2       | `read`+`write` | 4          |
| `mmap()`+`write()`           | 1       | 2       | `mmap`+`write` | 4          |
| `sendfile()`                 | 1       | 2       | `sendfile`     | 2          |
| `sendfile()`+DMA gather copy | 0       | 2       | `sendfile`     | 2          |
| `splice()`                   | 0       | 2       | `splice`       | 2          |
