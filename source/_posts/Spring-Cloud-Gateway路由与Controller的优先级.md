---
title: Spring-Cloud-Gateway | 直接请求 @RequesetMapping 的 Controller 时  GlobalFilter 不生效
date: 2019-11-15 23:43:31
tags:
- Spring Cloud
- Spring Cloud Gateway
- Reactor
- WebFlux
categories:
- Spring Cloud
- Spring Cloud Gateway
---

# 请求网关层项目中的 Controller 且 ”请求路径“ 与 @RequestMapping 一致时，GlobalFilter 不生效

> 2019.11.19 补充：
> 基于 `org.springframework.web.server.WebFilter` 实现的过滤器能够覆盖包括 Spring-Cloud-Gateway 的所有请求
> 因此需要在网关层编写逻辑，使用 WebFilter 实现就可以了。

## 1 前言

之前的项目网关层都是使用Feign并复制上游的接口定义粘贴到网关层项目中，
然后编写Controller调用下游。即大部分接口都要在网关层复制一份，部分接口在网关做一些简单的逻辑。
这种方式非常繁琐，一旦上游修改接口或者DTO，网关层都要同步修改。当一份接口复制到多个网关层，修改起来存在一定工作量，且容易出错。也不知道是谁想出来这么做的🌚🌚

最近使用Spring Cloud Gateway作为应用网关。在网关层，只需进行一定配置，发送到网关的请求会根据一定的规则路由到对应的服务，省去了以前复制接口的工作，减少了网关层的工作量。

虽然大部分接口都是可以直接转发到对应服务，还有少部分接口需要在网关层做一些逻辑，例如生成打印图片返回给App端。
这部分逻辑不适合直接做在后端服务中，前端做起来也有难度，因此，这部分逻辑可以放到网关层实现。

**目前遇到一个问题，即本文探讨的问题：** 网关层的操作需要鉴权，因此存在一个用于鉴权的GlobalFilter，
前端的请求只有带有有效的登录信息才会被转发到后端对应的服务。
现在在网关层中编写了一个逻辑简单但必要的Controller，这个Controller中的接口同样需要鉴权。
但经过测试，对应Controller的RequestMapping的请求不会经过GlobalFilter，直接发到了Controller中，即未被鉴权。


## 2 实践

> 此处直接在开源版本的[xxl-sso](https://github.com/teslacn/xxl-sso)上创建一个基于Spring-Cloud-Gateway的Client
> 并通过GlobalFilter实现鉴权过滤器

### 2.1 编写Controller

```java
import com.xxl.sso.core.conf.Conf;
import com.xxl.sso.core.entity.ReturnT;
import com.xxl.sso.core.user.XxlSsoUser;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/demo")
public class IndexController {

    @GetMapping("/user")
    public Mono<ReturnT<XxlSsoUser>> index(ServerWebExchange exchange) {
        XxlSsoUser xxlUser = exchange.getAttribute(Conf.SSO_USER);
        return Mono.just(new ReturnT<>(xxlUser));
    }
}
```

### 2.2 编写GlobalFilter

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Component
public class XxlSsoTokenGatewayFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        // 省略过滤逻辑
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}

```

### 2.3 路由配置

```properties
spring.cloud.gateway.routes[0].id=default
spring.cloud.gateway.routes[0].uri=forward:/
spring.cloud.gateway.routes[0].predicates[0]=Path=/demo/user
spring.cloud.gateway.routes[0].filters[0]=SetPath=/demo/user
spring.cloud.gateway.routes[1].id=internal
spring.cloud.gateway.routes[1].uri=forward:/
spring.cloud.gateway.routes[1].predicates[0]=Path=/internal/user
spring.cloud.gateway.routes[1].filters[0]=SetPath=/demo/user
spring.cloud.gateway.routes[2].id=github
spring.cloud.gateway.routes[2].uri=https://github.com
spring.cloud.gateway.routes[2].predicates[0]=Path=/TeslaCN/**
```
配置了3条路由：
1. 内部转发 `/demo/user` 到 `/demo/user`
2. 内部转发 `/internal/user` 到 `/demo/user`
3. 转发 `/TeslaCN/**` 到 [GitHub](https://github.com)

### 2.4 启动 DEBUG 打断点，发送请求

将断点打在了filter逻辑的第一行

![breakpoint](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/breakpoint.png)

#### 2.4.1 直接请求 `/TeslaCN/xxl-sso`

不经过登录，直接请求

![request_github_not_login](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_github_not_login.png)

返回结果：未登录

![request_github_not_login_result](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_github_not_login_result.png)

#### 2.4.2 请求SSO服务器登录后 再请求 `/TeslaCN/xxl-sso`

请求 SSO 登录接口  

![request_xxl_sso](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_xxl_sso.png)

返回结果：登录成功，data 即 Token

![request_xxl_sso_result](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_xxl_sso_result.png)

将 Token 放入 Headers 中，发送请求到网关

![request_github_logined](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_github_logined.png)

此时断点生效

![request_github_logined_breakpoint](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_github_logined_breakpoint.png)

继续运行，查看响应结果，可以看到转发到GitHub的请求成功了

![request_github_logined_result](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_github_logined_result.png)

#### 2.4.3 请求 Gateway 中的 Controller

##### 预期结果  

未登录时，拦截请求并返回如[2.4.1](#241-%e7%9b%b4%e6%8e%a5%e8%af%b7%e6%b1%82-teslacnxxl-sso)中的结果；
附带有效的xxl_sso_sessionid发送请求时，接口按照逻辑返回当前登录的用户信息

##### 实际结果  

准备请求

![request_demo_not_login](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_demo_not_login.png)

返回结果：并没有返回 "sso not login" 的结果，而是成功请求了接口，与预期结果不一致。

![request_demo_not_login-result](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/request_demo_not_login-result.png)

即GlobalFilter并没有Global🌚

##### 猜想

也许是因为网关路由的优先级低于RequestMapping，所以“请求路径”和`@RequestMapping`一致时优先匹配RequestMapping？

## 3 分析与验证

Spring Web MVC 有 `DispatcherServlet`，Spring Cloud Gateway (WebFlux) 同样也有 `DispatcherHandler`

先从 `DispatcherHandler` 的源码下手

### `DispatcherHandler` 源码节选

Version: org.springframework:spring-webflux:5.0.4.RELEASE

```java
/**
 * Central dispatcher for HTTP request handlers/controllers. Dispatches to
 * registered handlers for processing a request, providing convenient mapping
 * facilities.
 *
 * <p>{@code DispatcherHandler} discovers the delegate components it needs from
 * Spring configuration. It detects the following in the application context:
 * <ul>
 * <li>{@link HandlerMapping} -- map requests to handler objects
 * <li>{@link HandlerAdapter} -- for using any handler interface
 * <li>{@link HandlerResultHandler} -- process handler return values
 * </ul>
 *
 * <p>{@code DispatcherHandler} s also designed to be a Spring bean itself and
 * implements {@link ApplicationContextAware} for access to the context it runs
 * in. If {@code DispatcherHandler} is declared with the bean name "webHandler"
 * it is discovered by {@link WebHttpHandlerBuilder#applicationContext} which
 * creates a processing chain together with {@code WebFilter},
 * {@code WebExceptionHandler} and others.
 *
 * <p>A {@code DispatcherHandler} bean declaration is included in
 * {@link org.springframework.web.reactive.config.EnableWebFlux @EnableWebFlux}
 * configuration.
 *
 * @author Rossen Stoyanchev
 * @author Sebastien Deleuze
 * @author Juergen Hoeller
 * @since 5.0
 * @see WebHttpHandlerBuilder#applicationContext(ApplicationContext)
 */
public class DispatcherHandler implements WebHandler, ApplicationContextAware {

	private static final Log logger = LogFactory.getLog(DispatcherHandler.class);

	@Nullable
	private List<HandlerMapping> handlerMappings;

	@Nullable
	private List<HandlerAdapter> handlerAdapters;

	@Nullable
	private List<HandlerResultHandler> resultHandlers;

	/**
	 * Create a new {@code DispatcherHandler} for the given {@link ApplicationContext}.
	 * @param applicationContext the application context to find the handler beans in
	 */
	public DispatcherHandler(ApplicationContext applicationContext) {
		initStrategies(applicationContext);
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		initStrategies(applicationContext);
	}

	protected void initStrategies(ApplicationContext context) {
		Map<String, HandlerMapping> mappingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerMapping.class, true, false);

		ArrayList<HandlerMapping> mappings = new ArrayList<>(mappingBeans.values());
		AnnotationAwareOrderComparator.sort(mappings);
		this.handlerMappings = Collections.unmodifiableList(mappings);

		Map<String, HandlerAdapter> adapterBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerAdapter.class, true, false);

		this.handlerAdapters = new ArrayList<>(adapterBeans.values());
		AnnotationAwareOrderComparator.sort(this.handlerAdapters);

		Map<String, HandlerResultHandler> beans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerResultHandler.class, true, false);

		this.resultHandlers = new ArrayList<>(beans.values());
		AnnotationAwareOrderComparator.sort(this.resultHandlers);
	}
}
```

#### `DispatcherHandler` 初始化

关注 Field 和初始化方法，在 `initStrategies` 方法打了个断点。

可以看到，此时3个 Field 都是 null

![dispatcherhandler-init-1](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/dispatcherhandler-init-1.png)


执行了第1行代码后，handlerMappings 的 Bean实例 都拿到了。
可以看到，数组中有 4个Mapping，其中包括了：
* `RequestMappingHandlerMapping`
* `RoutePredicateHandlerMapping`

![dispatcherhandler-init-2](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/dispatcherhandler-init-2.png)

进行排序操作后，它们的顺序如图

![ordered-handler-mappings](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/ordered-handler-mappings.png)

点开 `handlerMappings` 的属性

![handler-mappings-fields](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/handler-mappings-fields.png)

其实看到这里，答案也有了。

**`RequestMappingHandlerMapping` order < `RoutePredicateHandlerMapping` order**

HandlerMapping的顺序问题，我浏览了
[Reference Doc. 2.2.0 RC2](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.0.RC2/reference/html/)
貌似没有找到相关顺序的配置项

## 4 思考

官方文档没有提到 `HandlerMapping` 的顺序的配置项，是因为”遗漏了“还是”设计本如此“？

`RoutePredicateHandlerMapping` 源码节选
```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

	private final FilteringWebHandler webHandler;
	private final RouteLocator routeLocator;

	public RoutePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator) {
		this.webHandler = webHandler;
		this.routeLocator = routeLocator;

        // 构造方法直接将 order 写死为 1
		setOrder(1);
	}
}
```

`WebFluxConfigurationSupport` 源码节选
```java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();

    // order 写死为 0
    mapping.setOrder(0);

    mapping.setContentTypeResolver(webFluxContentTypeResolver());
    mapping.setCorsConfigurations(getCorsConfigurations());

    PathMatchConfigurer configurer = getPathMatchConfigurer();
    Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
    Boolean useCaseSensitiveMatch = configurer.isUseCaseSensitiveMatch();
    if (useTrailingSlashMatch != null) {
        mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
    }
    if (useCaseSensitiveMatch != null) {
        mapping.setUseCaseSensitiveMatch(useCaseSensitiveMatch);
    }
    return mapping;
}
```

在文档第15节
[15. Building a Simple Gateway Using Spring MVC or Webflux](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.0.RC2/reference/html/#building-a-simple-gateway-using-spring-mvc-or-webflux)
提到通过 `WebFlux` 构建简单的网关

![doc-15](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2019/09/Spring-Cloud-Gateway_GlobalFilter_doesnt_work/doc-15.png)

假设我调整了 `RequestMappingHandlerMapping` 和 `RoutePredicateHandlerMapping` 
的顺序，使后者顺序更前，我在本项目所写的 GlobalFilter 就能够直接作用在 Controller 上，
实际结果可能就和
[2.4.3](#243-%e8%af%b7%e6%b1%82-gateway-%e4%b8%ad%e7%9a%84-controller)
所提到的预期结果一致了

但同时，上述文档提到的通过 `WebFlux` / `MVC` 构建网关的方式，所有请求都会经过Filter，
~~此时会出现问题~~
貌似也不会有问题，让所有请求都经过了Filter。

**应该结合具体场景考虑**

## 5 当前问题解决方案

方案：
1. 抽象，将鉴权逻辑与 MVC / WebFlux / Gateway 代码解耦
2. 使用 webFlux Filter 再实现一遍鉴权逻辑
3. 基于 `org.springframework.web.server.WebFilter` 开发鉴权，可以覆盖所有请求

目前本人倾向于 第2种 解决方案。

xxl-sso 项目本身提供了基于 Servlet 的 Filter，
本人自己实现了基于 Spring-Cloud-Gateway 的过滤逻辑，
过滤逻辑 **比较简单** 且 **不会频繁修改** ，用 WebFlux 再实现一遍可能问题不大🌚

### 附：WebFilter示例
```java
package com.xxl.sso.sample.filter;

import com.xxl.sso.core.conf.Conf;
import com.xxl.sso.core.entity.ReturnT;
import com.xxl.sso.core.exception.XxlSsoException;
import com.xxl.sso.core.path.impl.AntPathMatcher;
import com.xxl.sso.core.user.XxlSsoUser;
import com.xxl.sso.sample.helper.XxlGatewaySsoTokenLoginHelper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.text.MessageFormat;

import static com.xxl.sso.core.conf.Conf.SSO_USER;

/**
 * 基于 GlobalFilter 的过滤器只能过滤 Spring-Cloud-Gateway 的路由配置
 * 基于 WebFilter 能够覆盖 RequestMapping 和 Gateway
 *
 * @author Wu Weijie
 * @see XxlSsoTokenWebFilter
 */
@Configuration
@Order(-1)
public class XxlSsoTokenWebFilter implements WebFilter {

    public static final String NOT_LOGIN_MESSAGE = "{\"code\":" + Conf.SSO_LOGIN_FAIL_RESULT.getCode() + ", \"msg\":\"" + Conf.SSO_LOGIN_FAIL_RESULT.getMsg() + "\"}";
    public static final String ERROR_MESSAGE_TEMPLATE = "'{'\"code\":\"500\", \"msg\":\"{0}\"}";
    public static final String LOGOUT_SUCCESS_MESSAGE = "{\"code\":" + ReturnT.SUCCESS_CODE + ", \"msg\":\"\"}";

    private static final AntPathMatcher ANT_PATH_MATCHER = new AntPathMatcher();
    @Value("${xxl-sso.excluded.paths}")
    private String excludedPaths;
    @Value("${xxl.sso.server}")
    private String ssoServer;
    @Value("${xxl.sso.logout.path}")
    private String logoutPath;
    @Value("${server.api-prefix}")
    private String apiPrefix;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();

        /*
        获取请求路径并去掉统一前缀
        eg:
        api-prefix=/api
        requestPath=/api/user/demo
        处理后：/user/demo
        统一的前缀可以理解为 server.servlet.context-path
        但 spring cloud gateway 中暂时没有前缀配置方法，因此自定义一个路径前缀参数
         */
        String requestPath = request.getPath().value().substring(apiPrefix.length());

        if (excludedPaths != null && !excludedPaths.trim().isEmpty()) {
            // if path in excludePaths
            for (String excludePath : excludedPaths.split(",")) {
                String uriPattern = excludePath.trim();

                if (ANT_PATH_MATCHER.match(uriPattern, requestPath)) {
                    // pass
                    return chain.filter(exchange);
                }
            }
        }


        // logout filter
        if (logoutPath != null
                && logoutPath.trim().length() > 0
                && logoutPath.equals(requestPath)) {

            // logout
            XxlGatewaySsoTokenLoginHelper.logout(request);

            // response
            response.setStatusCode(HttpStatus.OK);
            response.getHeaders().add("Content-Type", "application/json;charset=utf-8");
            return response.writeWith(
                    Flux.just(response.bufferFactory().wrap(LOGOUT_SUCCESS_MESSAGE.getBytes(StandardCharsets.UTF_8))));
        }

        // login filter
        try {
            XxlSsoUser xxlSsoUser = XxlGatewaySsoTokenLoginHelper.loginCheck(request);
            if (xxlSsoUser == null) {
                response.setStatusCode(HttpStatus.OK);
                response.getHeaders().add("Content-Type", "application/json;charset=utf-8");
                return response.writeWith(
                        Flux.just(response.bufferFactory().wrap(NOT_LOGIN_MESSAGE.getBytes(StandardCharsets.UTF_8))));
            }

            // 用户登录状态有效，滤过
            exchange.getAttributes().put(SSO_USER, xxlSsoUser);
            return chain.filter(exchange);

        } catch (XxlSsoException e) {
            response.setStatusCode(HttpStatus.OK);
            response.getHeaders().add("Content-Type", "application/json;charset=utf-8");
            // return error
            return response.writeWith(
                    Flux.just(response.bufferFactory().wrap(MessageFormat.format(ERROR_MESSAGE_TEMPLATE, e.getMessage()).getBytes(StandardCharsets.UTF_8))));
        }
    }
}
```
