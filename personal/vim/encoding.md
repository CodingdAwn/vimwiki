# vim中的编码问题
--------------

### 问题描述
--------------
经常会出现vim打开文件乱码的情况，到底怎么个情况呢

### vim中的编码变量
- encoding: 是vim内部使用的编码格式
- fileencoding: 文件读取，写入时用到的编码。当读取的文件的编码格式与vim内部不一样则会转换编码，也是airline处显示的格式
- fileencodings: 是一个列表，会依次从头匹配文件是否属于此种编码
- termencoding: 显示在终端时使用的编码，一般来说应该与terminal的设置一致

### 内网遇到的一个问题
文件的fileencoding是latin1，securecrt的设置是gb18030，encoding是utf-8，termencoding是utf-8  
尝试修改termencoding为latin1显示正确了

以上为个人现在的理解，具体还可以查看:h 

### refs
[vim编码介绍](http://edyfox.codecarver.org/html/vim_fileencodings_detection.html)
