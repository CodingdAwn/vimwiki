## 研读一下go net/http的实现

### 目录结构
删除了所有的test文件

简单的介绍下各个文件的作用

```
.
├── cgi
│   ├── child.go
│   ├── host.go
│   └── testdata
│       └── test.cgi
├── client.go           -> 有defaultclient 用来调用get post等
├── clone.go
├── cookie.go           -> cookie implement
├── cookiejar
│   ├── jar.go          -> cookie jar
│   ├── punycode.go
├── doc.go              -> just a document for introduce some tips
├── fcgi
│   ├── child.go
│   ├── fcgi.go
├── filetransport.go
├── fs.go
├── h2_bundle.go
├── header.go
├── http.go
├── httptrace
│   ├── trace.go        -> provide trace mechanisms
├── httputil
│   ├── dump.go
│   ├── httputil.go
│   ├── persist.go
│   ├── reverseproxy.go
├── internal
│   ├── ascii
│   │   ├── print.go
│   ├── chunked.go
│   └── testcert
│       └── testcert.go
├── jar.go              -> cookie jar todo: difference with cookiejar/
├── method.go
├── omithttp2.go
├── pprof
│   ├── pprof.go
├── request.go
├── response.go
├── roundtrip.go        -> just a function transport.roundtrip
├── server.go           -> server side
├── sniff.go
├── socks_bundle.go
├── status.go           -> status code
├── transfer.go
├── transport.go        -> 保存了长连接的信息 为了近期的reuse而不是创建新的连接
└── triv.go
```

### 流程
只分析下主要的流程

首先client的结构

```
type Client struct {
	Transport RoundTripper
	CheckRedirect func(req *Request, via []*Request) error
	Jar CookieJar
	Timeout time.Duration
}
```

request的结构
```
```

0. 准备request

1. client.Get client.Post client.Head 都是调用client.do

2. client.do 
```
// 调用send发送请求
resp, didTimeout, err = c.send(req, deadline)
// 根据resp返回的code看是否需要redirect
redirectMethod, shouldRedirect, includeBody = redirectBehavior(req.Method, resp, reqs[0])
```

3. client.send
```
// 发送前添加cookie
if c.Jar != nil {
	for _, cookie := range c.Jar.Cookies(req.URL) {
		req.AddCookie(cookie)
	}
}
  
// 发送
resp, didTimeout, err = send(req, c.transport(), deadline)
// 发送后根据resp设置cookie
if c.Jar != nil {
	if rc := resp.Cookies(); len(rc) > 0 {
		c.Jar.SetCookies(req.URL, rc)
	}
}
```

4. send
```
// 如果有用户名密码 设置authorization
if u := req.URL.User; u != nil && req.Header.Get("Authorization") == "" {
	username := u.Username()
	password, _ := u.Password()
	forkReq()
	req.Header = cloneOrMakeHeader(ireq.Header)
	req.Header.Set("Authorization", "Basic "+basicAuth(username, password))
}

// 设置取消函数 timer等
stopTimer, didTimeout := setRequestCancel(req, rt, deadline)

// call RoundTrip rt is Transport (type is RoundTrip)
// default implement at transport.go
resp, err = rt.RoundTrip(req)
```

5. roundtrip
先介绍一下 Transport的结构吧
```
type Transport struct {
  idleMu       sync.Mutex
  // persistConn 把长连接保存下来 for reusing
  idleConn     map[connectMethodKey][]*persistConn // most recently used at end
  idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
  idleLRU      connLRU
  
  altMu    sync.Mutex   // guards changing altProto only
  altProto atomic.Value // of nil or map[string]RoundTripper, key is URI scheme
  
  connsPerHostMu   sync.Mutex
  connsPerHost     map[connectMethodKey]int
  connsPerHostWait map[connectMethodKey]wantConnQueue // waiting getConns
  
  DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
  DialTLSContext func(ctx context.Context, network, addr string) (net.Conn, error)
  
  // http/2相关 todo
  TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper
  
  // http2 relevant
  nextProtoOnce      sync.Once
  h2transport        h2Transport // non-nil if http2 wired up
  tlsNextProtoWasNil bool        // whether TLSNextProto was nil when the Once fired
  ForceAttemptHTTP2 bool
}
```

然后是长连接的保存persistConn
```
type persistConn struct {
  // if http/2 use this and others is nonactive
  alt RoundTripper
  
  t         *Transport
  conn      net.Conn
  tlsState  *tls.ConnectionState
  br        *bufio.Reader       // from conn
  bw        *bufio.Writer       // to conn
  nwrite    int64               // bytes written
  reqch     chan requestAndChan // written by roundTrip; read by readLoop
  writech   chan writeRequest   // written by roundTrip; read by writeLoop
  writeLoopDone chan struct{} // closed when write loop ends
  mu                   sync.Mutex // guards following fields
  reused bool  // whether conn has had successful request/response and is being reused.
  mutateHeaderFunc func(Header)
}
```

roundtrip logic
```
// reuse recently conn
pconn, err := t.getConn(treq, cm)
  
  // details of getConn
  1. make a wantConn{} 
  2. queueForIdleConn(wantConn) 从队列中找一个dile的conn
     delails of queueForIdleConn
     - check DisableKeepAlives 如果没有keepalive 应该就不会保存conn
     - 每个connectMethodKey都有一个连接链表 key是 proxy schme addr的集合
        list, ok := t.idleConn[w.key]
        如果没有找到idle的 则吧wantConn放到idleConnWait中
     - 期间会检查一些过期的idleconn 从链表中删除
   
  3. if queueForIdleConn找到闲置的conn 直接返回persistConn
  4. 没有找到闲置的conn call queueForDial 建立一个连接
  5. dialConnFor call dialConn
     details of dialConn
      建立新的连接 conn, err := t.dial(ctx, "tcp", cm.addr())
      启动读写goroutine
      go pconn.readLoop()
      go pconn.writeLoop()
  6. detail for dial
     // 使用预设的DialContext创建新的连接
     // 默认一般使用net.dial package内的 func (d *Dialer) DialContext
     return t.DialContext(ctx, network, addr)

最终找到了一个persistConn返回
承接上文 找到conn后 
resp, err = pconn.roundTrip(treq)
  // details of roundTrip
  - 写入request请求
  pc.writech <- writeRequest{req, writeErrCh, continueCh}
    // writeLoop 处理写入
    wr := <-pc.writech
      // 写入到buffio中
      wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
      // 写入到conn中
      err = pc.bw.Flush()

  // resc是一个接受response的channel
  resc := make(chan responseAndError)
  // 将请求转发给 readLoop处理 处理服务器返回
  pc.reqch <- requestAndChan{
  	req:        req.Request,
  	cancelKey:  req.cancelKey,
  	ch:         resc,
  	addedGzip:  requestedGzip,
  	continueCh: continueCh,
  	callerGone: gone,
  }
  
    // readLoop 处理读server's response
    resp, err = pc.readResponse(rc, trace)
    // 然后尝试将连接放到idleConn to reuse
  
  之后会等待respnse结果
	case re := <-resc:
    return re.res 返回response

return resp
// body是在外面通过io读取的

如果失败 会检测是否需要重试
```

### server side
服务器大体上就是一般的 go conn.serve

### 关注的问题
如何处理一个req一个reponse
- 关键应该在于 requestAndChan中`ch`是一个unbuffered channel
衍生问题 那么多条连接 可以保证顺序么
- 不能 而且也没必要啊 
如何处理同步 异步request
异步request是有`上层逻辑`处理 

### 一些新的写法
使一个struct 不可比较 很多struct都加上了 _ incomparable
```
type responseAndError struct {
	_   incomparable
	res *Response // else use this response (see res method)
	err error
}

// 原理为 在类中插入了一个func 不会占用长度
// incomparable is a zero-width, non-comparable type. Adding it to a struct
// makes that struct also non-comparable, and generally doesn't add
// any size (as long as it's first).
type incomparable [0]func()
```


