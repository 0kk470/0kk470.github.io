---
title: AssetBundle指定打包的坑
date: 2022-11-04 19:42:54
tags: [Unity, AssetBundle]
categories: Unity
---

# 前言

抽空给开源项目[金庸群侠传3D重制版](https://github.com/jynew/jynew)贡献代码时，有开发者同好觉得该项目现有的一键打包功能不是特别方便导出MOD，于是写了个优化过的MOD导出工具，这里记录下一些思路和总结。

# 一键打包的问题

旧的一键打包存在以下几个缺陷。
* 打包会囊括所有已经打好标签的```AssetBundle```，导致打包时间较长。
* 一键打包过程不能中断，期间发生了错误也会一直进行，不利于即时修正问题。
* 打包结束后的后续操作繁琐，需要手动将相关的MOD资源挪到输出目录下，而且还需要手动建立一个存储MOD相关信息的xml文件。

# 解决问题

那么如何解决这些问题？

* 首先，我们要确定首要目标是提供一个方便开发者导出MOD的工具，那么最好是构建另一个全新的打包```Pipeline```，与现有的```Pipeline```独立开，尽量不影响原有的一键打包逻辑。之后所有的开发思路都以```方便MOD导出```为目标来构建。

* 然后针对以上已存在的问题，提前整理好思路：
    * 打包时间太长。归根结底，打包时间太长还是因为一次性需要构建的资源太多。因此，可以让Unity只打包与该MOD相关的```AssetBundle```，在构建Pipeline中排除掉其他MOD的资源来节省时间。
    * 打包过程不能中断。首先确定下为什么打包过程不能中断，有没有什么办法可以在报错时提前中断，围绕这一思路去查阅相关资料（比如官方Manual里就提到可以开启```AssetBundle```的```StrictMode```，这样错误一发生就会中断打包）。然后对于一些容易传入的错误参数，提前做好校验，在打包前尽可能将其排除掉并给使用者弹窗提示相关信息（比如校验MOD配置是否合法，校验输出路径是否合法）等等。

    * 打包结束需要手动配置资源。分析下每个MOD的文件结构，其实都非常类似，包含一个资源包、一个地图包、一个XML配置文件，而且每个MOD在编辑器内都是有相关的```ScritableObject```配置文件。那么可以根据配置文件确定好每个MOD的相关信息，然后构建时按照一定的规则将相关资源和文件输出到指定文件夹即可。

# 结果

按照以上思路，最后整了个简单的MOD导出工具

![ModExportTool](1.png)

该工具优化掉了旧一键打包工具的缺陷点，缩短了导出MOD的持续时间，新的MOD构建导出流程也得到了部分MOD开发者同好的赞赏与感谢。


# 遇到的一些坑点

## [AssetBundleBuild](https://docs.unity3d.com/ScriptReference/AssetBundleBuild.html)的一些隐晦问题

为了使得Unity只构建指定MOD相关的```AssetBundle```，该工具使用了参数为```AssetBundleBuild```的重载函数
```CSharp
public static AssetBundleManifest BuildAssetBundles(string outputPath, AssetBundleBuild[] builds, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);
```
代替了常用的打包函数
```CSharp
public static AssetBundleManifest BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);
```
前者需要手动传入AB名和其相关的资源路径。

坑点如下:

* 分批次调用```BuildPipeline```的方式会导致AB依赖关系断裂
    
    * 原因: 为了让该工具也支持一次导出多个MOD，所以分了多次来调用打包API，结果打出来后的AB```manifest文件```全都没有依赖关系，由于看不到引擎源码，猜测依赖关系构建可能是在```BuildAssetBundles```内部自动完成的。
    * 解决: 将所有需要打包的MOD配置缓存起来，只调用一次```BuildAssetBundles```即可，然后再根据配置缓存一次挪动到指定输出目录。

* 打好的所有AB包磁盘容量大小总是比一键打包多```200MB```
    * 原因: 第一时间就想到了肯定是Resources目录的问题，毕竟之前就踩过坑了，用```AssetStudio```拆包看了下果然如此，
    * 解决: 将基础资源包也包含在pipleline中，这样Resources目录的资源会只达到基础资源包中，不过Resources资源冗余的问题依然存在（这个只能完全弃用该目录API方可解决，项目历史遗留问题）。

## 函数```AssetDatabase.GetAssetPathsFromAssetBundle```的坑

使用了该API来获取指定AB的所有资源路径来设置到```AssetBundleBuild```当中，结果构建时报错
![error2](2.png)
* 原因: 猜测是```GetAssetPathsFromAssetBundle```这个API会返回所有相关的资源文件，并且```AssetBundleBuild```作为参数的```BuildAssetBundles```方法内部并不会自动排除掉不合法的资源文件。
* 解决: 在获取路径时做一层过滤，排除掉脚本文件。

```CSharp
        private string[] GetValidAssetPathsInBundle(string bundle)
        {
            var results = AssetDatabase.GetAssetPathsFromAssetBundle(bundle).
                Where(assetPath => Path.GetExtension(assetPath) != ".cs").ToArray();
            return results;
        }
```


## OdinInspector的坑

由于项目集成了该插件，索性顺势就用该插件编写工具窗口，结果在选择文件路径时总是会报错。

![error3](3.png)

* 原因: 反编译了插件DLL看了下源码，发现是调用```OnGUI```绘制时在某些极端的情况下没有调用```EndLayoutGroup```。
* 解决: 查看了下插件版本，发现Odin是比较老的版本```3.0.5```，于是去插件官网搜了下最新的版本然后查看了下更新日志，发现这个问题在```3.1.0```版本之后修复掉了，于是手动更新了下，然后就正常了。

# 其他

导出工具源码参见 [Jyx2Modtool.cs](https://github.com/jynew/jynew/blob/main/jyx2/Assets/Editor/BuildTools/Jyx2ModTool.cs)