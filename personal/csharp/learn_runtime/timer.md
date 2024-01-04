# dotnet runtime Timer
timer的代码涉及线程相关，EventWaitHandle等待的实现，这些可能并不会详细介绍

## Timer模块
Timer有两个模块
* System.Timers 这个是暴露给用户使用的 源码在[libraries/System.ComponentModel.TypeConverter/src/System/Timers]
* System.Threading 真正的Timer的实现

## Timer的基本代码流程
注意所有的代码引用可能都有删减，只保留了一些流程代码.
1. 首先看一下官方的例子中 Timer的使用方式
```      
    Timer timer = new Timer(1000);
    timer.Elapsed += async ( sender, e ) => await HandleTimer();
    timer.Start();
    
    // 另外一个例子
    aTimer = new System.Timers.Timer(2000);
    aTimer.Elapsed += OnTimedEvent;
    aTimer.AutoReset = true;
    aTimer.Enabled = true;
    
    -- 实际上Enable设置的设置的时候调用的是
    if (_timer == null)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        int i = (int)Math.Ceiling(_interval);
        _cookie = new object();
        _timer = new Threading.Timer(_callback, _cookie, Timeout.Infinite, Timeout.Infinite);
        _timer.Change(i, _autoReset ? i : Timeout.Infinite);
        --<<< 这里_timer就是Threading中的Timer
    }
    
    -- 而Start实际上是 调用的
    public void Start() => Enabled = true;
```

2. Threading.Timer中的调用
``` 
    internal bool Change(uint dueTime, uint period)
    {
        lock (_associatedTimerQueue)
        {
            if (dueTime == Timeout.UnsignedInfinite)
                _associatedTimerQueue.DeleteTimer(this);
            else
                success = _associatedTimerQueue.UpdateTimer(this, dueTime, period);
        }
        return success;
    }
    
    --<<< 这里解释下 _associatedTimerQueue, 它是TimerQueue，一开始就根据cpu的核数进行了初始化
    --<<< 而在新加一个timer的实收，会根据线程id进行如下分配
    --<<< 所以不同线程是在固定的TimerQueue中添加timer，而TimerQueue中timer也可能来自不同线程
    _associatedTimerQueue = TimerQueue.Instances[(uint)Thread.GetCurrentProcessorId() % TimerQueue.Instances.Length];
``` 

3. 由于timer本身是一个双向的列表，所以这里主要是做插入删除的操作，然后调用EnsureTimerFiresBy，
LinkTimer也会决定将timer放到long timer list中还是short timer list，实际上没有list，习惯这么表达了，timer本身就是双向的
``` 
    public bool UpdateTimer(TimerQueueTimer timer, uint dueTime, uint period)
    {
        long absoluteDueTime = nowTicks + dueTime;
        bool shouldBeShort = _currentAbsoluteThreshold - absoluteDueTime >= 0;
        LinkTimer(timer);

        timer._dueTime = dueTime;
        timer._period = (period == 0) ? Timeout.UnsignedInfinite : period;
        timer._startTicks = nowTicks;
        return EnsureTimerFiresBy(dueTime);
    }
``` 

4. 并不是所有的timer进来都会触发SetTimer，但是肯定会调用LinkTimer加入到timer的列表中
``` 
    private bool EnsureTimerFiresBy(uint requestedDuration)
    {
        if (_isTimerScheduled)
        {
            ---<<< 而后面这个TimerQueue再次加入timer时，因为已经加入到列表中了
            ---<<< 1. 如果之前的设置的时间已经到触发时间了，那么久不管了，因为这个队列已经加入到了s_scheduledTimers中了
            ---<<< 所以等待那边TimerThread去更新就好了，因为那么处理的单位就是TimerQueue
            ---<<< 2. 如果新加的timer触发的时间比之前缓存要长，那么也不用SetTimer，等那边先把最近的触发了再说
            long elapsed = TickCount64 - _currentTimerStartTicks;
            if (elapsed >= _currentTimerDuration)
                return true; // the timer's about to fire

            uint remainingDuration = _currentTimerDuration - (uint)elapsed;
            if (actualDuration >= remainingDuration)
                return true; // the timer will fire earlier than this request
        }
    
        if (SetTimer(actualDuration))
        {
            ---<<< 这个队列第一个timer触发后 _isTimerScheduled，并且记录起始以及要经过的时间
            _isTimerScheduled = true;
            _currentTimerStartTicks = TickCount64;
            _currentTimerDuration = actualDuration;
            return true;
        }

        return false;
    }
``` 

5. 这里SetTimer会根据os来调用了，但最终都会调用Protable中的SetTimerPortable

s_scheduledTimers这是一个static变量，这里加入的TimerQueue而不是timer，之后会在timer线程去做处理，线程中处理的单位也是TimerQueue

InitializeScheduledTimerManager_Locked这个会在第一次调用时，创建新的s_scheduledTimers

到这里同步调用的代码就完事了，timer最终被加到了s_scheduledTimers
``` 
    private bool SetTimerPortable(uint actualDuration)
    {
        long dueTimeMs = TickCount64 + (int)actualDuration;
        AutoResetEvent timerEvent = s_timerEvent;
        lock (timerEvent)
        {
            if (!_isScheduled)
            {
                List<TimerQueue> timers = s_scheduledTimers ?? InitializeScheduledTimerManager_Locked();

                timers.Add(this);
                _isScheduled = true;
            }
            ---<<< 这里如果已经scheduled，那么新的timer只会使这里更新一下_scheduledDueTimeMs的时间
            _scheduledDueTimeMs = dueTimeMs;
        }
        ---<<< FIXME 感觉这里只是重新通知了一下wait的模块，看一下
        timerEvent.Set();
        return true;
    }
```

6. Timer线程的创建, 在执行InitializeScheduledTimerManager_Locked初始化的时候，会创建timer的线程
```
    Thread timerThread = new Thread(TimerThread)
    {
        Name = ".NET Timer",
        IsBackground = true
    };
    timerThread.UnsafeStart();
```           

7. Timer线程逻辑
```           
    private static void TimerThread()
    {
        while (true)
        {
            ----<<< WaitOne涉及到了WaitSystem的东西，简单看了一下基本上就是block线程等待了
            ----<<< 因为这里是Timer单独的线程，所以等待没有关系，因为等的是最近的那个timer
            ----<<< while第一次shortestWaitDurationMs是Timeout.Infinite(-1)
            timerEvent.WaitOne(shortestWaitDurationMs);
            lock (timerEvent)
            {
                ---<<< 处理所有的TimerQueue，如果有能fire就fire，没到fire时间的，计算最近的fire时间shortestWaitDurationMs，然后再次等待
                for (int i = timers.Count - 1; i >= 0; --i)
                {
                    TimerQueue timer = timers[i];
                    long waitDurationMs = timer._scheduledDueTimeMs - currentTimeMs;
                    if (waitDurationMs <= 0)
                    {
                        ---<<< 这里scheduled也设置为false了，说明这个TimerQueue已经fire过了
                        timer._isScheduled = false;
                        timersToFire.Add(timer);

                        int lastIndex = timers.Count - 1;
                        if (i != lastIndex)
                        {
                            timers[i] = timers[lastIndex];
                        }
                        timers.RemoveAt(lastIndex);
                        continue;
                    }
                }
            }

            ---<<< 这里直接放到线程池里了，但是这里是HighPriority
            foreach (TimerQueue timerToFire in timersToFire)
                ThreadPool.UnsafeQueueHighPriorityWorkItemInternal(timerToFire);
        }
    }
```           

8. 线程池有一个queue，执行到timer后会调用
```
    void IThreadPoolWorkItem.Execute() => FireNextTimers();
```

9. 调用timer的回调，遍历所有这个TimerQueue的timer，看看哪些可以fire
这里的代码很长
```
    private void FireNextTimers()
    {
        lock (this)
        {
            long nowTicks = TickCount64;
            TimerQueueTimer? timer = _shortTimers;
            for (int listNum = 0; listNum < 2; listNum++) // short == 0, long == 1
            {
                ---<<<
                while (timer != null)
                {
                    TimerQueueTimer? next = timer._next;

                    long elapsed = nowTicks - timer._startTicks;
                    long remaining = timer._dueTime - elapsed;
                    ---<<< 到时间了
                    if (remaining <= 0)
                    {
                        ---<<< 如果是重复的timer，还需要再把TimerQueue放回到s_scheduledTimers中去
                        if (timer._period != Timeout.UnsignedInfinite)
                        {
                            if (timer._dueTime < nextTimerDuration)
                            {
                                haveTimerToSchedule = true;
                                nextTimerDuration = timer._dueTime;
                            }

                            bool targetShortList = (nowTicks + timer._dueTime) - _currentAbsoluteThreshold <= 0;
                            if (timer._short != targetShortList)
                            {
                                MoveTimerToCorrectList(timer, targetShortList);
                            }
                        }
                        else
                        {
                            // Not repeating; remove it from the queue
                            DeleteTimer(timer);
                        }
                    
                        if (timerToFireOnThisThread == null)
                            timerToFireOnThisThread = timer;
                        else
                            ThreadPool.UnsafeQueueUserWorkItemInternal(timer, preferLocal: false);
                    }
                    else
                    {
                        ---<<< 有些没到时间的timer，计算最近的一个下次触发时间，到时候把这个TimerQueue放回s_scheduledTimers
                        ---<<< 如果是长列表还会检测下是否需要转到短列表中去
                        if (remaining < nextTimerDuration)
                        {
                            haveTimerToSchedule = true;
                            nextTimerDuration = (uint)remaining;
                        }
                    }
                    timer = next;
                }

                --<<< 切换成长列表
                if (listNum == 0)
                {
                    long remaining = _currentAbsoluteThreshold - nowTicks;
                    if (remaining > 0)
                    {
                        if (_shortTimers == null && _longTimers != null)
                        {
                            nextTimerDuration = (uint)remaining + 1;
                            haveTimerToSchedule = true;
                        }
                        break;
                    }

                    timer = _longTimers;
                    _currentAbsoluteThreshold = nowTicks + ShortTimersThresholdMilliseconds;
                }
            }

            ---<<< 1. 如果timer是重复执行的 2. 还有没有触发的timers
            ---<<< 那么fire完还需要再次从第4步开始
            if (haveTimerToSchedule)
            {
                EnsureTimerFiresBy(nextTimerDuration);
            }
        }

        // 第一个timer需要在当前的线程执行，后面所有timer都需要放到线程池中执行，这个具体为啥并没有想明白，是因为当前线程执行执行一次，剩下的就需要走队列了么，有可能
        timerToFireOnThisThread?.Fire();
    }
```

10. Fire Fire Fire 
```
    void IThreadPoolWorkItem.Execute() => Fire(isThreadPool: true);
    internal void Fire(bool isThreadPool = false)
    {
        CallCallback(isThreadPool);

        bool shouldSignal;
        lock (_associatedTimerQueue)
        {
            _callbacksRunning--;
            shouldSignal = _canceled && _callbacksRunning == 0 && _notifyWhenNoCallbacksRunning != null;
        }

        if (shouldSignal)
            SignalNoCallbacksRunning();
    }
```

11. 终于结束了，调回调了，可能从当前线程调用，也可能放到线程池中调用
```
    internal void CallCallback(bool isThreadPool)
    {
        // Call directly if EC flow is suppressed
        ExecutionContext? context = _executionContext;
        if (context == null)
            _timerCallback(_state);
        else
        {
            if (isThreadPool)
            {
                ExecutionContext.RunFromThreadPoolDispatchLoop(Thread.CurrentThread, context, s_callCallbackInContext, this);
            }
            else
            {
                ExecutionContext.RunInternal(context, s_callCallbackInContext, this);
            }
        }
    }
```
## FAQ
* WaitOne(-1)会怎么样，WaitOne是否肯定就是block的呢，Set函数作用
    --> 下面总结了
* 这里理解错了，不是timer加进来，而是TimerQueue，所以这里还需要再看一下 
    --> 如果操作timer的话太细了，timer thread效率也更低，现在这样其实就是处理一个timerqueue，然后只看其最近的那个，触发了之后，再从头来
* shortestWaitDurationMs关注一下这个到底怎么算的，不知道怎么保证等待最短的
    --> 就是在Timer thread中，每次while循环都是处理所有的TimerQueue，能fire就fire，没到时间的就计算最近时间
* 这里lock住了，那这个时候就不能再添加新的timer了？就算没锁住添加，那等待如果是block的，新的timer很近怎么办？
    -->  详细看代码，新的timer并不会加到线程这处理，除非新的timer的触发时间比之前的更短，那么只会更新scheduledDueTimeMs
    
## 关于EventWaitHandle的处理
1. s_timerEvent是一个静态变量，initialState=false，EventResetMode.AutoReset
AutoResetEvent ---<<<(继承自) EventWaitHandle ---<<< WaitHandle
```
    private static readonly AutoResetEvent s_timerEvent = new AutoResetEvent(false);
```

2. EventWaitHandle的构造函数，调用CreateEventCore创建event
```
    public EventWaitHandle(bool initialState, EventResetMode mode, string? name, out bool createdNew)
    {
        if (mode != EventResetMode.AutoReset && mode != EventResetMode.ManualReset)
            throw new ArgumentException(SR.Argument_InvalidFlag, nameof(mode));

        CreateEventCore(initialState, mode, name, out createdNew);
    }
```

3. CreateEventCore是根据系统有不同的处理，这里看一下unix的实现流程
```
    ---<<< EventWaitHandle.Unix.cs
    private void CreateEventCore(bool initialState, EventResetMode mode, string? name, out bool createdNew)
    {
        SafeWaitHandle = WaitSubsystem.NewEvent(initialState, mode);
    }
```

4. WaitSubSystem中的实现，官方的说法Design goals: Behave similarly to wait operations on Windows
NewEvent在WaitSubsystem.Unix.cs中
```
    internal static partial class WaitSubsystem
    
    public static SafeWaitHandle NewEvent(bool initiallySignaled, EventResetMode resetMode)
    {
        return NewHandle(WaitableObject.NewEvent(initiallySignaled, resetMode));
    }
```

5. NewHandle 不去细看了，其实就是一些初始化，因为封装了很多层
```
    private static SafeWaitHandle NewHandle(WaitableObject waitableObject)
    {
        var safeWaitHandle = new SafeWaitHandle();
        IntPtr handle = IntPtr.Zero;
        try
        {
            handle = HandleManager.NewHandle(waitableObject);
        }
        finally
        {
            if (handle == IntPtr.Zero)
            {
                waitableObject.OnDeleteHandle();
            }
        }

        Marshal.InitHandle(safeWaitHandle, handle);
        return safeWaitHandle;
    }
    
    ---<<< Marshal.cs中
    public static void InitHandle(SafeHandle safeHandle, IntPtr handle)
    {
        // To help maximize performance of P/Invokes, don't check if safeHandle is null.
        safeHandle.SetHandle(handle);
    }
    
    ---<<< Runtime/InteropServices/CriticalHandle.cs 
    ---<<< namespace System.Runtime.InteropServices  
    ---<<< public abstract partial class CriticalHandle : CriticalFinalizerObject, IDisposable  
    protected void SetHandle(IntPtr handle)
    {
        this.handle = handle;
    }
```

6. 看一下WaitOne的流程，总体来说还是阻塞的调用
```
    ---<<< WaitHandle.cs
    public virtual bool WaitOne(int millisecondsTimeout)
    {
        ArgumentOutOfRangeException.ThrowIfLessThan(millisecondsTimeout, -1);
        return WaitOneNoCheck(millisecondsTimeout);
    }
    
    ---<<< WaitHandle.cs 很长，忽略一些代码
    internal bool WaitOneNoCheck(
        int millisecondsTimeout,
        object? associatedObject = null,
        NativeRuntimeEventSource.WaitHandleWaitSourceMap waitSource = NativeRuntimeEventSource.WaitHandleWaitSourceMap.Unknown)
    {
       ---<<< 这里可能会先进行一次nonblock的wait试一下，如果成功了，那么直接拿到结果了，如果没有那么久进行block的wait
       if (tryNonblockingWaitFirst)
       {
           waitResult = WaitOneCore(waitHandle.DangerousGetHandle(), millisecondsTimeout: 0);
           if (waitResult == WaitTimeout)
           {
               // Do a full wait and send the wait events
               tryNonblockingWaitFirst = false;
           }
           else
           {
               // The nonblocking wait was successful, don't send the wait events
               sendWaitEvents = false;
           }
       }

       if (!tryNonblockingWaitFirst)
       {
           waitResult = WaitOneCore(waitHandle.DangerousGetHandle(), millisecondsTimeout);
       }
    }
    
    ---<<< WaitHandle.Unix.cs
    private static int WaitOneCore(IntPtr handle, int millisecondsTimeout) =>
        WaitSubsystem.Wait(handle, millisecondsTimeout, true);
    
    ---<<< WaitSubsystem.Unix.cs
    public static int Wait(
        WaitableObject waitableObject,
        int timeoutMilliseconds,
        bool interruptible = true,
        bool prioritize = false)
    {
        return waitableObject.Wait(Thread.CurrentThread.WaitInfo, timeoutMilliseconds, interruptible, prioritize);
    }
    
    ---<<< WaitSubsystem.WaitableObject.Unix.cs class WaitableObject
    public int Wait(ThreadWaitInfo waitInfo, int timeoutMilliseconds, bool interruptible, bool prioritize)
    {
        return Wait_Locked(waitInfo, timeoutMilliseconds, interruptible, prioritize, ref lockHolder);
    }
    
    ---<<< WaitSubsystem.WaitableObject.Unix.cs class WaitableObject 太长精简掉了
    public int Wait_Locked(ThreadWaitInfo waitInfo, int timeoutMilliseconds, bool interruptible, bool prioritize, ref LockHolder lockHolder)
    {
        WaitableObject?[] waitableObjects = waitInfo.GetWaitedObjectArray(1);
        waitableObjects[0] = this;
        waitInfo.RegisterWait(1, prioritize, isWaitForAll: false);

        return
            waitInfo.Wait(
                timeoutMilliseconds,
                interruptible,
                isSleep: false,
                ref lockHolder);
    }
        
```

7. 上面大多数是封装判断，线程，还有锁的好多操作，下面有些实际的逻辑
```
    ---<<< WaitSubsystem.ThreadWaitInfo.Unix.cs 
    public int Wait(int timeoutMilliseconds, bool interruptible, bool isSleep, ref LockHolder lockHolder)
    {
        _waitMonitor.Acquire();
        try
        {
            ---<<< 找到-1啦
            if (timeoutMilliseconds < 0)
            {
                do
                {
                    _waitMonitor.Wait();
                } while (IsWaiting);

                int waitResult = ProcessSignaledWaitState();
                return waitResult;
            }

            int elapsedMilliseconds = 0;
            int startTimeMilliseconds = Environment.TickCount;
            while (true)
            {
                bool monitorWaitResult = _waitMonitor.Wait(timeoutMilliseconds - elapsedMilliseconds);
                ---<<< 根据_waitSignalState可能提交就返回了
                int waitResult = ProcessSignaledWaitState();
                if (waitResult != WaitHandle.WaitTimeout)
                {
                    return waitResult;
                }

                ---<<< 如果还没有道timeout呢，则继续while
                if (monitorWaitResult)
                {
                    elapsedMilliseconds = Environment.TickCount - startTimeMilliseconds;
                    if (elapsedMilliseconds < timeoutMilliseconds)
                    {
                        continue;
                    }
                }

                // Timeout
                break;
            }
        }
        return WaitHandle.WaitTimeout;
    }
    
    ---<<< LowLevelMonitor _waitMonitor 
    ---<<< LowLevelMonitor.cs
    public void Wait()
    {
        ResetOwnerThread();
        WaitCore();
        SetOwnerThreadToCurrent();
    }
    
    ---<<< WaitCore不带参数这个，实际上是参数-1调的
    private void WaitCore()
    {
        Interop.Sys.LowLevelMonitor_Wait(_nativeMonitor);
    }

    private bool WaitCore(int timeoutMilliseconds)
    {
        if (timeoutMilliseconds < 0)
        {
            WaitCore();
            return true;
        }

        return Interop.Sys.LowLevelMonitor_TimedWait(_nativeMonitor, timeoutMilliseconds);
    }
```

8. 更底层的Wait
Common/src/Interop/Unix/System.Native/Interop.LowLevelMonitor.cs

~~调用coreclr的代码了~~，实际上是native/libs/System.Native/pal_threading.c
```
    [LibraryImport(Libraries.SystemNative, EntryPoint = "SystemNative_LowLevelMonitor_Wait")]
    internal static partial void LowLevelMonitor_Wait(IntPtr monitor);

    [LibraryImport(Libraries.SystemNative, EntryPoint = "SystemNative_LowLevelMonitor_TimedWait")]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static partial bool LowLevelMonitor_TimedWait(IntPtr monitor, int timeoutMilliseconds);
    
    ---<<< 不带参数的
    void SystemNative_LowLevelMonitor_Wait(LowLevelMonitor* monitor)
    {
        int error = pthread_cond_wait(&monitor->Condition, &monitor->Mutex);
    }
    
    ---<<< 带timeout时间 根据os版本不同 调用方式不一 pthread_cond_timedwait_relative_np是比较老的os的方法
    int32_t SystemNative_LowLevelMonitor_TimedWait(LowLevelMonitor *monitor, int32_t timeoutMilliseconds)
    {
        struct timespec timeoutTimeSpec;
    #if HAVE_CLOCK_GETTIME_NSEC_NP
        error = pthread_cond_timedwait_relative_np(&monitor->Condition, &monitor->Mutex, &timeoutTimeSpec);
    #else
        error = pthread_cond_timedwait(&monitor->Condition, &monitor->Mutex, &timeoutTimeSpec);
    #endif
    }
```
[pthread_cond_wait的文档](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_cond_wait.html)

9. 接下来看一下Set
```
    ---<<< EventWaitHandle.Unix.cs
    public bool Set()
    {
        SafeWaitHandle waitHandle = ValidateHandle();
        WaitSubsystem.SetEvent(waitHandle.DangerousGetHandle());
    }
    
    ---<<< WaitSubsystem.Unix.cs
    public static void SetEvent(WaitableObject waitableObject)
    {
        LockHolder lockHolder = new LockHolder(s_lock);
        waitableObject.SignalEvent(ref lockHolder);
    }
            
    ---<<< WaitSubsystem.WaitableObject.Unix.cs
    public void SignalEvent(ref LockHolder lockHolder)
    {
        switch (_type)
        {
            case WaitableObjectType.AutoResetEvent:
                SignalAutoResetEvent();
                break;
        }
    }
    
    ---<<< WaitSubsystem.WaitableObject.Unix.cs
    private void SignalAutoResetEvent()
    {
        for (ThreadWaitInfo.WaitedListNode? waiterNode = _waitersHead, nextWaiterNode;
            waiterNode != null;
            waiterNode = nextWaiterNode)
        {
            // Signaling a waiter will unregister the waiter node, but it may only abort the wait without satisfying the
            // wait, in which case we would try to signal another waiter. So, keep the next node before trying.
            nextWaiterNode = waiterNode.NextThread;

            if (waiterNode.WaitInfo.TrySignalToSatisfyWait(waiterNode, isAbandonedMutex: false))
            {
                return;
            }
        }
    }
    
    ---<<< WaitSubsystem.ThreadWaitInfo.Unix.cs
    public bool TrySignalToSatisfyWait(WaitedListNode registeredListNode, bool isAbandonedMutex)
    {
        // The wait would be satisfied. Before making changes to satisfy the wait, acquire the monitor and verify that
        // the thread can accept a signal.
        _waitMonitor.Acquire();
        UnregisterWait();

        if (wouldAnyMutexReacquireCountOverflow)
        {
            _waitSignalState = WaitSignalState.NotWaiting_SignaledToAbortWaitDueToMaximumMutexReacquireCount;
        }
        else
        {
            _waitedObjectIndexThatSatisfiedWait = signaledWaitedObjectIndex;
            ---<<< 这就是就是修改状态了，而在前面Wait的流程中while中会检测这个状态
            _waitSignalState =
                isAbandonedMutex
                    ? WaitSignalState.NotWaiting_SignaledToSatisfyWaitWithAbandonedMutex
                    : WaitSignalState.NotWaiting_SignaledToSatisfyWait;
        }
        
        ---<<< 实际上的之前的Wait会一直阻塞，由于调用了pthread_cond_wait导致线程因为条件变量而阻塞
        ---<<< 这里release会调用pthread_cond_signal从而unblock之前的等待
        _waitMonitor.Signal_Release();
    }
```
