---
title: Springboot test
date: 2019-11-23 15:32:10
tags: Springboot
categories: Springboot
---

### 写在前面之为什么要写测试用例

-  可以避免测试点的遗漏，为了更好的进行测试，可以提高测试效率 
-  可以自动测试，可以在项目打包前进行测试校验 
-  可以及时发现因为修改代码导致新的问题的出现，并及时解决 

这是TDD中最重要的一环。

### 正文

在springboot中引入springboot-test需要以下几个步骤：

1. **pom中引入 spring-boot-starter-test**

   ![pom](/intro/0041.png)

   如果需要打包是不运行test，则可以再pom中加入如下配置

   ```xml
   <!--打包跳过测试-->
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-surefire-plugin</artifactId>
       <configuration>
           <skip>true</skip>
       </configuration>
   </plugin>
   ```

2. **测试springboot Testing Spring Boot applications**

   Spring Boot提供了一个@SpringBootTest注解，当您需要Spring Boot功能时，它可以用作标准spring-test @ContextConfiguration注释的替代方法。注解的工作原理是通过SpringApplication在测试中创建ApplicationContext。

   示例代码如下：

   ```java
   /**
    * @Title: 基础测试类(其他需要环境进行测试的只需要继承此类即可)
    */
   @RunWith(SpringRunner.class)
   @SpringBootTest(classes = DemoApplication.class)
   @ActiveProfiles("test")
   public abstract class BaseTest {
   	
   }
   /**
    * @Title: 其他业务测试类
    * @Desc: 都可以继承BaseTest, 构建的时候就可以共用一个环境避免每次加载上下文, 以节约时间
    */
   public class DemoTest extends BaseTest {
   
   	/**
   	 * 打包时不测试此方法
   	 */
   	@Test
   	@Ignore
   	public void ignoreTest() {
   	}
   
   	/**
   	 * 打包要测试的方法
   	 * 注: 方法名请不要和其他
   	 * 适用范围: 如一些重点、复杂的查询接口测试
   	 */
   	@Test
   	public void test01() {
   		// 断言可以放测试类里头检验结果,不符合断言条件的会报错,停止打包,如检验状态码/对象等,避免人工输出检查结果,提高自动化测试效率
   		// 是否相等
   		Assert.assertEquals(1, 1);
   		Assert.assertEquals(1, 1);
   		Assert.assertNotEquals(1, 2);
   		// 是否为空
   		Assert.assertNull(null);
   		Assert.assertNotNull("");
   		// 对错判断
   		Assert.assertTrue(1 == 1);
   		Assert.assertFalse(1 == 2);
   	}
   
   	/**
   	 * 写操作事物测试
   	 * Transactional: 测完写数据会回滚的测试用例
   	 */
   	@Test
   	@Transactional
   	public void transactionalTest() {
   	}
   
   }
   ```

   重点在于此：@RunWith(SpringRunner.class)。跟踪源代码可以发现spring test原理上是继承了了junit。

   ![pom](/intro/0042.png)

   可以使用**@SpringBootTest**的**webEnvironment**属性来进一步优化测试的运行方式：

- **MOCK** ： 加载一个WebApplicationContext并提供一个模拟servlet环境。嵌入式servlet容器在使用此注释时不会启动。如果servlet API不在你的类路径上，这个模式将透明地回退到创建一个常规的非web应用程序上下文。可以与@AutoConfigureMockMvc结合使用，用于基于MockMvc的应用程序测试。
- **RANDOM_PORT ：** 加载一个EmbeddedWebApplicationContext并提供一个真正的servlet环境。嵌入式servlet容器启动并在随机端口上侦听。
- **DEFINED_PORT ：** 加载一个EmbeddedWebApplicationContext并提供一个真正的servlet环境。嵌入式servlet容器启动并监听定义的端口（即从application.properties或默认端口8080）。
- **NONE ：** 使用SpringApplication加载ApplicationContext，但不提供任何servlet环境（模拟或其他）。

