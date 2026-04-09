+++
date = '2025-08-21T01:07:55+08:00'
draft = false
title = 'Linux C/C++ IO多路复用'

+++

在linux服务器编写中，为每个连接单独开一个进程/线程不现实，

我们采用io多路复用。

## 基本TCP socket模型

## select/poll

select内部有一个定长(1024)的bitsmap，以存储要监听特定事件的fd。fd从用户空间拷贝到内核，内核里遍历修改一遍后，再把就绪的fd拷贝到用户空间，用户再遍历一遍。

poll不再用bitsmap，改用动态数组(链表)，不再受限于特定个数的fd。

但二者其实区别不大，本质都是遍历了两遍，查找有事件的fd时间复杂度为`O(n)`，也都需要来回拷贝fd集合。这些操作都很消耗性能。


## epoll

~~~c
int sockfd = socket(AF_NET, SOCK_STREAM, 0);
bind( ... );
listen( ... );

int epoll_fd = epoll_create1(0);
// 添加fd
epoll_ctl(epoll_fd, ...);

while (1) {
    int n = epoll_wait( ... );
    for ( ... ) {
        // 有事件的fd
    }
}
~~~

- epoll不再使用线性的存储结构，改用红黑树（一种自平衡二叉树），增删改的时间复杂度一般为`O(log n)`。

- epoll内部维护了一个就绪fd链表，每次`epoll_wait`只需要处理链表，然后把所有就绪fd拷贝到用户空间。

- epoll不适用轮询，而是事件驱动。


epoll有两种触发模式，**ET**（边缘触发，select/poll不支持）和**LT**（水平触发），区别主要体现在读上。

LT模式下，只要可读（socket读缓冲区中有数据），就会触发事件，直到缓冲区被读完。

ET模式下，仅当数据到来时（缓冲区数据从无到有）触发一次事件，后续将不再触发，即使缓冲区还有数据。

LT一般搭配阻塞读写，ET一般搭配非阻塞读写。
