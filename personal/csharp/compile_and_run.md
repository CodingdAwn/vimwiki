## 学习一下dotnet的编译执行流程

### 编译生成的文件
dotnet build生成的exe dll并不是我们一般认为的可执行文件和动态库

* dll就是一个IL code 可以使用file A.dll查看
* exe is dotnet bytecode

```
PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows
```

也可以使用ildasm来查看IL code, ildasm可以使用runtime工程coreclr编译出来
```
ildasm A.dll
```

虽然exe文件可以直接运行，就想我们使用其他windows的可执行文件一样

但是实际上，它是还是得找到runtime，用runtime来加载IL code然后运行

### JIT
c# 实际上是使用JIT compile将IL code转化为 machine code运行的，跟其他jit差不多的道理

比如当执行到method之后，如果是第一次就会将IL code convert to machine code执行，然后缓存，下次直接执行machine code

### 执行流程
1. 根据不同的语言选择编译器
2. 编译成MSIL
3. 将MSIL转化为机器码
4. run it

[ref compile process](https://learn.microsoft.com/en-us/dotnet/standard/managed-execution-process)
