== vim == 

很多vim的问题,操作 每次查完知道了过段时间就忘了.这次把遇到的问题 直接记录下来吧，
有的时候再次查的时候还真不好查...

- <C-U>到底用处是啥 (延伸c-r)-> [[./vim/ctrl_u.md]]

- 将scriptnames的返回输出到新的buff中 -> [[./vim/scriptnames_output.md]]

- 内网vim启动报错的查找记录 -> [[./vim/error_detected_snr_52_maps]]

- 把当前行append到上一行的末尾 -> [[./vim/join_line.md]]

- 关于vim的中一些编码的理解 -> [[./vim/encoding.md]]

== some trick ==
* 活用vip viw vi{ vi() 等操作
* 选中多行后s/$/; 在每行最后添加;
* 选中多行后，可以使用正则拿到想要修改的文字，例如s/\(\w.*\)/data[0] = "\1";  选中文本被粘贴到\1处
* 数字多行选中后，使用g c-a会按照自然数修改如 0000 0123 这个在旧版本vim中无效
* [m ]m [[ 快速移动到函数开头

== some problem ==
* 之前总是发现目录下多了一个 '\'的文件 其实是在操作vim command的时候 :w \ 就会写入一个\的文件 操作失误属于
