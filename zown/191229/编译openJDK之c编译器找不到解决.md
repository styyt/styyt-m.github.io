---
title: windows编译openJDK之c编译器找不到解决
date: 2019-12-28 15:43:37
tags:
- openJDK
categories: openJDK
---

# 写在前面

为什么单独列出来这个问题呢？是因为其他问题网上都有很多类似的示例和解决方法，比如Cygwin 版本不对，包不对等等，这些我就不再写了。

c编译器找不到的问题花了我很多时间去找资料，功夫不负有心人，让我找到了结局方法，所以写下来记录。

# 问题

按照之前的流程安装好所必需的环境：cygwin 、VS2010、freeType 2.7后，开始对openJDK进行编译。

```bash
cd /cygdrive/c/openJDK8/jdk8u
bash ./configure --with-freetype=/cygdrive/c/openJDK8/freetype-2.10.1 --with-boot-jdk=/cygdrive/c/java7/jdk1.7.0_80 -with-target-bits=64 --with-jvm-variants=server --with-debug-level=release
```

编译过程中发现了一个报错：

![编译报错](/intro/0046.png)

具体错误如下

```bash
configure: The C compiler (located as
/cygdrive/c/progra~3/micros~1.0/vc/bin/cl) does not seem to be the required
microsoft compiler.
configure: The result from running it was: "Compilador de optimizaci▒n de
C/C++ de Microsoft (R) versi▒n 19.00.24234.1 para x86"
```

# 解决

经过查阅资料找到了简单的解决方法:https://bugs.openjdk.java.net/browse/JDK-8212986#，所以现在有两个解决思路

- 思路一

  按照文章后面提供的patch脚本里的内容修改openJDK的bash命令，风险较大，具体修改地方如下：

  ```bash
  diff -r 70fab3a8ff02 make/autoconf/toolchain.m4
  --- a/make/autoconf/toolchain.m4	Tue Jul 16 07:29:12 2019 +0900
  +++ b/make/autoconf/toolchain.m4	Tue Jul 16 10:51:44 2019 +0200
  @@ -459,7 +459,7 @@
       # Microsoft (R) 32-bit C/C++ Optimizing Compiler Version 16.00.40219.01 for 80x86
       COMPILER_VERSION_OUTPUT=`"$COMPILER" 2>&1 | $GREP -v 'ERROR.*UtilTranslatePathList' | $HEAD -n 1 | $TR -d '\r'`
       # Check that this is likely to be Microsoft CL.EXE.
  -    $ECHO "$COMPILER_VERSION_OUTPUT" | $GREP "Microsoft.*Compiler" > /dev/null
  +    $ECHO "$COMPILER_VERSION_OUTPUT" | $GREP "Microsoft" > /dev/null
       if test $? -ne 0; then
         AC_MSG_NOTICE([The $COMPILER_NAME compiler (located as $COMPILER) does not seem to be the required $TOOLCHAIN_TYPE compiler.])
         AC_MSG_NOTICE([The result from running it was: "$COMPILER_VERSION_OUTPUT"])
  ```

- 思路二

  可以看到根本原因是因为openJDK不支持非英语版vs，我们卸载中文版的，重新下载英文版的安装即可