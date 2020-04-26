---
title: idea之CommandLineIsTooLong问题解决
date: 2020-01-25 13:50:46
tags: idea
categories: idea
---

## 问题

在某一次新增springboot项目部署启动时报了如下错误：

```
Error running 'XXXApplication': Command line is too long. Shorten command line for XXXApplication or also for Spring Boot default configuration.
```

## 解决方法

​	经过排查发现是因为项目启动的命令行太长引起的。最终通过idea的配置来解决

![ideaConfig](/intro/0062.png)

shorten command line 选项提供三种选项缩短类路径，这里我们需要选JAR manifest。

　　none：这是默认选项，idea不会缩短命令行。如果命令行超出了OS限制，这个想法将无法运行您的应用程序，但是工具提示将建议配置缩短器。

　　JAR manifest：idea 通过临时的classpath.jar传递长的类路径。原始类路径在MANIFEST.MF中定义为classpath.jar中的类路径属性。

　　classpath file：idea 将一个长类路径写入文本文件中

修改后的命令行为：

```java
"C:\Program Files\Java\jdk1.8.0_201\bin\java.exe" -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:3876,suspend=y,server=n -Drebel.base=C:\Users\ss1\.jrebel -Drebel.env.ide.plugin.version=2019.2.1 -Drebel.env.ide.version=2019.1 -Drebel.env.ide.product=IU -Drebel.env.ide=intellij -Drebel.notification.url=http://localhost:2614 -agentpath:C:\Users\ss1\.IntelliJIdea2019.1\config\plugins\jr-ide-idea\lib\jrebel6\lib\jrebel64.dll -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always -Dcom.sun.management.jmxremote -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -javaagent:C:\Users\ss1\.IntelliJIdea2019.1\system\captureAgent\debugger-agent.jar -Dfile.encoding=UTF-8 -classpath C:\Users\ss1\AppData\Local\Temp\classpath2065001607.jar com.yyt.aiot.cloud.HetuApplication
```

启动ok！