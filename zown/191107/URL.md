---
title: SpringBoot启动报Failed to retrieve application JMX service URL
date: 2019-11-02 11:11:26
tags: SpringBoot
categories: SpringBoot
---

### 背景

最近将项目切换到IDEA上，SpringBoot启动报如下错误：

![启动报错](/intro/0034.png)

经过长时间分析和排查，终于解决了此问题。

### 解决思路和过程

首先看到报错信息指的是jmx连接失败，自己猜测可能有以下几种可能

1. jmx服务没有启动
2. IDEA配置有问题
3. IDEA用java发送的请求被防火墙或者其他安全设备阻拦

那么依次进行问题排查

1. 把程序搬到eclipse，启动本应用，发现可以正常启动，so排除了第一种可能

2. 在IDEA上用mvn package打包成jar包，然后再手动启动该jar包

   ![jar启动](/intro/0035.png)

   发现可以正常启动，so排除第二种可能

3. 使用管理员身份运行IDEA

   ![设置IDEA](/intro/0037.png)

   再重新启动

   ![IDEA启动](/intro/0036.png)

   成功了！果然是因为权限的问题！~