---
title: Windows下进行QT开发的一些总结
date: 2022-10-14 16:20:20
tags: QT
---

## 前言

花了两三天写了一个[小工具](https://github.com/0kk470/modtool_qt)，用于将开源游戏[金庸群侠传3D](https://github.com/0kk470/jynew)重制版的Mod上传到Steam创意工坊。

这里记录下使用QT开发的一些问题。


## 问题

### 使用信号槽机制connect相关回调时找不到可匹配的函数。
***
* 原因: 部分UI类的slot函数存在多个同名函数重载的情况.
* 解决: 使用```QOverload<T>::of```显示地指定对应函数地址。


### QMake文件找不到win64的条件编译宏
***
* 添加依赖库时需要区分``x86``和``x86_64``的```Steamlib```时发现没有针对64位系统的宏。
* 解决: 使用自定义宏手动控制，如下。
```C++
DEFINES += USE_x64 
win32
{
    contains(DEFINES, USE_x64){
        LIBS += -L$$PWD/steam/win64 -lsteam_api64
    }else{
        LIBS += -L$$PWD/steam/win32 -lsteam_api
    }
}
```

### 无法使用QWebEngineView依赖控件
***
* 解决: ```QTCreator```构建编译器默认选择的```MingGW```，但该环境下```QWebEngineView```不包含在依赖库里，换用```MSVC```编译即可。

### 切换到MSVC后编译错误 " Visual Studio error C2001:常量中有换行符 "
***
* 原因: ```MSVC```编译时使用的默认编码是根据windows本地语言来的，中文运行环境下为```GBK```，但是```QT```保存编码为```UTF-8```。因此```MSVC```编译时如果代码文件包含中文就会出现该问题。

* 解决: 项目的.pro文件里添加如下指令强制```MSVC``` 使用```UTF-8```编码
```C++
msvc {
    QMAKE_CFLAGS += /utf-8
    QMAKE_CXXFLAGS += /utf-8
}
```

### Release构建完成后只有exe没有相关DLL
***
* 解决: CMD命令行运行 ```qwindeploy.exe [exe输出路径]```即可生成相关依赖库。

## 总结

* [QT文档](https://doc.qt.io/qt-5.15/) + Google + Stackoverflow 能解决99%的问题。