---
layout: post
author: pzc
title: Linux的select、poll、epoll
tags: [IO多路复用, epoll]
date: 2021-02-05 00:00 +0800
toc:  true
---

## select
优点：
- 可以IO多路复用，不用每个新的连接都起一个线程
缺点：
- 一般最多支持1024个文件描述符，编译内核时就写死了。
- 性能较差，每次陷入内核时，都有复制一次fd_set。需要遍历fd_set的每一位，不管他们是否就绪。

## poll
<mark>int poll(struct pollfd *fds, nfds_t nfds, int timeout);</mark>  
poll是以数组方式传递文件描述符给内核的，内核会转为链表方式存储，对数量没有多大限制。其他的跟poll差不多

## epoll
epoll是对select和poll的改进：
- 内部使用红黑树管理文件描述符，使用队列储存就绪文件描述符
- 每个文件描述符只需要添加一次，后续可以通过修改监听的事件来读写
- 为每个文件描述符添加回调函数，就绪时调用回调函数自动添加到就绪队列上  

相关函数有三个：epoll_create、epoll_ctl和epoll_wait。 
 
水平触发（LT）：当有数据可读或者缓冲区可写时 
边缘触发（ET）：（读/写）
- 当缓冲区由不可读变成可读时；当缓冲区由不可写变成可写时
- 新数据到来时；数据发送出去时
- 进行EPOLL_CTL_MOD 修改EPOLLIN事件时；进行EPOLL_CTL_MOD 修改EPOLLOUT事件时