## wsl端口映射总结

### 问题描述
在文脉传奇的项目中 我是用wsl2开发一般,但是windows下启动客户端连接 wsl2的端口都不行

### 查询解决方案
查询后需要做端口映射

- 以下命令可能需要管理员权限访问
```
// 这个是gateway的端口映射
netsh interface portproxy add v4tov4 listenport=10810 connectaddress=172.19.174.103 connectport=10810 listenaddress=0.0.0.0
```
```
// 这个是center的端口映射
netsh interface portproxy add v4tov4 listenport=8000 connectaddress=172.19.174.103 connectport=8000 listenaddress=0.0.0.0
```
```
// 这个是查询所有映射端口
netsh interface portproxy show all
```
```
// 这个是删除映射
netsh interface portproxy delete v4tov4 listenport=10810
```

### 中途遇到的问题
1. gopath问题 ps. 这个是题外话了 gopath center和scene有冲突 使用临时变量解决
2. center启动失败 本地的mysql redis都没有。。
3. mysql root连接总是有权限问题 https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost 设置mysql_native_password
4. redis 没有密码的话也是一样的问题 添加一个密码解决
5. center scene gateway互相没有找到 修改config配置
6. windows下客户端连不上 wsl的服务 使用端口映射

至此终于客户端连上了wsl的游戏服务
