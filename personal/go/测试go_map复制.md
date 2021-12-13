## 测试一下go map=map是否有问题

### 为什么要做这个
项目中有热更表 map有竞争问题

解决方案是使用map=map 将新的map填充好数据再复制

测试下 没有出现问题 但实际上真的没有么？

### 疑问
由于map的在底层的结构是 hamp struct
那么直接复制 其实 绝对是原子操作 那么为什么测试下 没有出现问题呢 

### 亲自测一下
```
package main

var testdata map[int]string
var newMap map[int]string
var test int

func init() {
	testdata = make(map[int]string)
	newMap = make(map[int]string)
	testdata[0] = "A"
}

func main() {
	// 0x0012 00018 (main.go:14)	MOVQ	$1, "".test(SB)
	test = 1
	//
	//0x001d 00029 (main.go:15)	MOVQ	"".newMap(SB), AX
	//0x0024 00036 (main.go:15)	PCDATA	$0, $-2
	//0x0024 00036 (main.go:15)	CMPL	runtime.writeBarrier(SB), $0
	//0x002b 00043 (main.go:15)	JEQ	47
	//0x002d 00045 (main.go:15)	JMP	56
	//0x002f 00047 (main.go:15)	MOVQ	AX, "".testdata(SB)
	//0x0036 00054 (main.go:15)	JMP	71
	//0x0038 00056 (main.go:15)	LEAQ	"".testdata(SB), DI
	//0x003f 00063 (main.go:15)	NOP
	//0x0040 00064 (main.go:15)	CALL	runtime.gcWriteBarrier(SB)
	//0x0045 00069 (main.go:15)	JMP	71
	testdata = newMap
}

/*
func main() {
	// 这样汇编后 只有一条复制语句
	// 0x00ac 00172 (main.go:24)	MOVQ	BX, "".b+32(SP)
	a := make(map[string]int)
	b := a
	_ = b[""]

	//0x00d3 00211 (main.go:28)	MOVQ	"".a+40(SP), BX
	//0x00d8 00216 (main.go:28)	MOVQ	BX, "".b+32(SP)
	//a := make(map[string]int)
	//a["1"] = 1
	//b := a
	//_ = b[""]
}
*/
```

### 开启race测试
在操作operator= 和 map[index]的时候 还是会报race问题

### 结论
go的map其实就是 runtime下的*hmap

也就是其实就是个指针 所以map=map 按理说是一条指令

当前汇编之后发现 还是有各种情况的 有的时候是一条指令 有的时候是多条指令

### 查询资料后
https://github.com/Terry-Mao/gopush-cluster/issues/44 goim的作者 一直认为 指针赋值是原子的 在goim项目也有类似的操作
而相当一部分人也认为 是不确定行为 

https://stackoverflow.com/questions/21447463/is-assigning-a-pointer-atomic-in-go
https://forum.golangbridge.org/t/is-assigning-a-map-atomic-in-go/18602
http://yanyiwu.com/work/2015/02/07/golang-concurrency-safety.html
都认为应该明确使用sync.automic package才是明确的原子操作

最后我觉得也应该定义为 未定义行为 

因为是否是一条汇编指令 取决很多因素 故不应该依赖它 

而明确的使用sync.automic则绝对是确定性行为
