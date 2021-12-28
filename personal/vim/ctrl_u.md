## <C-U>到底有什么用

### 例子:
```
nnoremap <silent> <leader>od  :<C-u>CocList diagnostics<cr>
```
经常会看到vimscript中的map映射有<c-u>到底作用是什么的

### 结论
ctrl-u主要是因为: 在一些情况下会默认插入一些字符 c-u会清掉当前cursor前的所有字符

### vim help
:h c_CTRL-u

### 延伸 那么<c-r>呢
:h c_CTRL-r 就是为了插入register中的内容
c-r " 就可以插入unnamed register

