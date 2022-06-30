= gdb ctrl中断问题

问题概述:
使用gdb调试的时候continue之后使用ctrl c中断不了 
这样的话只能使用kill再重启gdb,这样效率太低

原因:
是因为程序可能会截获了signal

解决方案:
使用ctrl c可以正常显示QUIT说明发送的是SIGQUIT
使用 info singals 会显示所有singal现在对于signal的处理方式
stop print pass
可以看到SIGQUIT  pass 是YES 所以会把这个signal传给程序
使用handle SIGQUIT stop print nopass设置为不传递
done

!!!!
尴尬 今天又试了一次 link这么设置不行了  而且gs不用这么设置也行了。。
看了不是这个问题 这个就留作一个思路吧...
