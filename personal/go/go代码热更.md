## 需求:
一些运营活动 希望可以热更go的代码  
首先整个项目是go写的   
但是go是静态语言希望能够提供热更代码的机会  

## 方案:
1. 使用`动态链接库`的形式 热更替换so或者dll
2. 使用内嵌`脚本语言`
3. 使用`rpc`的形式 远程调用接口
4. 其他进程间通信方式

#### 方案对比
- 方案3: 使用rpc的形式可以直接套用grpc 但是这样很偏向微服的方向  
并不是太贴合现在的整体架构
- 方案4: 其他方式还不如rpc
- 方案2: 使用内嵌脚本的话 原有逻辑需要翻译成脚本  

还是使用替换动态链接库的方式 相对于现有框架更合适

**tips**: 现在的更多的做法 其实热更代码的需求很低了 
原因在于 现在大多数都是使用`微服架构` 如果服务是无状态的 基本上直接添加新服务 关闭旧服务就行了
但是游戏服务 还是很少做成无服务的 如果相对休闲一些的游戏 应该完全可以

## 方案探索
动态替换链接库有这么几种方案:
- `go plugin`
- 使用原生系统接口 调用cgo方法dlopen LoadLibrary

两种方案都可 因为go plugin也是调用的dlopen  
但是go plugin只能在*unix下使用

## 实践
#### 生成动态链接库
linux:
```
go build -buildmode=plugin -o plugin.so plugin.go
go build -buildmode=c-shared -o plugin.so plugin.go
```
window:
```
// windows下不支持plugin
go build -o plugin.dll -buildmode=plugin plugin.go
go build -buildmode=c-shared -o plugin.dll plugin.go
```
go help buildmode
```
-buildmode=c-shared
  Build the listed main package, plus all packages it imports,
  into a C shared library. The only callable symbols will
  be those functions exported using a cgo //export comment.
  Requires exactly one main package to be listed.

-buildmode=plugin
  Build the listed main packages, plus all packages that they
  import, into a Go plugin. Packages not named main are ignored.
```

btw gcc编译指令:
```
gcc -c -fPIC lib.c -o lib.o -fno-gnu-unique
gcc lib.o -shared -o lib.so -fno-gnu-unique
gcc -o main main.c -ldl --no-gnu-unique
添加--no-gnu-unique依旧不行
```

#### 加载plugin方式
这中间 试了很多种方案 试图解决不能原名替换so 加载的so内存无法释放
- 第一种尝试 使用 [dl library](github.com/rainycape/dl) 的库封装了dlopen等操作  
  但是发现两个问题:  
  1. 加载so的内存释放不了 调用close只能将句柄释放
  2. cp替换动态库 报段错误 只能换名字使用版本控制

- 第二次尝试 尝试直接使用系统函数  
  同样解决不了上面两个问题  
  查看man dlopen dlclose dlsym 得知dclose是只是引用计数减1  
  stackoverflow的回答 [answer](https://stackoverflow.com/questions/24467404/dlclose-doesnt-really-unload-shared-object-no-matter-how-many-times-it-is-call)

- 第三次尝试 使用c语言编写 使用gcc编译动态库  
  名字替换的问题得以解决  多次尝试后发现go build出来的就是不行.. gcc的可以
  内存释放问题 更改了gcc编译选项还是无果 暂时放弃
- 最终尝试 还是使用go plugin吧 至少官方的还安全些 其实代码都差不多
  
## 问题总结
1. 生成的动态库 不能替换
2. 生成的动态库 内存不会释放
3. go plugin`不支持windows`
4. c-shared模式没有导出.h头文件 -> 需要指定`//export funcName`
5. 使用RTLD_LOCAL RTLD_LAZY RTLD_NOW均不行.

## 代码简单摘要
pulgin.go:
```
package main
type greeting string
func (g greeting) Greet() {
	println("hello in dll")
}
var G greeting
```

main.go
```
package main
import (
	"fmt"
	"os"
	"plugin" // 1. 导入标准库
	"time"
)

// 2. 为插件定义接口
type Greeter interface {
	Greet()
}

type Process struct {
}

func (pro Process) Greet() {
	fmt.Println("process call")
}

func main() {
	var greeter Greeter
	for i := 0; i < 50; i++ {
		// 3. 查找并实例化插件
		plug, err := plugin.Open("./greet.so")
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		// 4. 找到插件导出的接口实例，其实这个不是必须的
		symGreeter, err := plug.Lookup("G")
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		if i > 10 {
			greeter = &Process{}
		} else {
			// 5. 类型转换
			greeter, _ = symGreeter.(Greeter)
		}

		// 6. 调用方法
		greeter.Greet()

		time.Sleep(time.Second)
	}
}
```

### 补充问题
1.plugin was built with a different version of package golang.org/x/sys/unix  
  原因在于go plugin不仅要求go的版本 package的`版本一致` 还需要`gopath一致`

### 补充一些其他的方向
[cgo.Handle](https://gocn.vip/topics/12493)
