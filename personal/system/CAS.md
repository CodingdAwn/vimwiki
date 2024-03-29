## CAS

compare and swap 属于一种乐观锁的方式

## 乐观与悲观
- 乐观者： 认为别的线程不会修改值 如果发现修改 需要再次`重试`
- 悲观者： 认为别的线程会修改者

### 优点
1. cpu指令级的操作， 只有一个原子操作
2. 不需要操作系统锁的支持 lock-free

### 缺点
1. `ABA`问题
2. 循环时间长开销大。自旋CAS如果长时间不成功，会给CPU带来非常大的`执行开销`
3. 只能保证`一个共享变量`的原子操作

### 操作流程
CAS的三个参数
1. 内存位置V（它的值是我们想要去更新的）
2. 预期原值A（上一次从内存中读取的值）
3. 新值B（应该写入的新值）

过程：  
将内存位置V的值与A比较（compare），
- 如果相等，则说明没有其它线程来修改过这个值，所以把内存V的的值更新成B（swap）
- 如果不相等，说明V上的值被修改过了，不更新，而是返回当前V的值，再重新执行一次任务再继续这个过程。

所以，当多个线程尝试使用CAS同时更新同一个变量时，其中一个线程会成功更新变量的值,剩下的会失败。  
失败的线程可以重试或者什么也不做。
