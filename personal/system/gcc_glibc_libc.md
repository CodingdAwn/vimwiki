# 理清楚gcc,glibc,libstdc++的关系
-------------

## 缘由
-------------
经常搞混这几个东西，每次查过明白了，过了一阵子又忘了，总结一下加深记忆

## 分析
-------------
* gcc: 额，这个应该不用多少说。。
* glibc: 封装了系统调用，比如打开文件什么的,glibc可以通过 ldd --version查看版本
* libstdc++:这个是c++的标准库，是跟对应的gcc配套的，即捆绑销售。gcc --version的版本即是libstdc++的版本

## Others
libstdc++的头文件在 /usr/include/c++/3.3.3/

libc -> whereis libc => /usr/lib/libc.a libc.so

另外还有libc++其实是clang配套的标准库

