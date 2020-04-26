---
title: spring-data-es抽象service集成简单增删查改之一
date: 2020-02-16 20:38:12
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

最近项目中用到了大量而是es进行数据的整合和查询，在项目完成后将自己花点时间整理了下组件，然后放上来，后面也可以继续用到~

## 代码

直接上代码，太忙了，先整理了base类，以下为代码，都有注释，就不在此赘述了

```java
import lombok.Data;
import org.springframework.data.elasticsearch.annotations.Document;

/** BaseDoc.java
 * es的基础doc只定义了基础的字段和结构，子类需用注解@Document
 * @see Document
 * @author yangyuting
 */
@Data
public class BaseDoc {
    private Long createdTime;
    private Long updatedTime;
    private Integer deleted;
    private String version;
    private String createdBy;
    private String updatedBy;

    /**
     * 获取业务主键id，用来进行修改或者新增的逻辑判断依据，子类必须实现
     * @return id
     */
    public String findPrimaryKey(){
        throw new UnsupportedOperationException();
    }

}

/** BaseDto.java
 * es查询基础dto
 * @author yangyuting
 */
@Data
public class BaseDto {
    private Integer deleted;
    private String version;

    //更新时间段
    private Long startTime;
    private Long endTime;

    //创建时间段
    private Long createBeginTime;
    private Long createEndTime;
    @NotNull(message = "startIndex不能为空")
    private Integer startIndex;
    @NotNull(message = "size不能为空")
    private Integer size;
    private SortBy sortBy;
}

/** BaseConstant.java
 * 基础常量类
 */
public class BaseConstant {
    /**
     * 未删除
     */
    public static final Integer FILED_NO_DELETED=0;
    /**
     * 已删除
     */
    public static final Integer FILED_DELETED=1;

}

/** SortBy.java
 * 排序实体
 */
@Data
public class SortBy {
    /** 排序的字段，0---updateTime，1---actionList	 */
    private Byte field;
    /** 0---asc，1---desc */
    private Byte type;
}

/** SortFieldEnum.java
 * 排序字段枚举
 * @author yyt
 */
public enum SortFieldEnum {

    SortFieldEnum0(new Byte("1"),"updatedTime"),
    SortFieldEnum3(new Byte("2"),"createTime"),
    ;



    SortFieldEnum(byte field,String name){
        this.field=field;
        this.name=name;
    }

    private Byte field;
    private String name;

    public String getName() {
        return name;
    }

    public Byte getField() {
        return field;
    }
    public static SortFieldEnum getVal(Byte field){
        SortFieldEnum[] enums=values();
        for(SortFieldEnum item:enums){
            if (item.getField().equals(field)) {
                return item;
            }
        }
        return null;
    }

    public static SortFieldEnum getSortField(Byte field){
        return SortFieldEnum.getVal(field);
    }
}

import org.elasticsearch.search.sort.SortOrder;

/** SortTypeEnum.java
 * 排序类型enum
 * @author yyt
 */
public enum SortTypeEnum {
    SortTypeEnum0(new Byte("0"), SortOrder.ASC),
    SortTypeEnum1(new Byte("1"),SortOrder.DESC);

    SortTypeEnum(byte field, SortOrder type){
        this.field=field;
        this.type=type;
    }

    private Byte field;
    private SortOrder type;

    public Byte getField(){
        return field;
    }

    public SortOrder getType() {
        return type;
    }

    public static SortTypeEnum getVal(Byte field){
        SortTypeEnum[] enums=values();
        for(SortTypeEnum item:enums){
            if (item.getField().equals(field)) {
                return item;
            }
        }
        return null;
    }

    public  static SortTypeEnum getSortType(Byte field){
        return SortTypeEnum.getVal(field);
    }
}

/** ScrollDataCallBack.java
 * es scroll data 自定义的原始数据处理回调函数
 * @author yyt
 */
@FunctionalInterface
public interface ScrollDataCallBack {
    Object handleData(List<Map<String, Object>> datas);
}
```

以下为逻辑类

```java
/** EsDocService.java
 * ES 基础service的超类 提供简单的增删查改
 * @author yangyuting
 * @param <T>
 */
@NoRepositoryBean
public interface EsDocService<T extends BaseDoc> {
    /**
     * 新增或者修改
     * @param t
     * @return
     */
    T saveOrUpdate(T t);

    /**
     * 根据主键id删除
     * @param id
     * @return
     */
    T delete(String id);

    /**
     * 查询
     * @param var1
     * @return
     */
    Iterable<T> search(QueryBuilder var1);

    /**
     * 分页查询
     * @param t
     * @param <E>
     * @return
     */
    <E extends BaseDto> Map<String,Object> searchByPage(E t);

    /**
     * 根据dto查询
     * @param t
     * @param <E>
     * @return
     */
    <E extends BaseDto> Iterable<T> search(E t);

    //公共查询参数封装
    default  <E extends BaseDto> QueryBuilder buildCommonQuery(E s){
        BoolQueryBuilder qbs = QueryBuilders.boolQuery();
        qbs.must(QueryBuilders.termQuery("deleted", BaseConstant.FILED_NO_DELETED));
        String version= s.getVersion();
        if(StringUtils.isNotBlank(version)){
            qbs.must(QueryBuilders.termQuery("version", version));
        }

        if(s.getStartTime()!=null){
            qbs.must(QueryBuilders.rangeQuery("updatedTime").gte(s.getStartTime()));
        }
        if(s.getEndTime()!=null){
            qbs.must(QueryBuilders.rangeQuery("updatedTime").lte(s.getEndTime()));
        }

        if(s.getCreateBeginTime()!=null){
            qbs.must(QueryBuilders.rangeQuery("createTime").gte(s.getCreateBeginTime()));
        }
        if(s.getCreateEndTime()!=null){
            qbs.must(QueryBuilders.rangeQuery("createTime").lte(s.getCreateEndTime()));
        }

        return qbs;
    }

    /**
     * 实现类根据查询参数append自己的查询条件
     * @param qbs QueryBuilder
     * @param s dto
     * @param <E>
     */
    <E extends BaseDto> void buildOwnQuery(QueryBuilder qbs, E s);

    /**
     * 封装公共的pageSearch
     * @param qbs
     * @param dto
     * @param <E>
     * @return
     */
    default <E extends BaseDto>  SearchQuery buildCommonPageSearch(QueryBuilder qbs, E dto){
        Pageable pageable= PageRequest.of(dto.getStartIndex(),dto.getSize());
        FieldSortBuilder fsb = SortBuilders.fieldSort(SortFieldEnum.getSortField(dto.getSortBy().getField()).getName())
                .order(SortTypeEnum.getSortType(dto.getSortBy().getType()).getType());
        SearchQuery query = new NativeSearchQueryBuilder().withQuery(qbs).withSort(fsb)
                .withPageable(pageable).build();
        return query;
    }
}
```

