# protobuf 

## protogen
在生成cs文件的时候，package的名字大小写有问题
比如package ET，导出之后是namespage Et
加上+names=original即可
protogen --csharp_out=. test.proto +names=original
