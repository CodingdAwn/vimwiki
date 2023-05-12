# 协程相关

## 介绍一下SynchronizationContext的用法
ET框架中使用了SynchronizationContext，一开始只是模糊的知道他是用这个来做单线程多协程的，但不清楚原理

1. 首先SynchronizationContext是用来做线程间通信的，也就是通过这个东西，我可以让一个函数在指定的线程去执行
2. 默认SynchronizationContext.Current是null
3. 当SynchronizationContext.Current is null时，在await时，函数一般会在其他线程异步执行，这个时候会切换context
4. 执行完后回调回来，切换回原context

而ET重载了SynchronizationContext以及post，使得所有的await都回到了主线程的context中执行，从而确保单线程多协程

## Task.Run
Task.Run默认会新建线程执行，所以如果需要task在别的线程执行，需要手动设置一下SynchronizationContext.SetSynchronizationContext

## Thread.CurrentThread.Join
这个函数会阻塞当前线程，直到线程结束或者超时，FreeRedis中的分布式锁

## ref
[介绍SynchronizationContext](https://www.c-sharpcorner.com/article/understanding-synchronization-context-task-configureawait-in-action/)
