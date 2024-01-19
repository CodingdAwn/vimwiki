# 学习dotnet runtime 源码

## 先学习一些简单的模块，然后循序渐进
列一些感兴趣的模块，慢慢了解

* [X] 学习[linq](./learn_runtime/linq.md) 
* [X] 学习[timer](./learn_runtime/timer.md)
* [ ] 学习[thread] thread的东西很多，也包括task，timer等，实际上其他模块理解多了，慢慢就对thread理解多了
* [ ] 学习[task]
* [ ] 插播一条Task.Delay(5)的调用流程 [delay](./learn_runtime/delay.md)
* [ ] 学习[gc]

## 工程的介绍
1. 工程地址 -> [github](https://github.com/dotnet/runtime.git)
2. 工程如何编译 -> 根目录下有build.sh，需要安装很多库，编译完成后输出在artifacts目录中
3. 如何生成sln -> 使用lsp ominisharp看的源码，需要sln，linux下生成sln使用dotnet build slngen.proj
4. private corelib c#代码是一个shproj，如何生成sln， 它的sln其实是在coreclr下的System.Private.CoreLib里面，这里的工程引用了shproj
5. TODO coreclr代码 c的工程，makefile执行失败，没法生成compile json

## 介绍一些runtime调用的流程
现在只是初步看了看代码的理解，很多东西肯定理解的不到位，但我希望能先按自己的理解介绍部分流程，机制，方便记忆，以及之后的学习。

runtime代码基本上都是在 runtime/src/libraries/目录下，其中private corelib比较特殊，他是一个shproj，也是连接coreclr的桥梁.
可以看到corelib中最后调用的地方在当前工程已经找不到了，转而查看coreclr中的代码，可以搜索到对应的接口的c#实现。
而c#的实现一般也是调用coreclr的封装。

coreclr是dotnet之所以跨平台的原因，他是使用c写的，部分代码还是汇编的，其中还有一部分的c#的代码 System.Private.CoreLib，可以看到与libraries下一样，
实际上也是使用的c#的partial来做的，这里只是各种模块的扩展实现

比如线程的创建：
```
    /// 这是我使用thread的代码
    // 实际上只是创建一个thread的类，还没有真正的创建一个线程
    this.thread = new Thread(this.NetThreadUpdate);
    // 调用start时才会向系统申请创建thread
    this.thread.Start();
    
    /// 调用runtime corelib中的代码
    
    1. 调用启动线程接口
    public void Start(object? parameter) => Start(parameter, captureContext: true);
    
    2. 可以看到实际上调用的是StartCore
    private void Start(object? parameter, bool captureContext, bool internalThread = false)
    {
        StartHelper? startHelper = _startHelper;
        if (startHelper != null)
        {
            startHelper._startArg = parameter;
            startHelper._executionContext = captureContext ? ExecutionContext.Capture() : null;
        }

        StartCore(); ----<<<<  调用coreclr中扩展代码
    }
    
    3. coreclr下的corelib代码，实际上调用coreclr中vm的c代码
    private unsafe void StartCore()
    {
        lock (this)
        {
            fixed (char* pThreadName = _name)
            {
                StartInternal(GetNativeHandle(), _startHelper?._maxStackSize ?? 0, _priority, pThreadName);
            }
        }
    }
    
    // 这里看到调用c代码的形式，具体还没有研究
    // 即在c代码中找搜索ThreadNative_Start
    [LibraryImport(RuntimeHelpers.QCall, EntryPoint = "ThreadNative_Start")]
    private static unsafe partial void StartInternal(ThreadHandle t, int stackSize, int priority, char* pThreadName);
    
    4. 调用c库的代码
    extern "C" void QCALLTYPE ThreadNative_Start(QCall::ThreadHandle thread, int threadStackSize, int priority, PCWSTR pThreadName)
    {
        QCALL_CONTRACT;
        BEGIN_QCALL;
        ThreadNative::Start(thread, threadStackSize, priority, pThreadName);
        END_QCALL;
    }
    
    void ThreadNative::Start(Thread* pNewThread, int threadStackSize, int priority, PCWSTR pThreadName)
    
    5. 创建线程的关键流程
    BOOL Thread::CreateNewThread(SIZE_T stackSize, LPTHREAD_START_ROUTINE start, void *args, LPCWSTR pName)
    {
        bRet = CreateNewOSThread(stackSize, start, args);
    }
    
    BOOL Thread::CreateNewOSThread(SIZE_T sizeToCommitOrReserve, LPTHREAD_START_ROUTINE start, void *args)
    {
        // 根据OS创建线程了
        #ifdef TARGET_UNIX
            h = ::PAL_CreateThread64(NULL     /*=SECURITY_ATTRIBUTES*/,
        #else
            h = ::CreateThread(      NULL     /*=SECURITY_ATTRIBUTES*/,
        #endif
                                     sizeToCommitOrReserve,
                                     start,
                                     args,
                                     dwCreationFlags,
                                     &ourId);
                                     
        // unix下就是使用pthread
        // windows下就是CreateThread返回句柄
    }
```
## 关于线程与cpu cores的一些认识
在看源码时，发现Thread.CurrentThread是一个static变量，一开始没注意，就认为那永远都只有一个线程在运作啊，那如果cpu是多核的，明明cpu可以真并行的执行任务，可到了c#端为啥变成static的了呢。
后来在discord问了大佬，才发现自己没注意c#的一个特性[ThreadStatic] 表明这个字段是每个线程独有的，这样就可以解释了，实际上runtim的运行也是并行的。
