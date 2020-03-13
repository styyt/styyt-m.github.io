---
title: es的模糊搜索
date: 2020-01-05 14:36:24
tags: es
categories: es
---

# 写在前面

最近新的项目中用到了大量的es，作为es方面的新手，逐渐熟悉es的DSL、analyzer、search等等，所以记录一下，加深自己的理解

理解es的模糊搜索首先理解搜索分词和存储分词

- 搜索分词指的是match类查询，比如match、match_phrase，同时我们可以指定分词器，term类查询是不分词查询,比如term、terms

- 存储分词es的text（2.x的string）类型字段存储，同搜索分词我们也可以利用不同的分词器来优化我们的搜索逻辑，比如默认的standard、ik等等

  - 存储不分词，在es2.x和5.x有不同的做法

    - 2.x

      由于2.x中没有text和keyword，所以需要用如下设置

      ```json
      {"type": "string",
      "index": "not_analyzed"}
      ```

    - 5x

      将字段类型设置为keyword即可

      ```json
      {"type": "keyword"}
      ```

es的模糊搜索就是利用搜索分词+存储分词或者wildcard+存储不分词(5.x)

- 搜索分词+存储分词（match_phrase+text）

  match_phrase是分词的，text也是分词的。match_phrase的分词结果必须在text字段分词中都包含，而且顺序必须相同，而且必须都是连续的。语法如下

  ```json
  {
      "query": {
          "bool": {
              "must": [
                  {
                      "match_phrase": {
                          "name": [
                              "yyt"
                          ]
                      }
                  }
              ]
          }
      }
  }
  ```

- wildcard+存储不分词（wildcard+keyword）

  首先设置字段存储为keyword不分词，然后利用wildcard来进行正则表达式匹配

  ```json
  {
    "query":{
      "wildcard":{
        "name":"*yyt*"
      }
    }
  }
  ```

# 思考

​	text类型在存储数据的时候会默认进行分词，并生成索引。而keyword存储数据的时候，不会分词建立索引，显然，这样划分数据更加节省内存。为了性能考虑，我们应该仔细斟酌一下 text or keyword ?