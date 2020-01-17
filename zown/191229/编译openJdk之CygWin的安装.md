---
title: windows编译openJdk之一-CYGWIN
date: 2019-12-13 15:26:46
tags: 
- openJDK
categories: openJDK
---

### 写在前面

最近在学习JVM，就接触到JDK的源码，传统的ORACLEJDK也就是JDK没有被开源，而OpenJDK是JDK的开放源码版本，以GPL协议的形式发布，适用于商业和个人。

准备openJDK编译环境，第一步就是要去安装个CYGWIN，用来在windows下模拟linux运行环境。

#### **JDK和OpenJDK的区别，可以归纳为以下几点：** 

- 授权协议的不同： 

  OpenJDK采用GPL V2协议发布，而JDK则采用JRL协议发布。
  两个协议虽然都是开放源代码的，但是在使用上的不同在于GPL V2允许在商业上使用，而JRL只允许个人研究使用。 

- OpenJDK只包含最精简的JDK： 

  OpenJDK不包含其他的软件包，比如Rhino Java DB JAXP……，并且可以分离的软件包也都是尽量的分离，但是这大多数都是自由软件，你可以自己下载加入。 

- OpenJDK源代码不完整： 

  这个很容易想到，在采用GPL协议的OpenJDK中，SUN JDK的一部分源代码因为产权的问题无法开放OpenJDK使用，其中最主要的部分就是JMX中的可选元件SNMP部分的代码。
  因此这些不能开放的源代码将它作成plug，以供OpenJDK编译时使用，你也可以选择不要使用plug。而Icedtea则为这些不完整的部分开发了相同功能的源代码(OpenJDK6)，促使OpenJDK更加完整。 

- 不能使用Java商标

  这个很容易理解，在安装OpenJDK的机器上，输入“java -version”显示的是OpenJDK

### 安装

- 首先去CYGWIN官网下载最新版的安装程序： https://cygwin.com/install.html 

- 运行安装程序，选择线上下载安装包并安装，在这里需要注意国内网速很慢，所以需要使用镜像服务器，这里我使用的是： http://mirrors.163.com/cygwin/ ，速度会快很多。

- 编译openJDK过程中除了CYGWIN默认安装的工具组件之外。还需要下面这些工具（因版本不同可能有所区别，我的版本：![版本](/intro/0045.png)）

  |     分类     |    包    |
  | :----------: | :------: |
  |    Devel     | binutils |
  |    Devel     |   make   |
  | Interpreters |    m4    |
  |    Utils     |   cpio   |
  |    Utils     |   awk    |
  |    Utils     |   file   |
  |   Archive    |   zip    |
  |   Archive    |  unzip   |
  |    System    |  procps  |

  ![选择模块](/intro/0043.png)
  
- 一步步走完之后，安装完毕，最后记得检查环境变量

  ![环境变量](/intro/0048.png)

未完待续... ...