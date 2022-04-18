= Error detected while processing function <snr>52_maps =

= 问题描述: =
在公司内网开发机上 由于系统环境比较老
记录一次查询vim启动报错的流程

= 流程 =
1. 发现报错后google没有相似问题，大多数都是mru的问题
2. 突然发现<snr>52_maps这个maps是函数名
3. grep vim-init发现有unimpaired.vim中有调用这个函数
4. 屏蔽次插件 第一次解决问题

unimpaired定义了一些]开头的快捷键 发现也都是我暂时用不到的

= TODO =
之后可以再看看具体是哪个有问题
more tagets than list items
