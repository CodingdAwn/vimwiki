## c# linq runtime source code
从代码上看其实无非就是实现各种迭代器movenext，关键的优点在于yield，因为linq返回的基本上都是IEnumerable<TResult>或者TResult
IEnumerable这种可迭代的对象，基本上只是分配特定的迭代器，而实际上获得对象是在迭代的时候获得的，也就是滞后执行

## Enumerable Iterator

## foreach 本质上就是调用movenext

## movenext
movenext很多linq中的模块都实现各自的movenext，都是一个switch来做的状态机，但基本上都是顺序执行的

