## windows端口映射脚本

#### 前提
- wsl2
- cmd或者powershell在`管理员模式`下

#### 添加映射的脚本
```
@echo OFF

wsl -- ifconfig eth0 | findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*"
for /F "tokens=2 delims= " %%i in ('wsl -- ifconfig eth0 ^| findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*"') do (
  set ip=%%i
)
echo %ip%

netsh interface portproxy show all
netsh interface portproxy add v4tov4 listenport=%1 connectaddress=%info% connectport=%1 listenaddress=0.0.0.0
netsh interface portproxy show all
```

wsl -- ifconfig eth0 是获取wsl内的ip

* findstr /r 
正则找出ip地址
* for /F in do
读取字符串或者文件内容 类似awk 其中tokens指的类似awk的 $1... delims分隔符

#### 删除映射的脚本
```
@echo OFF
netsh interface portproxy show all | findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*"
for /F "tokens=3-4 delims= " %%i in ('netsh interface portproxy show all ^| findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*"') do (
  echo %%i
  echo %%j
  netsh interface portproxy delete v4tov4 listenport=%%j listenaddress=0.0.0.0
)
netsh interface portproxy show all | findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*"
```
