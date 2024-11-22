# 一个java调用shell失败的问题
web工具比较老了，而且自己对web java shell啥的都不是比较懂，所以遇到了很多问题，总结一下

## 现象
java调用脚本时报错 permission denied. publickey gssapi keyex with ... blabla
而直接运行脚本实际上是没有问题的!

## 分析
1. 首先看到permission denied，总以为是什么权限问题
   尝试了很多方式，改文件权限，执行加sudo等都没效果
   实际上调用脚本是有权限的，因为脚本调用成功了，只是其中rsync调用失败了
2. rsync调用失败，查了很多都没说到点子上
   搜索publickey的那个报错时，发现ssh有可能是因为没有设置免密登录的原因
   这才想到我两个服务器间还没有设置免密登录，设置好公钥好才好

## 为啥直接脚本没有问题呢
我认为是因为ssh的工具设置了，使用同样的秘钥去自动连接其他服务器
