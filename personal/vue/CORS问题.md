## 什么是CORS
Cross-Origin Resource Sharing, 跨来源资源共享

**出于安全性考虑 浏览器一般不允许CORS**

## 会出现什么问题
在同源政策下，非同源的 request 則會因為安全性的考量受到限制。

瀏覽器會強制你遵守 CORS (Cross-Origin Resource Sharing，跨域資源存取) 的規範，否則瀏覽器會讓 request 失敗。

## 什么情况下算是同源呢
* 相同的通訊協定 (protocol)，即 http/https
* 相同的網域 (domain)
* 相同的通訊埠 (port)

舉例：

下列何者與 https://example.com/a.html 為同源？

- https://example.com/b.html (⭕️)
- http://example.com/c.html (❌，不同 protocol)
- https://subdomain.example.com/d.html (❌，不同 domain)
- https://example.com:8080/e.html (❌，不同 port)

## 实际例子
首先，瀏覽器發送跨來源請求時，會帶一個 Origin header，表示這個請求的來源。

Origin 包含通訊協定、網域和通訊埠三個部分。

所以從 https://shubo.io 發出的往 https://othersite.com/data 的請求會像這樣：
```
[[GET]] /data/
Host: othersite.com
Origin: https://shubo.io
...
```

## 解决非同源方案
* Access-Control-Allow-Origin

當 server 端收到這個跨來源請求時，它可以依據「請求的來源」，亦即 Origin 的值，決定是否要允許這個跨來源請求。

如果 server 允許這個跨來源請求，它可以「授權」給這個來源的 JavaScript 存取這個資源。

授權的方法是在 response 裡加上 Access-Control-Allow-Origin header：
```
Access-Control-Allow-Origin: https://shubo.io
```
如果 server 允許任何來源的跨來源請求，那可以直接回 *：
```
Access-Control-Allow-Origin: *
```
