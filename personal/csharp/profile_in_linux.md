# dotnet profile in linux

linux下只有两种方式 
1. dotnet-trace (which uses EventPipe) 
2. PerfCollect (which uses LTTng).

## 测试
尝试使用dotnet-trace，因为支持dotnet core版本更高

## 安装
dotnet tool install -g dotnet-trace

## 使用
dotnet的程序启动后
1. 可以通过dotnet-trace ps 来查看有哪些dotnet程序是可以trace的
2. 使用dotnet-trace collect -p pid 开始收集信息
3. 退出收集后可以看到对应的文件
4. 使用dotnet-trace collect -p pid --format speedscope来生成speedscope需要的格式
5. 把生成的结果直接拖到speedscope的browser中即可

## 如何查看
使用speedscope 

## ref
[linux performance trance](https://github.com/dotnet/coreclr/blob/master/Documentation/project-docs/linux-performance-tracing.md)
[speedscope document](https://github.com/jlfwong/speedscope/blob/main/README.md)

