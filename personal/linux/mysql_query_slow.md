# mysql写数据很慢的问题

## 问题
某游戏压测时发现角色的存储很慢，大量的数据下发现
cpu确没有跑满，游戏进程，mysql，redis都没有跑满

发现确实mysql的写数据时，耗时最多。应该是mysql这边使用的问题

## 探究
- 使用mysql自带的压测工具测试
mysqlslap --user=root --password=123456 --concurrency=10 --number-of-queries=10000 --auto-generate-sql
Average number of seconds to run all queries: 4.395 seconds
1w/4s
发现自带的工具测下来也是很慢，但是cpu mysql-server是将近跑满了的

- 测试io压力的工具 iotop,发现写的速度很慢7m/s

- 查询到mysql官方文档中有说明，cpu跑不满的情况下可能是disk io的问题了
[mysql官方文档ref](https://dev.mysql.com/doc/refman/8.0/en/optimizing-innodb-diskio.html)

## 下一阶段朝着io的方向去查
实际上应该iotop的刷新问题，利用率一直没到100%，但是只要退出重新iotop就能看到io就是满了
