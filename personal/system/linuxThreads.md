# 一个古老的线程实现问题

# 缘由
赤壁项目的uniquenamed每次启动都会有多个进程，  
但是在代码中并未搜索到任何fork，clone函数。  
然后问了下说去掉编译时的static就会只起一个进程

当时我就懵了，为啥去掉static编译flag会有这种变化

# 查询确定问题
1. 首先先确认了-static的用法： 告诉编译器要找静态链接库，而不是动态链接库
2. 基于上一点那么这个现象只是因为libpthread.so, libpthread.a的区别
3. 为什么静态库和动态库会有区别呢?
   搜了将近1个多小时，看到了一些关于linuxThread的说法，然后开始研究linuxthread nptl
4. 发现linuxThread实现的线程就是使用clone，会有一个新的进程生成所以pid也不同
5. 验证一些问题 ./libpthread.so.0 NPTL0.61版本 说明动态库下的线程肯定是NPTL
6. 使用新的libpthread.a替换老得，确实也不会生成多个进程了

# reference
linuxThreads与NPTL的比较: https://www.jianshu.com/p/6c507b966ad1
