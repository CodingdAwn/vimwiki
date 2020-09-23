= test git merge and rebase =

环境：
test_branch
master
并且人为制造冲突测试

1.git merge master T2时间早于M2
  M1 M1
  M2 T2
     M2
     T3 merge
  merge合回主干 因为主干现在没有修改 所以是fast merge   
  M1
  T2
  M2
  T3
2.git merge master T2时间早于M2 
  M2 T2
     M2
     T3
  rebase合回主干
  与上面一样的log

3.git rebase master T2时间早于M2
  M2 T2
     rebase
     M2
     conflict commit
     T2
     continue commit
  git rebase 回主干
  就算T2的时间早于M2
  rebase的操作 还是会 
  先把test分支的提交取消 然后使用master 
  再使用test分支提交
  
4.先rebase 再merge
  一样的结果 且merge不会产生更新的提交
5.master先提交
  merge到test分支
  master再次提交
  merge回master
  线比较乱
  再整理测试案例把

结论：
merge会生成多余的一次merge提交
所有冲突只会解决一次 并不会重复解决
合回主干都是一样的原因在于 主干一直没有修改！
网上有说merge跟时间有关系 这个案例测试没有关系 git merge master, master一定在test后

