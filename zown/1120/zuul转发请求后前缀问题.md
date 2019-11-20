---
title: zuul转发请求后前缀问题
date: 2019-11-16 19:50:25
tags: 
- zuul
- 源码解析
categories: zuul
---

### 示例

zuul作为springcloud的网关使用路由功能是，会默认把路由的前缀去掉

比如路由前：http://127.0.0.1:8181/api/demo/list 

路由后为： http://192.168.1.100:8080/demo/list

如果我们需要让他不去掉api这个前缀,那么需要在对应路由上加上该属性： strip-prefix: false 

### 源码解析

首先我们从最开始加载zuul说起

- @EnableZuulProxy

  最开始的配置日入口是这里

  ```java
  EnableCircuitBreaker
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Import({ZuulProxyMarkerConfiguration.class})
  public @interface EnableZuulProxy {
  }
  ```

  可以看到这里引用了ZuulProxyMarkerConfiguration

- ZuulProxyMarkerConfiguration

  ```java
  @Configuration
  public class ZuulProxyMarkerConfiguration {
      public ZuulProxyMarkerConfiguration() {
      }
  
      @Bean
      public ZuulProxyMarkerConfiguration.Marker zuulProxyMarkerBean() {
          return new ZuulProxyMarkerConfiguration.Marker();
      }
  
      class Marker {
          Marker() {
          }
      }
  }
  ```

  这里仅仅是加载了ZuulProxyMarkerConfiguration.Marker类型的bean，别无他操作

- spring.factories

  在Spring Boot中有一种非常解耦的扩展机制：Spring Factories，所以我们找到jar内的MEAT-INF/spring.factories

  ```java
  //部分内容
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.zuul.ZuulProxyAutoConfiguration
  ```

  这里声明了ZuulProxyAutoConfiguration作为实现类被发现

- ZuulProxyAutoConfiguration

  ```java
  @Configuration
  @Import({RestClientRibbonConfiguration.class, OkHttpRibbonConfiguration.class, HttpClientRibbonConfiguration.class, HttpClientConfiguration.class})
  //重点在这里 Marker的bean被装载，这个bean也会被装载
  @ConditionalOnBean({Marker.class})
  public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {
  .....
  }
  ```

- ZuulServerAutoConfiguration

  ZuulProxyAutoConfiguration的父类为ZuulServerAutoConfiguration，这里面就加载了我们在最前面提到的ZuulProperties

  ```java
  @Configuration
  @EnableConfigurationProperties({ZuulProperties.class})
  @ConditionalOnClass({ZuulServlet.class})
  @ConditionalOnBean({Marker.class})
  @Import({ServerPropertiesAutoConfiguration.class})
  public class ZuulServerAutoConfiguration {
      @Autowired
      protected ZuulProperties zuulProperties;
      ....
     }
  ```

- ZuulProperties

  ```java
  @ConfigurationProperties("zuul")
  public class ZuulProperties {
      public static final List<String> SECURITY_HEADERS = Arrays.asList("Pragma", "Cache-Control", "X-Frame-Options", "X-Content-Type-Options", "X-XSS-Protection", "Expires");
      private String prefix = "";
      //默认为true
      private boolean stripPrefix = true;
      private Boolean retryable = false;
      //加载了路由规则项，项值为ZuulRoute
      private Map<String, ZuulProperties.ZuulRoute> routes = new LinkedHashMap();
      private boolean addProxyHeaders = true;
      private boolean addHostHeader = false;
      private Set<String> ignoredServices = new LinkedHashSet();
      private Set<String> ignoredPatterns = new LinkedHashSet();
      private Set<String> ignoredHeaders = new LinkedHashSet();
      private boolean ignoreSecurityHeaders = true;
      private boolean forceOriginalQueryStringEncoding = false;
      private String servletPath = "/zuul";
      private boolean ignoreLocalService = true;
      private ZuulProperties.Host host = new ZuulProperties.Host();
      private boolean traceRequestBody = true;
      private boolean removeSemicolonContent = true;
      private Set<String> sensitiveHeaders = new LinkedHashSet(Arrays.asList("Cookie", "Set-Cookie", "Authorization"));
      private boolean sslHostnameValidationEnabled = true;
      private ExecutionIsolationStrategy ribbonIsolationStrategy;
      private ZuulProperties.HystrixSemaphore semaphore;
      private ZuulProperties.HystrixThreadPool threadPool;
  
      public ZuulProperties() {
          this.ribbonIsolationStrategy = ExecutionIsolationStrategy.SEMAPHORE;
          this.semaphore = new ZuulProperties.HystrixSemaphore();
          this.threadPool = new ZuulProperties.HystrixThreadPool();
      }
      ......
      }
  ```

- ZuulRoute

  ```java
  public static class ZuulRoute {
          private String id;
          private String path;
          private String serviceId;
          private String url;
          //默认为true
          private boolean stripPrefix = true;
          private Boolean retryable;
          private Set<String> sensitiveHeaders = new LinkedHashSet();
          private boolean customSensitiveHeaders = false;
          .......
      }
  ```

- zuulController

  ```
  //在org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration中加载了
  zuulController
  @Bean
  public ZuulController zuulController() {
  return new ZuulController();
  }
  
  
  --------------------------
  public class ZuulController extends ServletWrappingController {
      public ZuulController() {
      //设置ServletClass为ZuulServlet
          this.setServletClass(ZuulServlet.class);
          this.setServletName("zuul");
          this.setSupportedMethods((String[])null);
      }
    }
  ```

- ZuulServlet

  zuul核心。用来拦截指定url，进行filter拦截处理

  ```java
  public class ZuulServlet extends HttpServlet {
      private static final long serialVersionUID = -3374242278843351500L;
      private ZuulRunner zuulRunner;
  
      public ZuulServlet() {
      }
  
      public void init(ServletConfig config) throws ServletException {
          super.init(config);
          String bufferReqsStr = config.getInitParameter("buffer-requests");
          boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true");
          this.zuulRunner = new ZuulRunner(bufferReqs);
      }
  
      public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
          try {
              //初始化RequestContext，供后续拦截器使用
              this.init((HttpServletRequest)servletRequest, (HttpServletResponse)servletResponse);
              RequestContext context = RequestContext.getCurrentContext();
              context.setZuulEngineRan();
  			//preRoute
              try {
                  this.preRoute();
              } catch (ZuulException var12) {
                  this.error(var12);
                  this.postRoute();
                  return;
              }
  			
              try {
                  this.route();
              } catch (ZuulException var13) {
                  this.error(var13);
                  this.postRoute();
                  return;
              }
  
              try {
                  this.postRoute();
              } catch (ZuulException var11) {
                  this.error(var11);
              }
          } catch (Throwable var14) {
              this.error(new ZuulException(var14, 500, "UNHANDLED_EXCEPTION_" + var14.getClass().getName()));
          } finally {
              RequestContext.getCurrentContext().unset();
          }
      }
  
      void postRoute() throws ZuulException {
          this.zuulRunner.postRoute();
      }
  
      void route() throws ZuulException {
          this.zuulRunner.route();
      }
  
      void preRoute() throws ZuulException {
          this.zuulRunner.preRoute();
      }
  
      void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
          this.zuulRunner.init(servletRequest, servletResponse);
      }
  
      void error(ZuulException e) {
          RequestContext.getCurrentContext().setThrowable(e);
          this.zuulRunner.error();
      }
  
      @RunWith(MockitoJUnitRunner.class)
      public static class UnitTest {
          @Mock
          HttpServletRequest servletRequest;
          @Mock
          HttpServletResponseWrapper servletResponse;
          @Mock
          FilterProcessor processor;
          @Mock
          PrintWriter writer;
  
          public UnitTest() {
          }
  
          @Before
          public void before() {
              MockitoAnnotations.initMocks(this);
          }
  }
  ```

- ZuulFilterConfiguration

  ```java
  //同样在：org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration
  @Configuration
      protected static class ZuulFilterConfiguration {
          //自动装载所有ZuulFilter实现的bean
          @Autowired
          private Map<String, ZuulFilter> filters;
  
          protected ZuulFilterConfiguration() {
          }
  
          @Bean
          public ZuulFilterInitializer zuulFilterInitializer(CounterFactory counterFactory, TracerFactory tracerFactory) {
              FilterLoader filterLoader = FilterLoader.getInstance();
              FilterRegistry filterRegistry = FilterRegistry.instance();
              //注册至FilterRegistry，在zuulservlet中被用到
              return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory, filterLoader, filterRegistry);
          }
      }
  
  -------------------------------------
      //com.netflix.zuul.FilterProcessor#runFilters
      public Object runFilters(String sType) throws Throwable {
          if (RequestContext.getCurrentContext().debugRouting()) {
              Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
          }
  
          boolean bResult = false;
          List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
          if (list != null) {
              for(int i = 0; i < list.size(); ++i) {
                  ZuulFilter zuulFilter = (ZuulFilter)list.get(i);
                  Object result = this.processZuulFilter(zuulFilter);
                  if (result != null && result instanceof Boolean) {
                      bResult |= (Boolean)result;
                  }
              }
          }
  
          return bResult;
      }
  ```

  后续就是关于strip-prefix的

- PreDecorationFilter

  PreDecorationFilter是zuul默认的type为pre的拦截器，主要作用为处理上下文请求，获取route等功能

  调用链如下：![调用链](/intro/0040.png)

  ```java
  public class PreDecorationFilter extends ZuulFilter {
      private static final Log log = LogFactory.getLog(PreDecorationFilter.class);
      /** @deprecated */
      @Deprecated
      public static final int FILTER_ORDER = 5;
      private RouteLocator routeLocator;
      private String dispatcherServletPath;
      private ZuulProperties properties;
      private UrlPathHelper urlPathHelper = new UrlPathHelper();
      private ProxyRequestHelper proxyRequestHelper;
  
      public PreDecorationFilter(RouteLocator routeLocator, String dispatcherServletPath, ZuulProperties properties, ProxyRequestHelper proxyRequestHelper) {
          this.routeLocator = routeLocator;
          this.properties = properties;
          this.urlPathHelper.setRemoveSemicolonContent(properties.isRemoveSemicolonContent());
          this.dispatcherServletPath = dispatcherServletPath;
          this.proxyRequestHelper = proxyRequestHelper;
      }
  
      public int filterOrder() {
          return 5;
      }
  
      public String filterType() {
          return "pre";
      }
  
      public boolean shouldFilter() {
          RequestContext ctx = RequestContext.getCurrentContext();
          return !ctx.containsKey("forward.to") && !ctx.containsKey("serviceId");
      }
  
      public Object run() {
          RequestContext ctx = RequestContext.getCurrentContext();
          String requestURI = this.urlPathHelper.getPathWithinApplication(ctx.getRequest());
          Route route = this.routeLocator.getMatchingRoute(requestURI);
          String location;
          String xforwardedfor;
          String remoteAddr;
          if (route != null) {
              location = route.getLocation();
              if (location != null) {
                  ctx.put("requestURI", route.getPath());
                  ctx.put("proxy", route.getId());
                  if (!route.isCustomSensitiveHeaders()) {
                      this.proxyRequestHelper.addIgnoredHeaders((String[])this.properties.getSensitiveHeaders().toArray(new String[0]));
                  } else {
                      this.proxyRequestHelper.addIgnoredHeaders((String[])route.getSensitiveHeaders().toArray(new String[0]));
                  }
  
                  if (route.getRetryable() != null) {
                      ctx.put("retryable", route.getRetryable());
                  }
  
                  if (!location.startsWith("http:") && !location.startsWith("https:")) {
                      if (location.startsWith("forward:")) {
                          ctx.set("forward.to", StringUtils.cleanPath(location.substring("forward:".length()) + route.getPath()));
                          ctx.setRouteHost((URL)null);
                          return null;
                      }
  
                      ctx.set("serviceId", location);
                      ctx.setRouteHost((URL)null);
                      ctx.addOriginResponseHeader("X-Zuul-ServiceId", location);
                  } else {
                      ctx.setRouteHost(this.getUrl(location));
                      ctx.addOriginResponseHeader("X-Zuul-Service", location);
                  }
  
                  if (this.properties.isAddProxyHeaders()) {
                      this.addProxyHeaders(ctx, route);
                      xforwardedfor = ctx.getRequest().getHeader("X-Forwarded-For");
                      remoteAddr = ctx.getRequest().getRemoteAddr();
                      if (xforwardedfor == null) {
                          xforwardedfor = remoteAddr;
                      } else if (!xforwardedfor.contains(remoteAddr)) {
                          xforwardedfor = xforwardedfor + ", " + remoteAddr;
                      }
  
                      ctx.addZuulRequestHeader("X-Forwarded-For", xforwardedfor);
                  }
  
                  if (this.properties.isAddHostHeader()) {
                      ctx.addZuulRequestHeader("Host", this.toHostHeader(ctx.getRequest()));
                  }
              }
          } else {
              log.warn("No route found for uri: " + requestURI);
              xforwardedfor = this.dispatcherServletPath;
              if (RequestUtils.isZuulServletRequest()) {
                  log.debug("zuulServletPath=" + this.properties.getServletPath());
                  location = requestURI.replaceFirst(this.properties.getServletPath(), "");
                  log.debug("Replaced Zuul servlet path:" + location);
              } else {
                  log.debug("dispatcherServletPath=" + this.dispatcherServletPath);
                  location = requestURI.replaceFirst(this.dispatcherServletPath, "");
                  log.debug("Replaced DispatcherServlet servlet path:" + location);
              }
  
              if (!location.startsWith("/")) {
                  location = "/" + location;
              }
  
              remoteAddr = xforwardedfor + location;
              remoteAddr = remoteAddr.replaceAll("//", "/");
              ctx.set("forward.to", remoteAddr);
          }
  
          return null;
      }
      ........
   }
  ```

- org.springframework.cloud.netflix.zuul.filters.SimpleRouteLocator

  ```java
  protected Route getRoute(ZuulRoute route, String path) {
          if (route == null) {
              return null;
          } else {
              if (log.isDebugEnabled()) {
                  log.debug("route matched=" + route);
              }
  
              String targetPath = path;
              String prefix = this.properties.getPrefix();
              if (prefix.endsWith("/")) {
                  prefix = prefix.substring(0, prefix.length() - 1);
              }
  
              if (path.startsWith(prefix + "/") && this.properties.isStripPrefix()) {
                  targetPath = path.substring(prefix.length());
              }
  			//这里获取了之前设置的StripPrefix，默认true，处理targetPath
              if (route.isStripPrefix()) {
                  int index = route.getPath().indexOf("*") - 1;
                  if (index > 0) {
                      String routePrefix = route.getPath().substring(0, index);
                      targetPath = targetPath.replaceFirst(routePrefix, "");
                      prefix = prefix + routePrefix;
                  }
              }
  
              Boolean retryable = this.properties.getRetryable();
              if (route.getRetryable() != null) {
                  retryable = route.getRetryable();
              }
  			//封装为新的Route对象供后续filter使用
              return new Route(route.getId(), targetPath, route.getLocation(), prefix, retryable, route.isCustomSensitiveHeaders() ? route.getSensitiveHeaders() : null, route.isStripPrefix());
          }
      }
  ```

至此为止，我们已经清晰地明白了StripPrefix是如何起作用的，也大概明白了zuul的运行流程。

### 后序

zuul核心在于其依赖spring servlet，再加上自定义的拦截器实现请求的转发等

Zuul 中一共有四种不同生命周期的 Filter，分别是：

- pre：在 Zuul 按照规则路由到下级服务之前执行。如果需要对请求进行预处理，比如鉴权、限流等，都应考虑在此类 Filter 实现。
- route：这类 Filter 是 Zuul 路由动作的执行者，是 Apache Http Client 或 Netflix Ribbon 构建和发送原始 HTTP 请求的地方，目前已支持 Okhttp。
- post：这类 Filter 是在源服务返回结果或者异常信息发生后执行的，如果需要对返回信息做一些处理，则在此类 Filter 进行处理。
- error：在整个生命周期内如果发生异常，则会进入 error Filter，可做全局异常处理。

在实际项目中，往往需要自实现以上类型的 Filter 来对请求链路进行处理，根据业务的需求，选取相应生命周期的 Filter 来达成目的。在 Filter 之间，通过 com.netflix.zuul.context.RequestContext 类来进行通信，内部采用 ThreadLocal 保存每个请求的一些信息，包括请求路由、错误信息、HttpServletRequest、HttpServletResponse，这使得一些操作是十分可靠的，它还扩展了 ConcurrentHashMap，目的是为了在处理过程中保存任何形式的信息。