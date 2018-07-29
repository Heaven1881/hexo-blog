---
title: 使用windeployqt来自动部署Qt依赖
tags: Qt
categories: 编程语言
date: 2018-07-28 08:09:56
---


我们使用Qt动态库时，经常需要将依赖的动态库和应用程序一起打包。缺少动态库的Qt程序会出现各种各样的错误。


例如，启动Qt时如果找不到对应的动态库，应用程序可能会提示：

```text
it could not find or load the QT platform plugin “windows”
```
大多数情况下我们无法保证目标机器上部署了正确版本的Qt，所以Qt的开发人员通常会将Qt程序依赖的动态库一起打包到Qt应用程序中。

<!-- more -->

Qt提供了一个Windows平台的工具`windeployqt`，我们可以使用它来将所有依赖的Qt动态库拷贝到当前应用程序下。

这个工具的好处就是能够将程序动态加载的QtDLL也自动拷贝到当前目录下，具体方法如下：

-  将对应的`.exe`程序拷贝到一个新的文件夹
-  在当前文件夹下打开命令行
-  调用`windeployqt`命令，最好使用完整路径，例如

```
c:\Qt\Qt5.2.1\5.2.1\msvc2010_opengl\bin\windeployqt.exe application.exe
```

上面的命令执行成功后，当前目录应该就有应用程序依赖的所有DLL。

## 其他
除了Qt之外，我可能还要处理 MSVC 相关的DLL和其他第三方DLL依赖
- 对于MSVC这类的DLL，应在安装应用程序时同时安装对应的VC运行库
- 其他第三方DLL则需要我们手动拷贝到应用程序目录下

## 参考资料
> [Application failed to start because it could not find or load the QT platform plugin “windows” - Stackoverflow
](https://stackoverflow.com/questions/21268558/application-failed-to-start-because-it-could-not-find-or-load-the-qt-platform-pl)