---
title: 关于Locale和ResourceBundle踩的坑
date: 2020-03-29 10:52:05
tags: 
- Locale
- ResourceBundle
categories: Locale
---

## 问题

为新项目写了一个错误码支持国际化的缓存，但是在测试的时候遇到了一个难解的问题，测试环境不管header里传zh还是en都是英文提示，但是在本地调试却都是中文提示

## 解析

先缓存处理的代码

```java
public class ErrorCodeMessageResource {
    /** 将国际化信息存放在一个map中 */
    private static final Map<String, ResourceBundle> MESSAGES = new HashMap<String, ResourceBundle>();
    public static ResourceBundle getResourceBundle() {
        //获取语言，这个语言是从header中的Accept-Language中获取的，
        //会根据Accept-Language的值生成符合规则的locale，如zh、pt、en等

        Locale locale = LocaleContextHolder.getLocale();
        log.info("获取错误码，locale为：",locale.getLanguage());
        ResourceBundle message = MESSAGES.get(locale.getLanguage());
        if (message == null) {
            synchronized (MESSAGES) {
                //在这里读取配置信息
                message = MESSAGES.get(locale.getLanguage());
                if (message == null) {
                    log.info("开始初始化错误码及提示，locale:{}-{}",locale.getLanguage(),locale);
                    //注1
                    message = ResourceBundle.getBundle("i18n/message_errorCode", locale);
                    MESSAGES.put(locale.getLanguage(), message);
                }
            }
        }
        return message;
    }

    /** 获取国际化信息 */
    public static String getMessage(String key, Object... params) {
        ResourceBundle message = getResourceBundle();
        //此处获取并返回message
        if (params != null) {
            return String.format(message.getString(key), params);
        }
        return message.getString(key);
    }
    public static String getMessage(String key ) {
        ResourceBundle message = getResourceBundle();
        //此处获取并返回message
        return message.getString(key);
    }

}
```

resource内配置如下：

![i18n](/intro/0064.png)

后来经过大量debug和分析，发现了以下几个问题：

1、MESSAGES里存放的key是locale.getLanguage()，而ResourceBundle.getBundle是根据locale对象来获取的，getBundle方法是根据language+region来获取的

2、前端在header里传了错误的或者不全的locale，getBundle会获取jvm内的默认locale，然后再匹配对应的国际化文件，如下图：

![getBundle](/intro/0063.png)

然后就出现了上述问题：前端初次请求传了错误的header，header内只有language，如下

```java
Accept-Language=en
Accept-Language=zh_CN
等
```

，然后getBundle把所有语言返回了默认的（本地为zh_CN,测试为en_US）配置，然后再次请求，就算传了正确的header，如下：

```java
Accept-Language=en-US
Accept-Language=zh-CN
等
```

也不会返回正确的配置，因为缓存内的key获取的语言已经被错误的配置（默认的配置）占用了，所以才会出现如上问题。

PS：上面的language为什么是正确的配置呢？原因如下：

```java
//spring的RequestContextFilter会帮我们设置此次请求的LocaleContext
//org.springframework.web.filter.RequestContextFilter#doFilterInternal
	@Override
	protected void doFilterInternal(
			HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		ServletRequestAttributes attributes = new ServletRequestAttributes(request, response);
		initContextHolders(request, attributes);

		try {
			filterChain.doFilter(request, response);
		}
		finally {
			resetContextHolders();
			if (logger.isDebugEnabled()) {
				logger.debug("Cleared thread-bound request context: " + request);
			}
			attributes.requestCompleted();
		}
	}
//--->org.springframework.web.filter.RequestContextFilter#initContextHolders

	
//解析代码在此：org.eclipse.jetty.server.Request#getLocale
@Override
    public Locale getLocale()
    {
        MetaData.Request metadata = _metaData;
        if (metadata==null)
            return Locale.getDefault();

        List<String> acceptable = metadata.getFields().getQualityCSV(HttpHeader.ACCEPT_LANGUAGE);

        // handle no locale
        if (acceptable.isEmpty())
            return Locale.getDefault();

        String language = acceptable.get(0);
        language = HttpFields.stripParameters(language);
        String country = "";
        int dash = language.indexOf('-');
        if (dash > -1)
        {
            country = language.substring(dash + 1).trim();
            language = language.substring(0,dash).trim();
        }
        return new Locale(language,country);        
    }
```

![Accept-Language解析1](/intro/0065.png)

![Accept-Language解析2](/intro/0066.png)

## 解决方案

如上分析，解决方法为：缓存MESSAGES的key需要设置为language+region或者直接locale对象方可解决。如果前端传了错误的header，还是会获取默认的国际化配置（按照jvm配置来）