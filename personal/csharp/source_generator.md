# source generator
c#官方推出的一个代码生成的方案
总结一下最近使用source generator遇到的问题以及感受

## 优势(个人感觉)
- 由于source generator是编译期进行代码生成的，
    所以实际上不用像以前一样，每次都写一个脚本运行一下生成，再编译自己的工程
- 得益于他是在编译期生成，在代码生成过程中是，可以知道当前项目的所有编译信息的(语法树，所有的symbol，type等)
    这相当于全知全能了。拿着这些再去生成扩展类，非常的方便
- 还可以加载template(没有试过还)
- c#原生编写，宿主项目只需要引用一下即可

## 不足
- 经验不足，不敢臆想， 但是一开始查资料真的是很费劲
- 生成的代码总是不变，其实是官方问题，需要kill掉一些dotnet进程才行，同样在vs上有可能出现生成了但是代码引用错误，需要重启vs

## 遇到的问题
1. 一开始想法理解就是错的，一开始想着先用proto生成出代码，然后我在source generator的工程中使用反射，拿到所有的类型再生成代码。
    实际上source generator是在编译期啊，那里来的反射啊，反射是运行时啊，所以一开始的理解确实是错的
2. 第一次编译运行例子没有问题，可是稍微改下生成代码的程序，重新编译就是不管用
    原因就是上面说的，需要kill掉一些进程
3. 开始时没法调试，并不知道怎么调试代码生成的工程，所以如果生成代码的工程有什么问题，实际的结果就是引用的工程生成了空的文件，
    导致查问题也很麻烦
4. 官方的例子是不会生成实际的文件在硬盘上，而是在编译期间直接加入到编译中，而根据教程生成之后又有编译错误重复声明
```
    加上这个才好，就是就是告诉编译器不编译生成的文件
    <Compile Remove="$(CompilerGeneratedFilesOutputPath)/**/*.cs" />
```
5. 后面自己写的例子是用的`ISyntaxReceiver`，而实际上这个只能加载当前工程的ClassDeclarationSyntax，而当我把写好测试好的例子放到真的项目中却出错了
    因为真的项目中实际要获得的类型是在一个引用的工程中，后来使用查找引用的symbol的方式找到了对应symbol，然后找到property。
```
            var modelAssembly = context.Compilation.SourceModule.ReferencedAssemblySymbols.Where(
                symbol => symbol.Identity.Name == "Model"
                ).FirstOrDefault();
      
            var global = modelAssembly.GlobalNamespace;
            // find 'ET' namespace
            var et = global.GetMembers().Where(m => m.Name == "ET").FirstOrDefault();
            // find symbol 'RoleData'
            var roledata = et.GetMembers().Where(m => m.Name == "RoleData").FirstOrDefault();
```
6. 但是找到roledata的symbol后，发现我获得不了members，查了半天才看到文档中有接口，需要用迭代器去遍历
```
        private class FindAllSymbolsVisitor : SymbolVisitor
        {
            public List<IPropertySymbol> AllPropertySymbols { get; } = new List<IPropertySymbol>();

            public override void VisitNamedType(INamedTypeSymbol symbol)
            {
                foreach (var childSymbol in symbol.GetMembers())
                {
                    if (childSymbol is IPropertySymbol ps)
                    {
                        AllPropertySymbols.Add(ps);
                    }
                }
            }
        }
```
7. 有一些中文的注释生成时乱码，原因是一些生成文件的源文件是gbk的,生成出来的文件也是gbk的。 
    一开始在linux下file 文件看是utf-8 bom，以为是这个的问题，所以强制不使用bom生成的了utf8的文件，还是不对
    以为是常量的文本需要单独再转一次，也没有效果，最后发现是生成的源文件是gbk的，改成utf-8后可以了
8. 使用iconv还遇到一些问题，iconv -f iso-8859-1 -t utf-8 src -o dest，但是文件转完之后还是乱码
    iso-8859-1是file 出来的文件编码格式，然后使用iconv -f gbk -t utf-8 src -o dest转成功了
9. 还是iconv的问题，有一些文件生成后文件乱了，也不是乱码就是跟之前的代码不同了，感觉是转失败了
     原因在于我使用的是iconv -f gbk -t utf-8 src -o src,也就是直接写回源文件了，测试一个文件的没事
     但是遇到有些文件就报错了，然后尝试先生成到临时目录，然后再覆盖回来的方案解决

## 引用工程的工程文件配置

```
  <!-- 这里是为了输出生成的代码文件 -->
  <PropertyGroup>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
  </PropertyGroup>
  
  <ItemGroup> 
    <!-- 引用代码生成的工程 -->
    <ProjectReference Include="..\Tools\SourceGenerator\SourceGenerator.csproj"
                      OutputItemType="Analyzer"
                      ReferenceOutputAssembly="false" />
    <!-- 由于生成的代码输出到了项目内，导致重复的声明 -->
    <Compile Remove="$(CompilerGeneratedFilesOutputPath)/**/*.cs" />
  </ItemGroup>
  
  <!-- 编译时先删除生成的代码 -->
  <Target Name="CleanSourceGeneratedFiles" BeforeTargets="BeforeBuild" DependsOnTargets="$(BeforeBuildDependsOn)">
    <RemoveDir Directories="Generated" />
  </Target>
  <ItemGroup>
    <Compile Remove="Generated\**" />
    <Content Include="Generated\**" />
  </ItemGroup>
```  

## ref
[source generator debug](https://github.com/JoanComasFdz/dotnet-how-to-debug-source-generator-vs2022)

[github的整合source generator相关东西](https://github.com/amis92/csharp-source-generators)

[c# source generator cook book](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#table-of-content)
