---
title: spring-data-es抽象service集成简单增删查改之四
date: 2020-03-07 10:17:02
tags: elasticsearch
categories: elasticsearch
---

## 写在前面

本次主要是补充了上面两次的单元测试代码，以保障组件功能的正常运行

## 代码

注释已经比较清楚了，不在此赘述

```java
//doc
@Document(indexName = "t1",type = "t2", shards = 2, replicas = 1)
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
@ToString
public class IiapTemplateMatchDeviceDoc extends BaseDoc {
    @Id
    private String id;
    private String subjectModel;
    private String subjectName;
    private String templatePositionId;
    private String templatePositionName;
    private String templateGroupId;
    private String projectId;
    private String schemaId;
    private String templateDeviceId;
    private String templateDeviceSubjectId;
    private String manualMatchId;
    private String matchSpaceSubjectId;
    private String matchSpaceSubjectName;
    private Integer matchStatus;
    private String categoryName;
    private String categoryValues;
}

//dto
import lombok.Data;

@Data
public class IiapTemplateMatchDeviceDto extends BaseDto {
    private String subjectModel;
    private String subjectName;
    private String templatePositionId;
    private String templatePositionName;
    private String projectId;
    private String schemaId;
    private String templateDeviceId;
    private String templateDeviceSubjectId;
    private String manualMatchId;
    private String matchSpaceSubjectId;
    private String matchSpaceSubjectName;
    private Integer matchStatus;
    private String categoryName;
    private String categoryValues;
}
//Repository
/**
 * @author yyt
 */
public interface IiapImplTemplateMatchDeviceRepository extends ElasticsearchRepository<IiapTemplateMatchDeviceDoc, String> {
    IiapTemplateMatchDeviceDoc findByManualMatchIdAndTemplateDeviceSubjectId(String manualMatchId, String templateDeviceSubjectId);
}

//doc service
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.springframework.stereotype.Service;

@Slf4j
@Service("iiapDocMatchDeviceService")
public class IiapDocMatchDeviceServiceImpl extends EsAbstractDocService<IiapImplTemplateMatchDeviceRepository,IiapTemplateMatchDeviceDoc> {

    @Override
    protected void prepareSave(IiapTemplateMatchDeviceDoc iiapTemplateMatchDeviceDoc) {

    }

    @Override
    public <E extends BaseDto> void buildOwnQuery(QueryBuilder qbs, E s) {
        IiapTemplateMatchDeviceDto dto=(IiapTemplateMatchDeviceDto)s;
        if(StringUtils.isNotBlank(dto.getSubjectName())){
            ((BoolQueryBuilder)qbs).must(QueryBuilders.matchPhraseQuery("subjectName", dto.getSubjectName()));
        }
    }
}

//template service
import lombok.extern.slf4j.Slf4j;
import org.elasticsearch.index.query.QueryBuilder;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class IiapTemplateMatchDeviceService extends EsAbstractTemplateService<IiapTemplateMatchDeviceDoc> {

    @Override
    protected void prepareSave(IiapTemplateMatchDeviceDoc s) {

    }

    @Override
    public <E extends BaseDto> void buildOwnQuery(QueryBuilder qbs, E s) {

    }

    @Override
    protected int getBatchSize(){
        return 1;
    }
}

//base test & resource & junit test sample
/**
 * es test
 * 单单从spring-data-es的模块进行测试即可，无需依赖springBoot开始测试
 * @author yyt
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:/es-test.xml")
public class SpringBaseTest {
    protected static final Logger log = LoggerFactory.getLogger(SpringBaseTest.class);

    @Autowired
    private IiapImplTemplateMatchDeviceRepository iiapImplTemplateMatchDeviceRepository;


    @BeforeClass
    public static void setUpBeforeClass() throws Exception {
        System.out.println("==============单元测试开始=============");
    }

    @AfterClass
    public static void tearDownAfterClass() throws Exception {
        System.out.println("==============单元测试结束=============");
    }

    @Test
    public void baseTest(){
        System.out.println("success");
    }
}


import javax.annotation.Resource;
import java.util.List;
import java.util.Map;

/**
 * doc service test
 * @author yyt
 */
public class EsDocServiceTest extends SpringBaseTest{

    @Resource(name="iiapDocMatchDeviceService")
    private EsAbstractDocService iiapDocMatchDeviceService;

    @Test
    public void saveOrUpdateTest(){
        IiapTemplateMatchDeviceDoc doc=new IiapTemplateMatchDeviceDoc();
        doc.setManualMatchId("s.123456");
        doc.setCategoryName("网关");
        doc.setCategoryValues("1");
        doc.setMatchSpaceSubjectId("sp.26");
        doc.setMatchSpaceSubjectName("门窗传感器");
        doc.setSchemaId("sc.123456789");
        doc.setProjectId("pj.123456789");
        doc.setCreatedBy("test");
        doc.setCreatedTime(System.currentTimeMillis());
        doc.setSubjectName("docTest-"+System.currentTimeMillis());
        doc.setDeleted(BaseConstant.FILED_NO_DELETED);
        iiapDocMatchDeviceService.saveOrUpdate(doc);
        String id=doc.getId();
        Assert.assertNotNull(id);

        doc.setSubjectName("docTest-change-"+System.currentTimeMillis());
        iiapDocMatchDeviceService.saveOrUpdate(doc);
        String nid=doc.getId();
        Assert.assertNotNull(nid);
        Assert.assertTrue(id.equals(nid));
    }

    @Test
    public void searchTest(){
        saveOrUpdateTest();
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("docTest");
        Iterable result= iiapDocMatchDeviceService.search(dto);
        Assert.assertNotNull(result);
        Assert.assertTrue(!IterableUtils.isEmpty(result));
    }

    @Test
    public void searchByPageTest(){
        saveOrUpdateTest();
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("docTest");
        dto.setStartIndex(0);
        dto.setSize(1);
        Map<String,Object> result= iiapDocMatchDeviceService.searchByPage(dto);
        Object obj=result.get("data");
        Assert.assertNotNull(obj);
        Assert.assertTrue(!CollectionUtils.isEmpty((List)obj));
    }

    @Test
    public void deleteTest(){
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("docTest");
        Iterable<IiapTemplateMatchDeviceDoc> result= iiapDocMatchDeviceService.search(dto);
        Assert.assertNotNull(result);
        Assert.assertTrue(!IterableUtils.isEmpty(result));
        result.forEach(e->{
            iiapDocMatchDeviceService.delete(e.getId());
        });
        result= iiapDocMatchDeviceService.search(dto);
        Assert.assertTrue(IterableUtils.isEmpty(result));
    }


}

/**
 * template service test
 * @author yyt
 */
public class EsTemplateServiceTest extends SpringBaseTest{

    @Resource(name="iiapTemplateMatchDeviceService")
    private EsAbstractTemplateService iiapTemplateMatchDeviceService;

    @Test
    public void saveOrUpdateTest(){
        for(int i=0;i<10;i++){
            IiapTemplateMatchDeviceDoc doc=new IiapTemplateMatchDeviceDoc();
            doc.setManualMatchId("s.123456");
            doc.setCategoryName("网关");
            doc.setCategoryValues("1");
            doc.setMatchSpaceSubjectId("sp.26");
            doc.setMatchSpaceSubjectName("门窗传感器");
            doc.setSchemaId("sc.123456789");
            doc.setProjectId("pj.123456789");
            doc.setCreatedBy("test");
            doc.setCreatedTime(System.currentTimeMillis());
            doc.setSubjectName("templateTest-"+System.currentTimeMillis());
            doc.setDeleted(BaseConstant.FILED_NO_DELETED);
            iiapTemplateMatchDeviceService.saveOrUpdate(doc);
            String id=doc.getId();
            Assert.assertNotNull(id);

            doc.setSubjectName("templateTest-change-"+System.currentTimeMillis());
            iiapTemplateMatchDeviceService.saveOrUpdate(doc);
            String nid=doc.getId();
            Assert.assertNotNull(nid);
            Assert.assertTrue(id.equals(nid));
        }
    }

    @Test
    public void searchTest(){
        saveOrUpdateTest();
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("templateTest");
        Iterable<IiapTemplateMatchDeviceDoc> result= iiapTemplateMatchDeviceService.search(dto);
        Assert.assertNotNull(result);
        Assert.assertTrue(!IterableUtils.isEmpty(result));
        result.forEach(e->{
            Assert.assertNotNull(e.getId());
        });
    }

    @Test
    public void searchByPageTest(){
        saveOrUpdateTest();
        SortBy sortBy=new SortBy();
        //按照updateTime倒序
        sortBy.setField(SortFieldEnum.SortFieldEnum2.getField());
        sortBy.setType(SortTypeEnum.SortTypeEnum1.getField());
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSortBy(sortBy);
        dto.setSubjectName("templateTest");
        dto.setStartIndex(0);
        dto.setSize(1);
        Map<String,Object> result= iiapTemplateMatchDeviceService.searchByPage(dto);
        List<IiapTemplateMatchDeviceDoc>  obj=(List<IiapTemplateMatchDeviceDoc>)result.get("data");
        Assert.assertNotNull(obj);
        Assert.assertTrue(!CollectionUtils.isEmpty(obj));
        String id1=obj.get(0).getId();
        Long time1=obj.get(0).getCreatedTime();

        dto.setStartIndex(1);
        dto.setSize(1);
        result= iiapTemplateMatchDeviceService.searchByPage(dto);
        obj=(List<IiapTemplateMatchDeviceDoc>)result.get("data");
        Assert.assertNotNull(obj);
        Assert.assertTrue(!CollectionUtils.isEmpty(obj));
        String id2=obj.get(0).getId();
        Long  time2=obj.get(0).getCreatedTime();

        dto.setStartIndex(2);
        dto.setSize(1);
        result= iiapTemplateMatchDeviceService.searchByPage(dto);
        obj=(List<IiapTemplateMatchDeviceDoc>)result.get("data");
        Assert.assertNotNull(obj);
        Assert.assertTrue(!CollectionUtils.isEmpty(obj));
        String id5=obj.get(0).getId();
        Long  time5=obj.get(0).getCreatedTime();


        dto.setStartIndex(3);
        dto.setSize(2);
        result= iiapTemplateMatchDeviceService.searchByPage(dto);
        obj=(List<IiapTemplateMatchDeviceDoc>)result.get("data");
        Assert.assertNotNull(obj);
        Assert.assertTrue(!CollectionUtils.isEmpty(obj));
        String id3=obj.get(0).getId();
        String id4=obj.get(1).getId();
        Long time3=obj.get(0).getCreatedTime();
        Long time4=obj.get(1).getCreatedTime();

        Boolean timeO=time1>time2 && time2>time3 && time3>time4;
        Boolean idO=!id1.equals(id2)&& !id2.equals(id3)&& !id3.equals(id4);
        Assert.assertTrue(timeO);
        Assert.assertTrue(idO);
        return;
    }

    @Test
    public void deleteTest() throws InterruptedException {
        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("templateTest");
        Iterable<IiapTemplateMatchDeviceDoc> result= iiapTemplateMatchDeviceService.search(dto);
        Assert.assertNotNull(result);
        Assert.assertTrue(!IterableUtils.isEmpty(result));
        result.forEach(e->{
            iiapTemplateMatchDeviceService.delete(e.getId());
        });
        //es默认1秒刷新至硬盘,为测试准确，休眠1秒
        Thread.sleep(1000);

        result= iiapTemplateMatchDeviceService.search(dto);
        Assert.assertTrue(IterableUtils.isEmpty(result));
    }

    /**
     * deleteByQueryRequest内引用了updateByQueryRequest，相当于一起测试了
     * @throws InterruptedException
     */
    @Test
    public void deleteByQueryRequestTest() throws InterruptedException {

        IiapTemplateMatchDeviceDto dto= new IiapTemplateMatchDeviceDto();
        dto.setSubjectName("templateTest");
        Iterable<IiapTemplateMatchDeviceDoc> result= iiapTemplateMatchDeviceService.search(dto);
        Assert.assertNotNull(result);
        Assert.assertTrue(!IterableUtils.isEmpty(result));

        iiapTemplateMatchDeviceService.deleteByQueryRequest(dto);

        //es默认1秒刷新至硬盘,为测试准确，休眠1秒
        Thread.sleep(1000);

        result= iiapTemplateMatchDeviceService.search(dto);
        Assert.assertTrue(IterableUtils.isEmpty(result));
    }

}
```

补充resource：es-test.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:elasticsearch="http://www.springframework.org/schema/data/elasticsearch"
       xsi:schemaLocation="http://www.springframework.org/schema/data/elasticsearch https://www.springframework.org/schema/data/elasticsearch/spring-elasticsearch.xsd
		http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">


    <elasticsearch:repositories base-package="com.yyt.aiot.cloud.test.repository"/>

    <elasticsearch:transport-client id="client" cluster-nodes="10.0.14.50:9300" cluster-name="1-application" />

    <bean name="elasticsearchTemplate" class="org.springframework.data.elasticsearch.core.ElasticsearchTemplate">
        <constructor-arg name="client" ref="client"/>
    </bean>

    <bean name="iiapDocMatchDeviceService" class="com.yyt.aiot.cloud.test.service.doc.IiapDocMatchDeviceServiceImpl"></bean>
    <bean name="iiapTemplateMatchDeviceService" class="com.yyt.aiot.cloud.test.service.template.IiapTemplateMatchDeviceService"></bean>
</beans>
```

