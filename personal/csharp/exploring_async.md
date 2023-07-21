# Exploring the async/await State Machine
learn about c# await pattern

## The Awaitable Pattern
* When we await some operation, it first gets a chance to complete synchronously. 
* introduce IsComplete GetResult OnComplete
* 主要的是await不是async，如果async的函数中没有await的关键字，那么这个函数的执行就是同步的
* 异步其实就是回调

[ref](https://vkontech.com/exploring-the-async-await-state-machine-the-awaitable-pattern/)
---------------------------------

## Main Workflow and State Transitions
* 介绍了状态机的运行的流程
* 可以看出状态机根据await划分了step，然后不停的movenext检查每一步的状态机状态，每一步awaiter是否IsComplete
  如果完成，则调用GetResult获取结果，没有完成调用ContinueWith

[ref](https://vkontech.com/exploring-the-async-await-state-machine-main-workflow-and-state-transitions/)
[ref -> closures](https://vkontech.com/the-intuitive-guide-to-understanding-closures-in-c/)
---------------------------------

## Conceptual Implementation
* 通过实现的简单版状态机可以看出如果awaiter调用IsComplete没有完成，那么会调用OnComplete(MoveNext)也就是当任务完成时，会调用MoveNext即回调处理
* 相同类型的result只会创建一个awaiter，re-use

[ref](https://vkontech.com/exploring-the-async-await-state-machine-conceptual-implementation/)

---------------------------------

## Concrete Implementation
* live on the stack -> This means no work for the Garbage Collector and optimal memory footprint
* However, the main idea of async programming is to pause, offload the current thread, and continue later (probably on some other thread).
* boxing State Machine onto heap - > 因为要在之后继续调用呢，栈肯定没了，必须boxing到heap
  IAsyncStateMachine boxed = stateMachine; stateMachine本身是一个struct，但是这样复制就导致了boxing，boxed就在堆了，因为boxed是一个引用类型

[ref](https://vkontech.com/exploring-the-async-await-state-machine-concrete-implementation/)
---------------------------------

## Synchronization Context
* The default Synchronization Context implementation would just schedule your callback to the Thread Pool
* 

[ref](https://vkontech.com/exploring-the-async-await-state-machine-synchronization-context/)
---------------------------------
