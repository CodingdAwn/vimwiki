# 查询udp端口是否可用

## 遇事不决问google或者chatgpt
给出的方案有：
1. Using nmap: nmap -sU -v ip 会列出所有udp的端口情况 实际上不可用，不知道是否是服务器有策略阻止
2. Using netcat: nc -vz ip port 查看端口的连接情况，实际上使用也不可用，返回的成功实际上不可用

然后想到我直接在服务器上抓包得了
然后尝试使用tcpdump

tcpdump port 9009
发现没有任何消息

至此大致可以确定不是自己程序的问题，是运维的防火墙策略问题可能，最终找运维解决
