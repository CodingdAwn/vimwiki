## scriptnames 输出到新的buff中

### 需求
scriptnames命令输出的结果不好编辑，希望能在一个新的buff里输出，这样使用更好编辑查找

### 解决方案
```
exec ":vnew |pu=execute('scriptnames')"
```
vnew会打开一个新的vertical window\
| pipe \
pu -> put `:h put`\
excute('scriptnames')
