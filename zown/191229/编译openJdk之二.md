---
title: windows编译openJdk之二-freeType+VS+jdk
date: 2019-12-21 19:32:55
tags: 
- openJDK
categories: openJDK
---

## 安装

### openJDK下载

官方下载地址：http://jdk.java.net/java-se-ri/8

![下载](/intro/0050.png)

### freeType

- freeType最好下载已经编译好的，下载后解压至和openJDK同一目录下

  下载地址：https://codeload.github.com/ubawurinna/freetype-windows-binaries/zip/master

  ![目录](/intro/0049.png)

### Visual Studio 2010 Compliers安装

- 根据指导手册，OpenJDK Windows构建需要VS2010专业版编译器。编译器以及其它工具的安装位置希望由变量VS100COMNTOOLS定义，这个会由Microsoft Visual Studio installer自动设置。VS2010所需要的部分只有C++的部分而已。尽量安装在默认安装的位置，安装完成后，重启电脑，确保环境变量VS100COMNTOOLS已经设置了。确保TMP或TEMP也在你的Windows paths中设置了，例如：C:\temp。其它的路径格式都不对，这有这一种正确。假设这个区域是用户私有的，那么在默认安装后，你应该会看到一个不同的用户路径在这些变量中。

  虽然只需要C++部分，但是单独安装编译器比较麻烦，还是选择Visual Studio C++ 2010 Professional或者Visual Studio C++ 2010 Express版本。这个网上到处都是，随便下一个，用完之后就可以删了。这里提供一个比较齐全的下载地址：http://www.itellyou.cn/。下载后只需要安装C++的模块就行了。Visual Studio C++ 2010 Express是没有64位的版本的。推荐还是下载Professional版本。

### 开始编译

通过CYGWIN进入OpenJDK8的解压目录。这里提一些基本的Linux知识。/路径是根路径，我们Windows上的C、D、E、F等盘都挂载在/cygdrive/路径下。通过pwd查看当前路径，一般进入的时候都是在/usr/home/xxx下面。直接通过cd /cygdrive/d/openjdk 更换盘符和其它路径，就可以进入OpenJDK解压的位置了。

