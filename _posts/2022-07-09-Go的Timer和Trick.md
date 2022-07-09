---
layout: post
author: pzc
title: Go的Timer和Trick
tags: [Go]
date: 2022-07-09 00:00 +0800
toc:  true
---

## NewTimer
{% highlight js %}
func readWithTimeout(r chan int) {
  // 通过NewTimer新建一个timer，在时间过去1秒后，仅触发一次
  timer := time.NewTimer(1 * time.Second)
  select {
 case d := <-r:
  fmt.Println(d)
 case <-timer.C:
  fmt.Println("timeout")
  }
}
{% endhighlight %}

直接开始分析源码（版本1.18.3）吧。  
NewTimer函数会调用startTimer函数，startTimer调用addtimer
{% highlight js %}
func addtimer(t *timer) {
 ...
 t.status = timerWaiting

 when := t.when

 // Disable preemption while using pp to avoid changing another P's heap.
 mp := acquirem()

 pp := getg().m.p.ptr()
 lock(&pp.timersLock)
 // 清理已被标记为删除的timer和调整被修改时间的timer。
 cleantimers(pp)
 // 将timer添加到四叉堆上，插入时间复杂度为O(log n / log 4)，删除时间复杂度为
 // O(4 * log n / log 4)，不仅插入时间复杂度比二叉低，而且删除时间复杂度是一样的。
 //（d叉堆的插入时间复杂度为O(log n / log d)，删除复杂度为O(d * log n / log d)）
 doaddtimer(pp, t)
 unlock(&pp.timersLock)

 wakeNetPoller(when)

 releasem(mp)
}

func doaddtimer(pp *p, t *timer) {
  ...
  t.pp.set(pp)
 i := len(pp.timers)
 pp.timers = append(pp.timers, t)
 siftupTimer(pp.timers, i) // 调整堆
 if t == pp.timers[0] {
  atomic.Store64(&pp.timer0When, uint64(t.when)) // 将最早过去的timer的when设置到timer0When
 }
  ...
}

{% endhighlight %}

## Stop timer
Stop的文档中提到，需要判断Stop的返回值，如果返回false的话，并且没有其他地方会从timer的channel接收时，需要排空timer的channel：
{% highlight js %}
if !t.Stop() {
  <-t.C
}

// 例如以下用法
func main() {
 t := time.NewTimer(time.Second * 3)
 foo := make(chan int)
 go func() { foo <- 1 }()
 select {
 case <-t.C:
  fmt.Println("timeout")
 case <-foo:
  fmt.Println("foo")
  if !t.Stop() {
   <-t.C
  }
 }
}
{% endhighlight %}
为什么需要排空呢？如果需要重置timer时，没有排空，会出问题。（感觉这样的设计不是太好。）  
Stop函数最终会调用deltimer，而deltimer只是将timer标记为删除的状态，并没有真正删除。
{% highlight js %}
func deltimer(t *timer) bool {
 for {
  switch s := atomic.Load(&t.status); s {
  case timerWaiting, timerModifiedLater:
   // Prevent preemption while the timer is in timerModifying.
   // This could lead to a self-deadlock. See #38070.
   mp := acquirem()
   if atomic.Cas(&t.status, s, timerModifying) {
    // Must fetch t.pp before changing status,
    // as cleantimers in another goroutine
    // can clear t.pp of a timerDeleted timer.
    tpp := t.pp.ptr()
        // 标记为被删除的状态
    if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
     badTimer()
    }
    releasem(mp)
    atomic.Xadd(&tpp.deletedTimers, 1)
    // Timer was not yet run.
    return true
   } else {
    releasem(mp)
   }
  ...
}
{% endhighlight %}
这样做的目的大概是减少临界区，降低锁冲突，提升性能。

## Reset timer
<mark>func (t *Timer) Reset(d Duration) bool </mark>
只能在以及过期的或者以及停止的timer上调用Reset，并且要确认channel以及被排空了。  
Reset最终会调用[modtimer](https://github.com/golang/go/blob/dev.boringcrypto.go1.18/src/runtime/time.go#L424){:target="_blank"}，逻辑如下：  
- 如果timer已经被从堆中移除了，则调用doaddtimer将timer添加到堆上。
- 否则，只是修改timer的状态为timerModifiedLater或者timerModifiedEarlier，这么做估计也是为了减少锁冲突。

## Tricker
Tricker跟Timer是区别在于Tricker设置了period字段，可以周期执行，其他逻辑是一样的。