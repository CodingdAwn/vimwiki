## 总结一下Task.Delay的流程
首先有一个普遍的问题
Thread.Sleep和Task.Delay的区别在哪呢，Sleep实际上是会阻塞当前线程的，而Task.Delay都是配合task await的，所有是异步的

## 流程
对于await生成的一些代码，就不细讨论了
0. 代码位置:
```
libraries/System.Private.CoreLib/src/System/Threading/Task/Task.cs
```

1. 暴露的接口
```
/// 首先用户调用的接口
public static Task Delay(int millisecondsDelay) => Delay(millisecondsDelay, default);

/// 实际调用 可以看到实际上调用的是DelayPromise
private static Task Delay(uint millisecondsDelay, TimeProvider timeProvider, CancellationToken cancellationToken) =>
    cancellationToken.IsCancellationRequested ? FromCanceled(cancellationToken) :
    millisecondsDelay == 0 ? CompletedTask :
    cancellationToken.CanBeCanceled ? new DelayPromiseWithCancellation(millisecondsDelay, timeProvider, cancellationToken) :
    new DelayPromise(millisecondsDelay, timeProvider);
```

2. 实际上使用的是DelayPromise
```
    private class DelayPromise : Task
            
    /// 实际上逻辑 很少 很容易明白
    /// 实际上就是创建了一个Timer,这个TimerQueueTimer在读Timer的相关源码时介绍过了，可以翻一番Timer的实现
    /// 这里创建就会把timer放到队列中了
    internal DelayPromise(uint millisecondsDelay, TimeProvider timeProvider)
    {
        if (millisecondsDelay != Timeout.UnsignedInfinite) // no need to create the timer if it's an infinite timeout
        {
            if (timeProvider == TimeProvider.System)
            {
                _timer = new TimerQueueTimer(s_timerCallback, this, millisecondsDelay, Timeout.UnsignedInfinite, flowExecutionContext: false);
            }
            else
            {
                using (ExecutionContext.SuppressFlow())
                {
                    _timer = timeProvider.CreateTimer(s_timerCallback, this, TimeSpan.FromMilliseconds(millisecondsDelay), Timeout.InfiniteTimeSpan);
                }
            }

            if (IsCompleted)
            {
                // Handle rare race condition where the timer fires prior to our having stored it into the field, in which case
                // the timer won't have been cleaned up appropriately.  This call to close might race with the Cleanup call to Close,
                // but Close is thread-safe and will be a nop if it's already been closed.
                _timer.Dispose();
            }
        }
    }
```

3. Timer结束
```
    /// TrySetResult实际上就是调用Task的接口，将Task完成
    private void CompleteTimedOut()
       {
           if (TrySetResult())
           {
               Cleanup();
           }
       }
```

4. Task TrySetResult
```
    /// 重点是调用了FinishContinuations，完成所有的Continuations的任务
    internal bool TrySetResult()
    {
        if (AtomicStateUpdate(
            (int)TaskStateFlags.CompletionReserved | (int)TaskStateFlags.RanToCompletion,
            (int)TaskStateFlags.CompletionReserved | (int)TaskStateFlags.RanToCompletion | (int)TaskStateFlags.Faulted | (int)TaskStateFlags.Canceled))
        {
            ContingentProperties? props = m_contingentProperties;
            if (props != null)
            {
                NotifyParentIfPotentiallyAttachedTask();
                props.SetCompleted();
            }
            FinishContinuations();
        }
    }
```

5. 而实际上的continue的任务就是在 async生成的state machine的代码，即move next，所以后面就会调回到await之后的代码，而这中间当前还有一些流程
比如还会根据sync context来决定在线程执行等等，也可能放回到thread pool去执行，这就属于整个异步体系的逻辑了
