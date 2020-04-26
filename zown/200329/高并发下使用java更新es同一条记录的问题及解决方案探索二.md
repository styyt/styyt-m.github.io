---
title: 高并发下使用java更新es同一条记录的问题及解决方案探索二
date: 2020-03-21 16:45:20
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

接上文

## 分析

为了防止出现上文的这种情况，自己抽空写了一个简单的线程池，用来处理这些更新失败的request，因为之前已经在updateRequest内做了自定义的版本控制，所以放弃了es内部的乐观锁的版本控制，

具体实现代码如下，注释已经很清楚了，不在此赘述：

```java
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.elasticsearch.index.reindex.UpdateByQueryRequestBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.*;

/**
 * 解决设备es高并发下更新失败的线程处理类
 * @author yyt
 */
@Slf4j
@Component
public class DeviceThreadExecutor {

    /**
     * 经过多次单元测试，定义了一个基本符合业务的线程池，2个核心线程，最大线程是5,多余线程存活时间1秒，超大的任务队列
     */
    private static final ExecutorService singleThreadExecutor = new ThreadPoolExecutor(2, 5,
            1000L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>());

    /**
     * 最大重试次数
     */
    private static final Integer MAX_NUMBERS=10;


    /**
     * 对updateByQueryRequestBuilder进行tostring，以便日志打印，方便后续问题追踪
     * @param updateByQueryRequestBuilder
     * @return
     */
    public static String updateByQueryRequestBuildertoString(UpdateByQueryRequestBuilder updateByQueryRequestBuilder){
        StringBuilder stringBuilder=new StringBuilder();
        stringBuilder.append("source:").append(updateByQueryRequestBuilder.source());
        stringBuilder.append("request:").append(updateByQueryRequestBuilder.request());
        return stringBuilder.toString();
    }

    /**
     * elasticsearchTemplate，后面改进时会用到
     */
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;

    /**
     * 任务处理的实体类
     */
    @Data
    public static class DeviceHandler{
        /**
         * updateByQueryReques
         */
        private UpdateByQueryRequestBuilder updateByQueryRequestBuilder;
        /**
         * 重试次数
         */
        private Integer numbers=1;
        /**
         * 更新条数
         */
        private Long updated=0L;

        @Override
        public String toString(){
            StringBuilder stringBuilder=new StringBuilder();
            stringBuilder.append("numbers:").append(numbers).append(";updated").append(updated)
                    .append(";updateByQueryRequestBuilder:").append(updateByQueryRequestBuildertoString(updateByQueryRequestBuilder));
            return stringBuilder.toString();
        }
    }


    /**
     * 根据deviceHandler执行或放入任务至线程池
     * @param deviceHandler
     */
    public static void runDeviceHandler(DeviceHandler deviceHandler){
        DeviceHandlerRunable deviceHandlerRunable=new DeviceHandlerRunable();
        deviceHandlerRunable.setDeviceHandler(deviceHandler);
        singleThreadExecutor.execute(deviceHandlerRunable);
    }
    public static void runDeviceHandler(DoHandlerRunable doHandlerRunable){
        singleThreadExecutor.execute(doHandlerRunable);
    }

    /**
     * 抽象类
     */
    public abstract static class DoHandlerRunable implements Runnable{
        abstract void handle();

        @Override
        public void run() {
            this.handle();
        }
    }

    /**
     * 具体任务类
     */
    @Data
     private static class DeviceHandlerRunable extends DeviceThreadExecutor.DoHandlerRunable{
        private DeviceHandler deviceHandler;


         @Override
         public void handle() {
             /**
              *前置任务判断
              */
             if(this.deviceHandler==null){
                 return;
             }
             log.info("begin try updated:{}",this.deviceHandler);
             if(this.deviceHandler.getNumbers()>MAX_NUMBERS){
                 log.error("retry greater than  max times:{},now return",this.deviceHandler);
                 return;
             }
             if(this.deviceHandler.getUpdated()>0){
                 return;
             }
             /**
              * 默认休眠1秒，因es默认1秒将数据从内存写入磁盘
              */
             Long sleepTime=1000L;
             try {
                 Thread.sleep(sleepTime);
             } catch (InterruptedException e) {
                 log.error("DeviceHandlerRunable InterruptedException error:{}",this.deviceHandler,e);
                 return;
             }

             UpdateByQueryRequestBuilder updateByQueryRequestBuilder = this.deviceHandler.getUpdateByQueryRequestBuilder();
             try{
                 this.deviceHandler.updated= updateByQueryRequestBuilder.get().getUpdated();
                 if(this.deviceHandler.updated==0){
                     //未更新成功
                     this.deviceHandler.setNumbers(this.deviceHandler.getNumbers()+1);
                     if(this.deviceHandler.getNumbers()>MAX_NUMBERS){
                         log.error("retry greater than  max times:{},now return",this.deviceHandler);
                         return;
                     }
                     //重新丢入线程池进行尝试更新
                     DeviceThreadExecutor.runDeviceHandler(this);
                 }else{
                     log.info("update success:{} ",this.deviceHandler);
                     return;
                 }
             }catch (Exception e){
                 log.error("update try error:{}",this.deviceHandler,e);
                 //更新报错，次数+1
                 this.deviceHandler.setNumbers(this.deviceHandler.getNumbers()+1);
                 //重新丢入线程池进行尝试更新
                 DeviceThreadExecutor.runDeviceHandler(this);
             }
         }
     }
    //判断线程池是否已经执行结束，test用
     public static boolean isFinsh(){
        return ((ThreadPoolExecutor)singleThreadExecutor).getActiveCount()==0;
     }
}
```

单元测试代码如下

```java
@Test
    public void EsVersionTest2() throws Exception {

        String indexName=EsIndexConstants.IiapIndexConsts.IMPL_DEVICE_INDEX;
        BoolQueryBuilder idQuery = QueryBuilders.boolQuery();
        idQuery.must(QueryBuilders.termQuery("deviceId", "111111"));



        Map<String,Long> map=new HashMap<>();
        for(int i=0;i<10;i++){
            Map<String,Object> params=new HashMap<>();
            params.put("deleted","1");
            params.put("updatedTime",Long.toString(System.currentTimeMillis()));
            params.put("updatedBy","T"+i);
            Script script2= new Script(ScriptType.INLINE, "painless","ctx._source.deleted=params.deleted" +
                    ";ctx._source.updatedTime=params.updatedTime"+
                    ";ctx._source.updatedBy=ctx._source.updatedBy+'+'+params.updatedBy",params);
            UpdateByQueryRequestBuilder updateByQuery2 = UpdateByQueryAction.INSTANCE.newRequestBuilder(elasticsearchTemplate.getClient());
            updateByQuery2.source(indexName)
                    .filter(idQuery)
                    .script(script2);
            long updated2 = updateByQuery2.get().getUpdated();
            //System.out.println(String.format("update success:%s-%s",idQuery,updated2));
            map.put("t"+i,updated2);

            if(updated2==0){
                DeviceThreadExecutor.DeviceHandler deviceHandler=new DeviceThreadExecutor.DeviceHandler();
                deviceHandler.setUpdateByQueryRequestBuilder(updateByQuery2);
                deviceHandler.setUpdated(0L);
                DeviceThreadExecutor.runDeviceHandler(deviceHandler);
            }
            Thread.sleep(1L);

        }
        log.info("map 2 :{}",map);
        //等待线程池任务全部结束
        while(!DeviceThreadExecutor.isFinsh()){
            Thread.sleep(1000L);
        }
        log.info("sssssssssssss");
    }
```

![test结果](/intro/0071.png)

未完待续... 