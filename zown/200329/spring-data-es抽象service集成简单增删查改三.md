---
title: spring-data-es抽象service集成简单增删查改之三
date: 2020-02-29 10:45:21
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

接二

## 代码

继续整理了后续的实现类，注释已经比较清楚了，不在此赘述

本次主要更新了基于ElasticsearchTemplate来进行操作，主要是实现了批量的操作

```java
//EsTemplateBaseService.java
import org.elasticsearch.action.bulk.BulkProcessor;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.script.Script;

import java.util.List;

/**
 * es java api操作接口类，基于es template实现
 * @author yyt
 * @param <T>
 */
public interface EsTemplateBaseService<T extends BaseDoc> extends EsDocService<T> {
    /**
     * 获取index name
     * @return
     */
    String getIndexName();

    /**
     * 获取type name
     * @return
     */
    String getTypeName();


    List<T> bulkSaveOrUpdate(List<T> ts);

    /**
     * bulk save
     * @param ts doc集合
     * @param listener 自定义监听类
     * @return
     */
    List<T> bulkSaveOrUpdate(List<T> ts, BulkProcessor.Listener listener);
    List<T> bulkDeleteByDocPrimaryKey(List<T> ts);

    /**
     * bulk delete
     * @param ts doc集合
     * @param listener 自定义监听类
     * @return
     */
    List<T> bulkDeleteByDocPrimaryKey(List<T> ts, BulkProcessor.Listener listener);

    /**
     * scroll 查询
     * @param var1 查询条件
     * @return
     */
    List<T> scrollSearch(QueryBuilder var1);

    /**
     * scroll 查询
     * @param var1 查询条件
     * @param callBack 自定义的原始数据处理回调函数
     * @return
     */
    List<T> scrollSearch(QueryBuilder var1, ScrollDataCallBack callBack);

    <E extends BaseDto> List<T> bulkDelete(List<E> ts);
    /**
     * 根据单个dto bulk delete
     * @param e
     * @param <E>
     * @return
     */
    <E extends BaseDto> List<T> bulkDelete(E e);

    /**
     * 根据dto更新，
     * @param e dto
     * @param script 更新内容
     * @param <E>
     * @return
     */
    <E extends BaseDto> Long updateByQueryRequest(E e, Script script);

    /**
     * 根据boolQueryBuilder更新，
     * @param boolQueryBuilder boolQueryBuilder
     * @param script 更新内容
     * @return
     */
    Long updateByQueryRequest(QueryBuilder boolQueryBuilder, Script script);
    /**
     * 根据dto删除，逻辑删除
     * @param e dto
     * @param <E>
     * @return
     */
    <E extends BaseDto> Long deleteByQueryRequest(E e);

    /**
     * 根据boolQueryBuilders删除，逻辑删除
     * @param boolQueryBuilder boolQueryBuilder
     * @return
     */
    Long deleteByQueryRequest(QueryBuilder boolQueryBuilder);

}


import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.collections4.IterableUtils;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.action.bulk.BulkProcessor;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.search.ClearScrollRequest;
import org.elasticsearch.action.search.SearchRequestBuilder;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.client.Client;
import org.elasticsearch.common.unit.ByteSizeUnit;
import org.elasticsearch.common.unit.ByteSizeValue;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.reindex.UpdateByQueryAction;
import org.elasticsearch.index.reindex.UpdateByQueryRequestBuilder;
import org.elasticsearch.script.Script;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.sort.SortOrder;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.mapping.ElasticsearchPersistentProperty;
import org.springframework.data.elasticsearch.core.query.IndexQuery;
import org.springframework.data.elasticsearch.core.query.IndexQueryBuilder;
import org.springframework.data.elasticsearch.core.query.SearchQuery;
import org.springframework.data.elasticsearch.core.query.UpdateQueryBuilder;
import org.springframework.data.mapping.MappingException;
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;

import java.io.IOException;
import java.lang.reflect.ParameterizedType;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.elasticsearch.common.xcontent.XContentFactory.jsonBuilder;

/**
 * 用es template操作的抽象类，提供了大部分常用功能
 * @author yyt
 * @param <T>
 */
@Slf4j
@Component
public abstract class EsAbstractTemplateService<T extends BaseDoc> implements EsTemplateBaseService<T>, InitializingBean {
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    private BulkProcessor bulkProcessor;
    private Client client;

    private String indexName;
    private String typeName;

    /**
     * T的主键，注解为@Id的属性
     */
    private ElasticsearchPersistentProperty idProperty;

    private Class<T> Tcls;
    private int batchSize = 1000;

    /**
     * 新增或者修改
     * 新增的话会将id设置到doc的@id字段中
     * @param t
     * @return
     */
    @Override
    public T saveOrUpdate(T t){
        //保存前逻辑处理
        prepareSave(t);
        String id=findIdValue(t);
        if(StringUtils.isNotBlank(id)){
            UpdateRequest updateRequest = new UpdateRequest();
            updateRequest.doc(JSON.toJSONString(t),XContentType.JSON);
            UpdateQueryBuilder updateQuery = new UpdateQueryBuilder()
                    .withIndexName(getIndexName()).withType(getTypeName())
                    .withId(id)
                    .withUpdateRequest(updateRequest);
            elasticsearchTemplate.update(updateQuery.build());
        }else{
            IndexQuery indexQuery= new IndexQueryBuilder().withIndexName(getIndexName()).withType(getTypeName())
                    .withObject(t)
                    .build();

            elasticsearchTemplate.index(indexQuery);
        }

        return t;
    }

    /**
     * 根据主键id删除。不用bulk，直接DeleteRequest
     * @param id
     * @return
     */
    @Override
    public T delete(String id){
        if(StringUtils.isNotBlank(id)){
            bulkDeleteByPrimaryKey(this.bulkProcessor,indexName, typeName, id);
        }
        return null;
    }

    /**
     * 根据query scroll全量查询
     * @param var1
     * @return
     */
    @Override
    public Iterable<T> search(QueryBuilder var1){
        return scrollSearch(var1);
    }

    /**
     * 根据dto分页查询，由于之前业务限制，不能使用scroll分页。
     * @param dto
     * @param <E>
     * @return
     */
    @Override
    public <E extends BaseDto> Map<String,Object> searchByPage(E dto){
        QueryBuilder qbs=getQueryBuilderByDto(dto);
        SearchQuery sq=buildCommonPageSearch(qbs,dto);
        /*Page<T> rep= elasticsearchTemplate.startScroll(3000,sq,Tcls);*/
        Page<T> rep = elasticsearchTemplate.queryForPage(sq,Tcls);
        Map<String,Object> rem=new HashMap<>(2);

        rem.put("total",rep.getTotalElements());
        rem.put("data",rep.getContent());

        /*elasticsearchTemplate.clearScroll(((ScrolledPage)rep).getScrollId());*/
        return rem;
    }

    /**
     * 根据dto全量scroll搜索
     * @param t
     * @param <E>
     * @return
     */
    @Override
    public <E extends BaseDto> Iterable<T> search(E t){
        QueryBuilder qbs=getQueryBuilderByDto(t);
        return scrollSearch(qbs);
    }

    protected <E extends BaseDto> QueryBuilder getQueryBuilderByDto(E t){
        QueryBuilder qbs=buildCommonQuery(t);

        buildOwnQuery(qbs,t);
        return qbs;
    }

    /**
     * 根据业务主键判断是否已经有业务数据，有的话则填充es的id
     * @param s
     */
    protected abstract void prepareSave(T s);

    /**
     * 自己的分页查询,自己实现
     * @param query
     * @return
     */
    /*protected abstract Page<T> ownPageSearch(SearchQuery query);*/

    /**
     * 默认初始化
     */
    /*protected IiapAbstractTemplateDocService(){
        init();
    }*/
    /**
     * 由子类确定es-template并初始化
     */
    /*@Autowired
    protected IiapAbstractTemplateDocService(ElasticsearchTemplate elasticsearchTemplate){
        this.elasticsearchTemplate=elasticsearchTemplate;
        init();
    }*/

    @Override
    public List<T> scrollSearch(QueryBuilder queryBuilder, ScrollDataCallBack callBack){
        List<T> result=new ArrayList();
        SearchRequestBuilder search = client.prepareSearch(indexName).setTypes(typeName);
        search.addSort("_doc", SortOrder.ASC);
        // 批量读取条数
        search.setSize(batchSize);

        search.setQuery(queryBuilder);
        // 设置search context 快照维护20sec有效期
        search.setScroll(TimeValue.timeValueSeconds(20));
        // 获得首次查询结果
        SearchResponse scrollResp = search.get();
        Long totalHits=scrollResp.getHits().getTotalHits();


        log.debug("scroll query count " + totalHits);
        if(totalHits>0){
            int count = 1;
            do {
                log.debug("scroll query times: " + count);
                List<Map<String, Object>> subResult= readHits(scrollResp.getHits().getHits());
                if(callBack!=null){
                    callBack.handleData(subResult);
                }
                addAllSub(result,subResult);
                count++;
                scrollResp = client.prepareSearchScroll(scrollResp.getScrollId())
                        .setScroll(TimeValue.timeValueSeconds(20))
                        .execute().actionGet();
            } while (scrollResp.getHits().getHits().length != 0);
        }

        // 清除
        ClearScrollRequest scrollRequest = new ClearScrollRequest();
        scrollRequest.addScrollId(scrollResp.getScrollId());
        client.clearScroll(scrollRequest);
        return result;
    }


    @Override
    public List<T> scrollSearch(QueryBuilder queryBuilder){
        return scrollSearch(queryBuilder,null);
    }

    /**
     * 将es返回的默认结构解析并放入result
     * 个性化的doc需要子类重写
     * @param result
     * @param subResult
     */
    protected void addAllSub(List<T> result, List<Map<String, Object>> subResult) {
        if(CollectionUtils.isEmpty(subResult)){
            return;
        }
        subResult.forEach(e->{
            T t=JSONObject.parseObject(JSONObject.toJSONString(e),Tcls);
            putIdValue(t,(String) e.get("_idValue"));
            result.add(t);
        });
    }


    private List<Map<String, Object>> readHits(SearchHit[] searchHits){
        List<Map<String, Object>> list = new ArrayList<>();
        for (SearchHit hit: searchHits){
            Map<String, Object> map = hit.getSource();
            map.put("_idValue", hit.getId());
            list.add(map);
        }
        return list;
    }

    private BulkProcessor buildBulkProcessor(BulkProcessor.Listener listener){
       return BulkProcessor.builder(this.client, listener)
               // 每n个action flush一次
                .setBulkActions(getBatchSize())
               // bulk数据每达到nMB flush一次
                .setBulkSize(new ByteSizeValue(4, ByteSizeUnit.MB))
                .setConcurrentRequests(0)
                .setFlushInterval(TimeValue.timeValueSeconds(3))
                .build();
    }

    /**
     * bulk新增或者修改
     * @param ts
     * @return
     */
    @Override
    public List<T> bulkSaveOrUpdate(List<T> ts){
        if(CollectionUtils.isNotEmpty(ts)){
            ts.forEach(e->{
                prepareSave(e);
                bulkSaveOrUpdate(this.bulkProcessor,indexName, typeName, e);
            });
        }
        return ts;
    }

    /**
     * bulk新增或者修改
     * @param ts
     * @param listener 自定义的监听器
     * @return
     */
    @Override
    public List<T> bulkSaveOrUpdate(List<T> ts,BulkProcessor.Listener listener){
        BulkProcessor own=buildBulkProcessor(listener);
        if(CollectionUtils.isNotEmpty(ts)){
            ts.forEach(e->{
                bulkSaveOrUpdate(own,indexName, typeName, e);
            });
        }
        return ts;
    }
    /**
     * bulk删除
     * @param ts
     * @param listener 自定义的监听器
     * @return
     */
    @Override
    public List<T> bulkDeleteByDocPrimaryKey(List<T> ts,BulkProcessor.Listener listener){
        BulkProcessor own=buildBulkProcessor(listener);
        if(CollectionUtils.isNotEmpty(ts)){
            ts.forEach(e->{
                bulkDeleteByPrimaryKey(own,indexName, typeName, findIdValue(e));
            });
        }
        return ts;
    }

    /**
     * 按照es的主键id删除
     * @param ts
     * @return
     */
    @Override
    public List<T> bulkDeleteByDocPrimaryKey(List<T> ts){
        if(CollectionUtils.isNotEmpty(ts)){
            ts.forEach(e->{
                bulkDeleteByPrimaryKey(this.bulkProcessor,indexName, typeName, findIdValue(e));
            });
        }
        return ts;
    }

    /**
     * 按照doc里的属性来删除
     * @param ts
     * @return
     */
    @Override
    public <E extends BaseDto> List<T> bulkDelete(List<E> ts){
        List<T> rs=new ArrayList<>();
        if(CollectionUtils.isNotEmpty(ts)){
            ts.forEach(e->{
                Iterable<T> es= search(e);
                if(!IterableUtils.isEmpty(es)){
                    es.forEach(t->{
                        rs.add(t);
                        bulkDeleteByPrimaryKey(this.bulkProcessor,indexName, typeName, findIdValue(t));
                    });
                }
            });
        }
        return rs;
    }

    /**
     * 根据单个dto bulk delete
     * @param e
     * @param <E>
     * @return
     */
    @Override
    public <E extends BaseDto> List<T> bulkDelete(E e){
        List<E> list=new ArrayList<>();
        list.add(e);
        return bulkDelete(list);
    }

    /**
     * bulk 新增或者修改
     * @param ownbulkProcessor bulk processor
     * @param indexName
     * @param typeName
     * @param obj
     */
    private void bulkSaveOrUpdate(BulkProcessor ownbulkProcessor,String indexName, String typeName, T obj){
        String id=findIdValue(obj);
        if(StringUtils.isNotBlank(id)){
            UpdateRequest indexRequest = new UpdateRequest()
                    .index(indexName)
                    .type(typeName).id(id)
                    .doc(JSON.toJSONString(obj), XContentType.JSON);
            ownbulkProcessor.add(indexRequest);
        }else{
            IndexRequest indexRequest = new IndexRequest()
                    .index(indexName)
                    .type(typeName)
                    .source(JSON.toJSONString(obj), XContentType.JSON);
            ownbulkProcessor.add(indexRequest);
        }
    }

    /**
     * 根据主键bulk删除
     * @param ownbulkProcessor
     * @param indexName
     * @param typeName
     * @param id es的主键id
     */
    private void bulkDeleteByPrimaryKey(BulkProcessor ownbulkProcessor,String indexName, String typeName, String id){
        if(StringUtils.isNotBlank(id)){
            /*DeleteRequest indexRequest = new DeleteRequest()
                    .index(indexName)
                    .type(typeName).id(id);
            ownbulkProcessor.add(indexRequest);*/
            UpdateRequest updateRequest= null;
            try {
                updateRequest = new UpdateRequest().index(indexName).type(typeName).id(id).doc(jsonBuilder().startObject()
                        // 对没有的字段添加, 对已有的字段替换
                        .field("deleted", BaseConstant.FILED_DELETED).endObject());
                ownbulkProcessor.add(updateRequest);
            } catch (IOException e) {
                log.error("bulkDeleteByPrimaryKey jsonBuilder error:{}-{}-{}",indexName,typeName,id);
                throw new CustomBusinessException(ErrorCodeUtils.CommonErrorCode.ERROR_SERVER_INTERNAL,"bulkDeleteByPrimaryKey jsonBuilder error");
            }
        }
    }

    @Override
    public <E extends BaseDto> Long updateByQueryRequest(E e, Script script){
        QueryBuilder qbs=getQueryBuilderByDto(e);
        return updateByQueryRequest(qbs,script);
    }

    @Override
    public Long updateByQueryRequest(QueryBuilder queryBuilder, Script script){
        Long count=doCount(queryBuilder);
        if(count==0){
            return 0L;
        }

        UpdateByQueryRequestBuilder updateByQuery = UpdateByQueryAction.INSTANCE.newRequestBuilder(client)
                //为了防止版本冲突导致updateByQuery中止，设置终止冲突为（false）
                .abortOnVersionConflict(false);
        updateByQuery.source(getIndexName())
                .filter(queryBuilder)
                .size(count.intValue())
                .script(script);
        long updated = updateByQuery.get().getUpdated();
        log.debug("update success:{}-{}",queryBuilder,updated);
        return updated;
    }

    private Long doCount(QueryBuilder queryBuilder){
        SearchRequestBuilder countRequestBuilder = this.client.prepareSearch(getIndexName());
        countRequestBuilder.setSize(0);
        countRequestBuilder.setQuery(queryBuilder);
        return (countRequestBuilder.execute().actionGet()).getHits().getTotalHits();
    }

    @Override
    public <E extends BaseDto> Long deleteByQueryRequest(E e){
        QueryBuilder qbs=getQueryBuilderByDto(e);
        Script script= new Script("ctx._source.deleted = '"+BaseConstant.FILED_DELETED+"'" +
                ";ctx._source.updatedTime='"+System.currentTimeMillis()+"'");
        return updateByQueryRequest(qbs,script);
    }

    @Override
    public Long deleteByQueryRequest(QueryBuilder queryBuilder){
        Script script= new Script("ctx._source.deleted = '"+BaseConstant.FILED_DELETED+"'" +
                ";ctx._source.updatedTime='"+System.currentTimeMillis()+"'");
        return updateByQueryRequest(queryBuilder,script);
    }

    protected void setIndexName(String writeIndexName){
        this.indexName=writeIndexName;
    }

    protected void setTypeName(String typeName){
        this.typeName=typeName;
    }

    @Override
    public String getIndexName(){
        return this.indexName;
    }
    @Override
    public String getTypeName(){
        return this.typeName;
    }

    @Override
    public void afterPropertiesSet() throws Exception{
        init();
    }

    /**
     * 确定bulkProcess index、 client,idProperty
     */
    protected void init(){
        this.client=this.elasticsearchTemplate.getClient();
        this.bulkProcessor = buildBulkProcessor(new BulkProcessor.Listener(){
            @Override
            public void beforeBulk(long l, BulkRequest bulkRequest) {
            }

            @Override
            public void afterBulk(long l, BulkRequest bulkRequest, Throwable throwable) {
                log.error(" bulk insert_error : {}", throwable);
            }

            @Override
            public void afterBulk(long l, BulkRequest bulkRequest, BulkResponse bulkResponse) {
                log.debug(" bulk insert success  size: {}",bulkRequest.numberOfActions());
            }
        });

        //根据泛型获取默认的index和type
        this.Tcls=(Class<T>) ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];
        Assert.isTrue(Tcls.isAnnotationPresent(Document.class), "Unable to identify index name. " + Tcls.getSimpleName() + " is not a Document. Make sure the document class is annotated with @Document(indexName=\"foo\")");
        Document document = this.Tcls.getAnnotation(Document.class);
        setIndexName(document.indexName());
        setTypeName(document.type());
        this.idProperty= (ElasticsearchPersistentProperty)elasticsearchTemplate.getPersistentEntityFor(Tcls).getIdProperty();
        if(idProperty == null || !idProperty.getType().isAssignableFrom(String.class)){
            throw new MappingException(String.format("Couldn't find idProperty for type %s!", Tcls.getSimpleName()));
        }
    }

    /**
     * 获取T的主键的值
     * @return
     */
    protected String findIdValue(T t){
        return (String) elasticsearchTemplate.getPersistentEntityFor(Tcls).getPropertyAccessor(t).getProperty(idProperty);
    }

    /**
     * 设置主键的值
     * @param t
     * @param id
     */
    protected void putIdValue(T t,String id){
        elasticsearchTemplate.getPersistentEntityFor(Tcls).getPropertyAccessor(t).setProperty(idProperty,id);
    }

    /**
     * 获取批处理批次数
     * @return
     */
    protected int getBatchSize(){
        return this.batchSize;
    }
}

```
