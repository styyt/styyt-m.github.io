---
title: spring-data-es抽象service集成简单增删查改之二
date: 2020-02-22 18:15:23
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

接一

## 代码

继续整理了后续的实现类，注释已经比较清楚了，不在此赘述

主要是基于ElasticsearchRepository来实现简单的增删改查的

```java

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

/**
 * 公共查询的抽象service,基于spring-data-es，提供简单的增删查改
 * @author yyt
 * @param <M>
 * @param <S>
 */
public abstract class EsAbstractDocService<M extends ElasticsearchRepository<S,String>,S extends BaseDoc> implements EsDocService<S> {

    @Autowired
    private M elasticsearchRepository;

    @Override
    public S saveOrUpdate(S s){
        prepareSave(s);
        elasticsearchRepository.save(s);
        return s;
    }

    /**
     * 根据业务主键判断是否已经有业务数据，有的话则填充es的id
     * @param s
     */
    protected abstract void prepareSave(S s);

    @Override
    public S delete(String id){
        if(StringUtils.isEmpty(id)){
            return null;
        }
        Optional<S> os=elasticsearchRepository.findById(id);
        S s=os.get();
        s.setDeleted(BaseConstant.FILED_DELETED);
        s.setUpdatedTime(System.currentTimeMillis());
        elasticsearchRepository.save(s);
        return s;
    }



    @Override
    public <E extends BaseDto> Iterable<S> search(E e){
        QueryBuilder qbs=buildCommonQuery(e);

        buildOwnQuery(qbs,e);
        return elasticsearchRepository.search(qbs);
    }

    @Override
    public <E extends BaseDto> Map<String,Object> searchByPage(E dto){
        QueryBuilder qbs=buildCommonQuery(dto);

        buildOwnQuery(qbs,dto);
        SearchQuery query = buildCommonPageSearch(qbs,dto);
        Page<S> rep= elasticsearchRepository.search(query);
        Map<String,Object> rem=new HashMap(2);

        rem.put("total",rep.getTotalElements());
        rem.put("data",rep.getContent());
        return rem;
    }

    @Override
    public Iterable<S> search(QueryBuilder var1){
        return elasticsearchRepository.search(var1);
    }


}
```
