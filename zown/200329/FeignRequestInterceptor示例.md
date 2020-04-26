---
title: FeignRequestInterceptor示例
date: 2020-02-08 20:04:10
tags: feign
categories: 
- springboot
- feign
---

背景

项目中A服务需要频繁调用B服务，且参数内有许多共同属性，所以考虑利用feign的拦截器来进行统一参数的获取和封装，减少调用B服务前重复代码的编写

示例

直接上代码，后续遇到相同场景可以参考参考

```java
@Component
@Slf4j
public class LeisiRequestInterceptor implements RequestInterceptor, InitializingBean {

    private IiapBaseManagerImpl iiapBaseManagerImpl;
    @Autowired
    private VersionCacheUtil versionCacheUtil;

    @Autowired
    private ApplicationContext applicationContext;

    @Override
    public void apply(RequestTemplate template) {
        if(iiapBaseManagerImpl==null){
            //如果实现类,则直接返回
            return;
        }
        String url=template.url();
        //只会拦截去指定地址的请求
        if(LeisiConstants.needFill(url)){
            if(template!=null){
                log.info("start leisi client interceptor----- {}",template);
                fillVersionByRequest(template);
                log.info("end leisi client interceptor----- ");
            }
        }
        return;
    }

    private void fillVersionByRequest(RequestTemplate template) {
        String method=template.method();

        String  projectId,schemaId,roleCode,userId;
        if("POST".equals(method)){
            JSONObject jsonObject=(JSONObject)JSONObject.parse(template.body());
            if(jsonObject==null){
                return ;
            }
            //post则取body
            projectId=jsonObject.getString("projectId");
            schemaId=jsonObject.getString("schemaId");
            roleCode=jsonObject.getString("roleCode");
            userId=jsonObject.getString("userId");

        }else{
            //get则取query里的
            Map<String, Collection<String>> queries= template.queries();
            if(MapUtils.isEmpty(queries)){
                return;
            }
            //get则取param
            projectId=queries.get("projectId")==null?"":(String)queries.get("projectId").toArray()[0];
            schemaId=queries.get("schemaId")==null?"":(String)queries.get("schemaId").toArray()[0];
            roleCode=queries.get("roleCode")==null?"":(String)queries.get("roleCode").toArray()[0];
            userId=queries.get("userId")==null?"":(String)queries.get("userId").toArray()[0];
        }



        UserVersionDto dto=new UserVersionDto();
        
        if(StringUtils.isEmpty(schemaId)){
            if(StringUtils.isEmpty(projectId)){
                return ;
            }
            if(StringUtils.isEmpty(roleCode) || StringUtils.isEmpty(userId)){
                dto=iiapBaseManagerImpl.getVersionsByProjectId(projectId);
            }else{

                dto.setProjectId(projectId);
                dto.setRoleCode(roleCode);
                dto.setUserId(userId);
                iiapBaseManagerImpl.getVersionsByDto(dto);
            }
        }else{
            if( StringUtils.isEmpty(projectId)){
                return ;
            }
            VersionDTO vdto=versionCacheUtil.getVersion(projectId,schemaId);
            dto.setProjectId(projectId);
            List<String> scs=new ArrayList();
            scs.add(schemaId);
            Map<String,VersionDTO> ms=new HashMap();
            ms.put(schemaId,vdto);

            dto.setSchemaIds(scs);
            dto.setVersionMaps(ms);
        }
        if(dto!=null){
            if("POST".equals(method)){
                //重新封装body
                JSONObject obj=(JSONObject)JSONObject.parse(template.body());
                obj.put("userVersionDto",dto);
                template.body(JSONObject.toJSONString(obj));
                log.info("new body：{}", template);
            }else{
                String dtos ,charSet=template.charset()==null ?"UTF-8":template.charset().name();
                try {
                    dtos = URLEncoder.encode(JSONObject.toJSONString(dto),charSet);
                } catch (UnsupportedEncodingException e) {
                    log.error("leisi request interceptor encode error{}",JSONObject.toJSONString(dto),e);
                    return;
                }
                //重新封装params
                Map<String, Collection<String>> queries= template.queries();
                Map<String, Collection<String>> newQy=new HashMap();
                newQy.put("userVersionDto",new ArrayList<String>(){{add(dtos);}});
                newQy.putAll(queries);

                template.queries(newQy);
                log.info("new queries：{}", template);
            }
        }
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Map<String,IiapBaseManagerImpl> mp=applicationContext.getBeansOfType(IiapBaseManagerImpl.class);
        if(MapUtils.isNotEmpty(mp)){
            this.iiapBaseManagerImpl= ((Map.Entry<String,IiapBaseManagerImpl>)(mp.entrySet().toArray()[0])).getValue();
        }
    }
}
```

