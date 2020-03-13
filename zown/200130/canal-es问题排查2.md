---
title: 通过一次cannal监听问题浅读spring-dada源码及其在es中的应用二
date: 2020-01-18 16:54:13
tags: 
- es
- spring
categories: es
---

# 接上文

​	可以看到queries在new的时候初始化，将repository中需要代理的方法和具体实现类也就是RepositoryQuery实现类放入其中，然后再拦截器中判断并执行实现的方法即：

![RepositoryQuery](/intro/0060.png)

```java
//org.springframework.data.elasticsearch.repository.query.ElasticsearchPartQuery#execute
public Object execute(Object[] parameters) {
    //封装自己的查询参数
        ParametersParameterAccessor accessor = new ParametersParameterAccessor(this.queryMethod.getParameters(), parameters);
    //生成自己的查询条件   
    CriteriaQuery query = this.createQuery(accessor);
    //进行具体调用哪种方法的判断
        if (this.tree.isDelete()) {
            //具体实现方法调用，也就是ElasticsearchTemplate内的方法
            Object result = this.countOrGetDocumentsForDelete(query, accessor);
            this.elasticsearchOperations.delete(query, this.queryMethod.getEntityInformation().getJavaType());
            return result;
        } else if (this.queryMethod.isPageQuery()) {
            query.setPageable(accessor.getPageable());
            return this.elasticsearchOperations.queryForPage(query, this.queryMethod.getEntityInformation().getJavaType());
        } else if (this.queryMethod.isStreamQuery()) {
            Class<?> entityType = this.queryMethod.getEntityInformation().getJavaType();
            if (query.getPageable().isUnpaged()) {
                int itemCount = (int)this.elasticsearchOperations.count(query, this.queryMethod.getEntityInformation().getJavaType());
                query.setPageable(PageRequest.of(0, Math.max(1, itemCount)));
            }

            return StreamUtils.createStreamFromIterator(this.elasticsearchOperations.stream(query, entityType));
        } else if (this.queryMethod.isCollectionQuery()) {
            if (accessor.getPageable() == null) {
                int itemCount = (int)this.elasticsearchOperations.count(query, this.queryMethod.getEntityInformation().getJavaType());
                query.setPageable(PageRequest.of(0, Math.max(1, itemCount)));
            } else {
                query.setPageable(accessor.getPageable());
            }

            return this.elasticsearchOperations.queryForList(query, this.queryMethod.getEntityInformation().getJavaType());
        } else {
            return this.tree.isCountProjection() ? this.elasticsearchOperations.count(query, this.queryMethod.getEntityInformation().getJavaType()) : this.elasticsearchOperations.queryForObject(query, this.queryMethod.getEntityInformation().getJavaType());
        }
    }
```

看到这里终于简单明白了spring-data的拦截和封装流程！而我们最开始提到的那个问题也在这个方法里找到了答案

```java
//org.springframework.data.elasticsearch.core.ElasticsearchTemplate#queryForObject(org.springframework.data.elasticsearch.core.query.CriteriaQuery, java.lang.Class<T>)
public <T> T queryForObject(CriteriaQuery query, Class<T> clazz) {
        Page<T> page = this.queryForPage(query, clazz);
    //
        Assert.isTrue(page.getTotalElements() < 2L, "Expected 1 but found " + page.getTotalElements() + " results");
        return page.getTotalElements() > 0L ? page.getContent().get(0) : null;
    }
```

原来是数据重复了，删掉重复的数据或者按照查询集合的方法规则（如下，returnType为集合类型即可）即来写即可！~

```java
//org.springframework.data.repository.query.QueryMethod#isCollectionQuery
public boolean isCollectionQuery() {

		if (isPageQuery() || isSliceQuery()) {
			return false;
		}

		Class<?> returnType = method.getReturnType();

		if (QueryExecutionConverters.supports(returnType) && !QueryExecutionConverters.isSingleValue(returnType)) {
			return true;
		}

		if (QueryExecutionConverters.supports(unwrappedReturnType)) {
			return !QueryExecutionConverters.isSingleValue(unwrappedReturnType);
		}

		return ClassTypeInformation.from(unwrappedReturnType).isCollectionLike();
	}
```

附注：大部分的判断到底是删除还是查询还是更新等的标准

```java
//变量注释法，不用再解释啦
private static final String QUERY_PATTERN = "find|read|get|query|stream";
	private static final String COUNT_PATTERN = "count";
	private static final String EXISTS_PATTERN = "exists";
	private static final String DELETE_PATTERN = "delete|remove";

private static final String DISTINCT = "Distinct";

//count
		private static final Pattern COUNT_BY_TEMPLATE = Pattern.compile("^count(\\p{Lu}.*?)??By");

		private static final Pattern EXISTS_BY_TEMPLATE = Pattern.compile("^(" + EXISTS_PATTERN + ")(\\p{Lu}.*?)??By");
		private static final Pattern DELETE_BY_TEMPLATE = Pattern.compile("^(" + DELETE_PATTERN + ")(\\p{Lu}.*?)??By");
		private static final String LIMITING_QUERY_PATTERN = "(First|Top)(\\d*)?";
		private static final Pattern LIMITED_QUERY_TEMPLATE = Pattern
				.compile("^(" + QUERY_PATTERN + ")(" + DISTINCT + ")?" + LIMITING_QUERY_PATTERN + "(\\p{Lu}.*?)??By");
```

