## 并发模型介绍

### Actor
在Actor模型中，Actor彼此之间发送消息，不需要中间，消息是异步发送和处理的,
在Actor中存在 mailbox，所有消息都是发送到mailbox队列中等待处理的 所以是异步的

Actor模型的默认规则
1. 所有Actor状态是Actor本地的，外部无法访问。
2. Actor必须只有通过消息传递进行通信。　　
3. 一个Actor可以响应消息:推出新Actor,改变其内部状态,或将消息发送到一个或多个其他参与者。
4. Actor可能会堵塞自己,但Actor不应该堵塞它运行的线程。

### CSP
Communicating Sequential Process
使用channel进行通信 比如go的goroutine和channel就是基于CSP设计的

因为使用channel作为通信方式 所以调用方和接收方互不了解 松耦合

CSP默认的方式是同步阻塞的 所以go在实现同步模型时为了解决阻塞问题 channel设计为可以缓冲的
