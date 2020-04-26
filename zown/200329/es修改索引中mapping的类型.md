---
title: es修改索引中mapping的类型
date: 2020-02-02 09:23:04
tags: es
categories: es
---

## 问题

es不允许对索引的type进行字段的删除和修改，只允许新增。

### 新增字段

```js
PUT my_index/_mapping/my_type
{
  "properties": {
    "new_column": { 
      "type":     "integer"
    }
  }
}
```

## 删除和修改的步骤比较麻烦，具体步骤如下：

- 新增索引t2为想要的数据类型

- 将t1 reindex到t2

  ```
  POST _reindex
  {
    "source": {
      "index": "t1"
    },
    "dest": {
      "index": "t2",
      "version_type": "external"
    },
    "script": {
      "source":"def w = ctx._source.remove('weight');ctx._source.weight=w;",
      "lang": "painless"
    }
  }
  ```

- 数据reindex完成后删除t1
- 设置索引t1为想要的数据类型

- 将t2 reindex到t1，语法同步骤2