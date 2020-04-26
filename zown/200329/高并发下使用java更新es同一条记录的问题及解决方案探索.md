---
title: 高并发下使用java更新es同一条记录的问题及解决方案探索一
date: 2020-03-14 12:37:05
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

生产上在高并发使用java的UpdateByQueryRequest对es进行update的时候（与java线程无关，UpdateByQueryRequest内部本身就是异步请求）会出现一个情况：第一次更新成功，后续短时间内的更新条数都是0

## 分析

后来根据查阅资料发现因为如下原因：

​	es内部为了防止出现并发操作的脏写情况出现，使用了_version的乐观锁控制。

​	_version元数据
​	第一次创建一个document的时候，它的_version内部版本号就是1；以后，每次对这个document执行修改或者删除操作，都会对
​	这个_version版本号自动加1；哪怕是删除

​	在删除一个document后，他不是立即物理删除掉的，因为它的一些版本号等信息还是保留的，先删除一条document，再重新创建
​	这条document，其实会在delete version基础之上，再把version+1

我们可以使用如下代码进行测试：

```java
//多线程测试
@Test
    public void EsVersionTest() throws Exception {

        String indexName=EsIndexConstants.IiapIndexConsts.IMPL_DEVICE_INDEX;
        BoolQueryBuilder idQuery = QueryBuilders.boolQuery();
        idQuery.must(QueryBuilders.termQuery("deviceId", "11111111"));

        Map<String,Long> map=new HashMap<>();

        Thread t3=new Thread(new Runnable() {
            @Override
            public void run() {
                Script script2= new Script("ctx._source.deleted='"+1+"'" +
                        ";ctx._source.updatedTime='"+System.currentTimeMillis()+"'"+
                        ";ctx._source.updatedBy='canalTest3'");
                UpdateByQueryRequestBuilder updateByQuery2 = UpdateByQueryAction.INSTANCE.newRequestBuilder(elasticsearchTemplate.getClient());
                updateByQuery2.source(indexName).abortOnVersionConflict(false)
                        .filter(idQuery)
                        .script(script2);
                long updated2 = updateByQuery2.get().getUpdated();
                //System.out.println(String.format("update success:%s-%s",idQuery,updated2));
                map.put("t3",updated2);
            }
        },"t3");

        Thread t4=new Thread(new Runnable() {
            @Override
            public void run() {
                Script script= new Script("ctx._source.deleted='"+1+"'" +
                        ";ctx._source.updatedTime='"+System.currentTimeMillis()+"'"+
                        ";ctx._source.updatedBy='canalTest4'");
                UpdateByQueryRequestBuilder updateByQuery = UpdateByQueryAction.INSTANCE.newRequestBuilder(elasticsearchTemplate.getClient());
                updateByQuery.source(indexName).abortOnVersionConflict(false)
                        .filter(idQuery)
                        .script(script);
                long updated = updateByQuery.get().getUpdated();
                map.put("t4",updated);
            }
        },"t4");

        t3.start();
        t4.start();
        t3.join();
        t4.join();
        log.info("map 2 :{}",map);
        log.info("ssssssssssss");
    }

//单线程测试
    @Test
    public void EsVersionTest2() throws Exception {

        String indexName=EsIndexConstants.IiapIndexConsts.IMPL_DEVICE_INDEX;
        BoolQueryBuilder idQuery = QueryBuilders.boolQuery();
        idQuery.must(QueryBuilders.termQuery("deviceId", "1111111"));



        Map<String,Long> map=new HashMap<>();
        for(int i=0;i<100;i++){
            Map<String,Object> params=new HashMap<>();
            params.put("deleted","1");
            params.put("updatedTime",Long.toString(System.currentTimeMillis()));
            params.put("updatedBy","T"+i);
            Script script2= new Script(ScriptType.INLINE, "painless","ctx._source.deleted=params.deleted" +
                    ";ctx._source.updatedTime=params.updatedTime"+
                    ";ctx._source.updatedBy=params.updatedBy",params);
            UpdateByQueryRequestBuilder updateByQuery2 = UpdateByQueryAction.INSTANCE.newRequestBuilder(elasticsearchTemplate.getClient());
            updateByQuery2.source(indexName)
                    .filter(idQuery)
                    .script(script2);
            long updated2 = updateByQuery2.get().getUpdated();
            //System.out.println(String.format("update success:%s-%s",idQuery,updated2));
            map.put("t3",updated2);

            if(updated2==0){
                DeviceThreadExecutor.DeviceHandler deviceHandler=new DeviceThreadExecutor.DeviceHandler();
                deviceHandler.setUpdateByQueryRequestBuilder(updateByQuery2);
                deviceHandler.setUpdated(0L);
                DeviceThreadExecutor.runDeviceHandler(deviceHandler);
            }
            //map.put("t4",updated);

            log.info("map 2 :{}",map);
        }
        log.info("sssssssssssss");
    }
```

执行后可以看到，第一次update成功后，后续的更新都没有成功。

![test1结果](/intro/0069.png)

![test2结果](/intro/0070.png)

未完待续... 