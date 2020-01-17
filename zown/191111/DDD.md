---
title: mybatis_plus_generator with DDD
date: 2019-11-09 19:52:10
tags:
- mybatis_plus_generator
categories: mybatis_plus_generator
---

### 写在前面

​	随着项目越来越庞大，MVC的传统web架构已经出现了弊端，就算是分了微服务，项目中的各个模块间也不可避免的出现了各种复杂的关系与调用，可能改动一个小地方，将会牵一发而动全身。

​	所以公司开始大力推广DDD，经过几天的学习和上手，自己简单的根据mybatis-generator写了个可以生成DDD模型的工具类（并且可以轻松切换生成代码的type，MVC或者DDD）

### 正文

代码地址为： https://github.com/styyt/mybatis-generator-plus-ddd 

生成的结构大致为：![代码结构](/intro/0038.png)

模型服务内调用过程为：![调用过程](/intro/0039.png)

​	