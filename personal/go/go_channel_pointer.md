## go passing pointer type to channel

## 原因
发现有些blog会提倡传递在channel中传递指针

但是想想channel的数据都是copy value的

那么是否会有以下问题

1. 既然是指针 传递后修改指针指向数据,其他channel在收到后处理是否数据改变
2. 那么问题来了 传递指针的话多个goroutine就可以同时访问一个地址,那么竞争问题是否存在

## example
```go
package main

import (
	"fmt"
	"time"
)

type Data struct {
	data int
}

func printData(c chan *Data) {
	time.Sleep(time.Second * 1)
	data := <-c
	for {
		data.data = 100
	}
	//fmt.Println(fmt.Sprintf("Data in channel is: %v", *data))
}

func main() {
	fmt.Println("Main started...")
	a := Data{data: 1}
	b := &a

	//create channel
	c := make(chan *Data, 10)
	go printData(c)
	fmt.Println(fmt.Printf("Value of b before putting into channel %v", *b))
	c <- b
	for {
		b.data = 20
	}
	//fmt.Println(fmt.Sprintf("Updated value of a: %v", a))
	//fmt.Println(fmt.Sprintf("Updated value of b: %v", *b))
	//time.Sleep(time.Second * 2)
}
```

以上代码测试`go build -race` 
1，2问题同时满足

## 总结
channel中传递指针有一个大问题就是数据竞争,为了避免数据竞争而使用了channel,但是传递指针又会破坏这个机制

所以个人认为传递值还是指针,完全需要根据使用场景来区分

那么一些指针类型的数据呢? slice map interface?
复制一份新的再send

