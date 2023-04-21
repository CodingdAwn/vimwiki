# neomutt

# neomutt折腾记录
第三方的软件想要获取gmail的信息，现在不能像以前一样使用不安全的认证方式了，只能通过oauth2的方式

## 遇到的问题
- 家里的manjaro上，neomutt总是连接imap失败 超时，
- google申请api的那个凭证也总是不对
- 使用oauth2.py的脚本总是报错
下面说下几个问题的解决办法

### google api credentials 总是弄不对
1. 凭证一开始 欢迎界面的页面 不配置那些链接总是过不去，实际上那些链接都不需要，再次修改配置时直接去掉也通过了 
2. 需要选择 桌面应用 
3. 需要添加测试账号

### 使用oauth2.py的脚本总是报错 
1. 总是返回json的解析失败，打印出来确实多了很多没用字符，实在不知道为啥，暂时使用字符串提取了json的内容
2. 使用neomutt时，调用refresh token的接口 也是同样的问题
现在使用response[4:-7]
response为
1bf
{
    // 真正的json
}
0


# refs
[凭证的申请流程](https://www.jianshu.com/p/89022852d340)
