---
layout: post
author: pzc
title: Go的channel与select
tags: [Go]
date: 2022-07-04 15:32 +0800
toc:  true
---

## channel使用
- channel分为无缓冲和带缓冲两种，无缓冲的channel发送或者接收时会阻塞起来，等待出现接收者或者发送者。
- 带缓冲的channel在buffer未满时，发送不会被阻塞，buffer中存在数据时，接收不会被阻塞。  
- 如果channel为nil，发送或者接收的go协程都会被永久阻塞住。

## select使用
- 如果是空的<mark>select{}</mark>，go协程会被永久阻塞起来。
- 如果不带有default，会打乱case的顺序，随机选择一个能接收或者发送的case。如果所有case都发送不了数据或者没数据接收，将会阻塞起来。
- 如果带有default，在所有case都不能执行的情况下，选择default。

## channel原理
{% highlight js %}
type hchan struct {
  qcount   uint           // total data in the queue
  dataqsiz uint           // size of the circular queue
  buf      unsafe.Pointer // points to an array of dataqsiz elements
  elemsize uint16
  closed   uint32
  elemtype *_type // element type
  sendx    uint   // send index
  recvx    uint   // receive index
  recvq    waitq  // list of recv waiters
  sendq    waitq  // list of send waiters

  // lock protects all fields in hchan, as well as several
  // fields in sudogs blocked on this channel.
  //
  // Do not change another G's status while holding this lock
  // (in particular, do not ready a G), as this can deadlock
  // with stack shrinking.
  lock mutex
}
{% endhighlight %}
可以看到，channel内部是通过ringbuffer+mutex实现的，recvq是等待接收数据的协程队列，sendq是等待发送数据的协程队列。
有缓冲的channel跟无缓冲的channel相同与不同点：
- 向channel发送数据，如果是buffered channel，并且ringbuffer满了，发送者会阻塞起来，和unbuffered channel一样。不一样的地方在于当接收者出现时，如果是unbuffered cahnnel，则接收者直接从发送者那里拷贝数据。但如果是buffered cahnnel，则先冲ringbuffer中recvx处获取一个数据，再把第一个阻塞起来的发送者的数据拷贝到ringbuffer的recvx处。
- 向channel接收数据时，如果是buffered channel，并且ringbuffer空了，接收者会阻塞起来，和unbuffered channel一样。如果此时出现了发送者，都是直接从recvq取出第一个等待的协程，然后直接向接收方写数据。
  
## select与channel
具体实现可以看<mark>selectgo()</mark>函数。  
select内部会将case顺序随机打乱，按cahnnel的地址大小顺序逐一加锁（应该是防止死锁）,接着遍历乱序的case，逐一判断是否可读、可写或者channel是否被关闭了。如果全部都不满足，则将go协程插入recvq或者sendq，并阻塞起来。
{% highlight js %}
for _, casei := range pollorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c

    if casi >= nsends {
      sg = c.sendq.dequeue()
      if sg != nil {
        goto recv
      }
      if c.qcount > 0 {
        goto bufrecv
      }
      if c.closed != 0 {
        goto rclose
      }
    } else {
      if raceenabled {
        racereadpc(c.raceaddr(), casePC(casi), chansendpc)
      }
      if c.closed != 0 {
        goto sclose
      }
      sg = c.recvq.dequeue()
      if sg != nil {
        goto send
      }
      if c.qcount < c.dataqsiz {
        goto bufsend
      }
    }
  }

  for _, casei := range lockorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c
    sg := acquireSudog()
    sg.g = gp
    sg.isSelect = true
    // No stack splits between assigning elem and enqueuing
    // sg on gp.waiting where copystack can find it.
    sg.elem = cas.elem
    sg.releasetime = 0
    if t0 != 0 {
      sg.releasetime = -1
    }
    sg.c = c
    // Construct waiting list in lock order.
    *nextp = sg
    nextp = &sg.waitlink

    if casi < nsends {
      c.sendq.enqueue(sg)
    } else {
      c.recvq.enqueue(sg)
    }
  }
  {% endhighlight %}