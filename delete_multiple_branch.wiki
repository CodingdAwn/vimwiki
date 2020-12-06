= how to delete multiple local branches and remote branches =

== delete local branch ==
- 简单的删除一个分支
  git branch -D branch_name
- 删除多个
  git branch -D branch_nameA branch_nameB
- 上面的方法太麻烦了？ 试试模糊匹配
  git branch -D $(git branch --list 'prefix*')


== delete remote branch ==
- 简单删除一个分支
  git push origin --delete branch_name
- 试试复杂的删除多个
  git branch -r | awk -F/ '/prefix/{print $2}' | xargs -I bname git push origin --delete bname

let's explan what it is??

git branch -r 列出所有remote分支

awk非常强大 用于将输入 根据分割符 分别用于处理指定的action
awk -F/ 使用/作为分割符
'{print $2}' 需要执行的action
/prefix/ 指定前缀

xargs 用于将标准输入转为 命令的参数 非常适合和find等命令联动
xargs -I bname 将参数绑定到bname上
git push origin --delete 删除远程分支的命令

