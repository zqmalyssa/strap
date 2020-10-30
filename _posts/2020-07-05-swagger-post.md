---
layout: post
title: Swagger集成
tags: [code, java]
author-id: zqmalyssa
---

项目需要对接外部系统，API暴露是个问题，整合一下swagger，让别人也能方便查接口

#### SpringBoot集成Swagger2

使用很简单，但是有坑

```java
<properties>
  <swagger.version>2.7.0</swagger.version>
</properties>


<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger2</artifactId>
  <version>${swagger.version}</version>
</dependency>
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger-ui</artifactId>
  <version>${swagger.version}</version>
</dependency>

```

坑在版本，2.9.2最新的swagger2版本会引用guava（使用最多），版本是`20.0`，但是项目中已经引用了`19.0`的guava了，Maven是如何选择的呢？Maven是按照就近原则选择的，层级越是浅的依赖越会被选择，所以启动项目的时候会报错，因为，2.9.2版本的swagger会找不到依赖的方法，springboot项目启动不了

解决办法是降级，用2.7.0版本，2.7.0的springfox-swagger2会引用guava，版本是`18.0`，理想肯定是向下兼容的

还有可以用插件在pom中进行排除，用的`dependency analyzer`没有检测到，`Maven Helper`应该是这个工具，比较奇怪

看下配置

```java
import io.swagger.annotations.Api;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ClassPathResource;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.handler.SimpleUrlHandlerMapping;
import org.springframework.web.servlet.resource.PathResourceResolver;
import org.springframework.web.servlet.resource.ResourceHttpRequestHandler;
import org.springframework.web.util.UrlPathHelper;
import springfox.documentation.annotations.ApiIgnore;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.DocumentationCache;
import springfox.documentation.spring.web.json.Json;
import springfox.documentation.spring.web.json.JsonSerializer;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger.web.ApiResourceController;
import springfox.documentation.swagger.web.SecurityConfiguration;
import springfox.documentation.swagger.web.SwaggerResource;
import springfox.documentation.swagger.web.UiConfiguration;
import springfox.documentation.swagger2.annotations.EnableSwagger2;
import springfox.documentation.swagger2.mappers.ServiceModelToSwagger2Mapper;
import springfox.documentation.swagger2.web.Swagger2Controller;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Configuration
@EnableSwagger2
public class SwaggerConfig {

  @Value("${swagger.enable:false}")
  private boolean enableSwagger;

  private static final String DEFAULT_PATH = "/api";

  // 下面是swagger的配置

  @Bean
  public Docket api() {
    return new Docket(DocumentationType.SWAGGER_2).enable(enableSwagger).apiInfo(apiInfo())
        .useDefaultResponseMessages(false).select().apis(RequestHandlerSelectors.withClassAnnotation(Api.class))
        .build();
  }

  private ApiInfo apiInfo() {
    return new ApiInfoBuilder().title("XXXX")
        .contact(new Contact("xxxx", "http://xxxx.com/", "xxxx@xxxx.com"))
        .description("swagger generate").version("x.x.x").build();

  }

  // 下面是配置修改swagger的默认访问路径

  @Bean
  public SimpleUrlHandlerMapping swaggerUrlHandlerMapping(ServletContext servletContext,
      @Value("${swagger.mapping.order:10}") int order) throws Exception {
    SimpleUrlHandlerMapping urlHandlerMapping = new SimpleUrlHandlerMapping();
    Map<String, ResourceHttpRequestHandler> urlMap = new HashMap<>();
    {
      PathResourceResolver pathResourceResolver = new PathResourceResolver();
      pathResourceResolver.setAllowedLocations(new ClassPathResource("META-INF/resources/webjars/"));
      pathResourceResolver.setUrlPathHelper(new UrlPathHelper());

      ResourceHttpRequestHandler resourceHttpRequestHandler = new ResourceHttpRequestHandler();
      resourceHttpRequestHandler.setLocations(Arrays.asList(new ClassPathResource("META-INF/resources/webjars/")));
      resourceHttpRequestHandler.setResourceResolvers(Arrays.asList(pathResourceResolver));
      resourceHttpRequestHandler.setServletContext(servletContext);
      resourceHttpRequestHandler.afterPropertiesSet();
      // 设置新的路径
      urlMap.put(DEFAULT_PATH + "/webjars/**", resourceHttpRequestHandler);
    }
    {
      PathResourceResolver pathResourceResolver = new PathResourceResolver();
      pathResourceResolver.setAllowedLocations(new ClassPathResource("META-INF/resources/"));
      pathResourceResolver.setUrlPathHelper(new UrlPathHelper());

      ResourceHttpRequestHandler resourceHttpRequestHandler = new ResourceHttpRequestHandler();
      resourceHttpRequestHandler.setLocations(Arrays.asList(new ClassPathResource("META-INF/resources/")));
      resourceHttpRequestHandler.setResourceResolvers(Arrays.asList(pathResourceResolver));
      resourceHttpRequestHandler.setServletContext(servletContext);
      resourceHttpRequestHandler.afterPropertiesSet();
      // 设置新的路径
      urlMap.put(DEFAULT_PATH + "/**", resourceHttpRequestHandler);
    }
    urlHandlerMapping.setUrlMap(urlMap);
    // 调整DispatcherServlet关于SimpleUrlHandlerMapping的排序
    urlHandlerMapping.setOrder(order);
    return urlHandlerMapping;
  }

  @Controller
  @ApiIgnore
  @RequestMapping(DEFAULT_PATH)
  public static class SwaggerResourceController implements InitializingBean {

    @Autowired
    private ApiResourceController apiResourceController;

    @Autowired
    private Environment environment;

    @Autowired
    private DocumentationCache documentationCache;

    @Autowired
    private ServiceModelToSwagger2Mapper mapper;

    @Autowired
    private JsonSerializer jsonSerializer;

    private Swagger2Controller swagger2Controller;

    @Override
    public void afterPropertiesSet() {
      swagger2Controller = new Swagger2Controller(environment, documentationCache, mapper, jsonSerializer);
    }

    @RequestMapping("/swagger-resources/configuration/security")
    @ResponseBody
    public ResponseEntity<SecurityConfiguration> securityConfiguration() {
      return apiResourceController.securityConfiguration();
    }

    @RequestMapping("/swagger-resources/configuration/ui")
    @ResponseBody
    public ResponseEntity<UiConfiguration> uiConfiguration() {
      return apiResourceController.uiConfiguration();
    }

    @RequestMapping("/swagger-resources")
    @ResponseBody
    public ResponseEntity<List<SwaggerResource>> swaggerResources() {
      return apiResourceController.swaggerResources();
    }

    @RequestMapping(value = "/v2/api-docs", method = RequestMethod.GET,
        produces = {"application/json", "application/hal+json"})
    @ResponseBody
    public ResponseEntity<Json> getDocumentation(@RequestParam(value = "group", required = false) String swaggerGroup,
        HttpServletRequest servletRequest) {
      return swagger2Controller.getDocumentation(swaggerGroup, servletRequest);
    }
  }
}
```

#### 遇到的问题

1、启动后，访问不鸟，加载不出来数据，显示是相关的接口信息一直加载中

百度google一大圈，其实不是跟什么fastjson相关，项目里都没有用，而是用到了`Gson`，只要有序列化，反序列化就会有问题

用浏览器访问呢http://localhost:8080/aj/v2/api-docs

```html
{
"value": "{\"swagger\":\"2.0\",\"info\":{\"description\":\"this is restful api document\",\"version\":\"1.0.0\",\"title\":\"MyApp API文档\",\"contact\":{\"name\":\"xxx\",\"url\":\"http://xxxx.com\",\"email\":\"xxx@xxx.com\"}},\"host\":\"localhost:8080\",\"basePath\":\"/\",\"tags\":[{\"name\":\"login-controller\",\"description\":\"Login Controller\"},{\"name\":\"user-controller\",\"description\":\"User Controller\"}],\"paths\"}

```
看到json格式才知道具体的是跟json转换器有关，swagger-ui里正确的json格式是一下格式的

```html
{
    "swagger": "2.0",
    "info": {
        "description": "this is restful api document",
        "version": "1.0.0",
        "title": "MyApp API文档",
        "contact": {
            "name": "userName",
            "url": "http://xxxuri.com",
            "email": "mail@mailbox.com"
        }
    },
    "host": "localhost:8080",
    "basePath": "/",
    "tags": [{
        "name": "login-controller",
        "description": "Login Controller"
    }, {
        "name": "user-controller",
        "description": "User Controller"
    }],
    "paths": {
        "/login/verify/{userId}": {
            "get": {
                "tags": [
                    "login-controller"
                ],
                "summary": "askGroupsAndRoles",
                "operationId": "askGroupsAndRolesUsingGET",
                "consumes": [
                    "application/json"
                ],
                "produces": [
                    "*/*"
                ],
                "parameters": [{
                    "name": "userId",
                    "in": "path",
                    "description": "userId",
                    "required": true,
                    "type": "string" }],
                "responses": {
                    "200": { "description": "OK" },
                    "401": { "description": "Unauthorized" },
                    "403": { "description": "Forbidden" },
                    "404": { "description": "Not Found" } }
            }
        }
}
```

因Java后台项目已经完成，需要修改一下gson转换

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

  @Autowired
  EmployeeVOAdapter employeeVOAdapter;
  @Autowired
  ExternalEmpVOAdapter externalEmpVOAdapter;

  @Override
  public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {

    List<MediaType> supportedMediaTypes = new ArrayList<>();
    supportedMediaTypes.add(MediaType.APPLICATION_JSON);
    supportedMediaTypes.add(MediaType.TEXT_PLAIN);

    GsonHttpMessageConverter converter = new GsonHttpMessageConverter();
    converter.setGson(new GsonBuilder().registerTypeAdapter(LocalDateTime.class, new LocalDateTimeTypeConverter())
        .registerTypeAdapter(EmployeeVO.class, employeeVOAdapter)
        .registerTypeAdapter(ExternalEmpVO.class, externalEmpVOAdapter)
        .registerTypeAdapter(Json.class, new JsonSerializer<Json>() {
          @Override
          public JsonElement serialize(Json json, Type typeOfSrc, JsonSerializationContext context) {
            if (json != null)
              return new JsonParser().parse(json.value());
            else
              return new JsonPrimitive("");
          }
        }).create());
    converter.setSupportedMediaTypes(supportedMediaTypes);
    converters.add(0, converter);
  }

}
```
其中，这个Json是springfox.documentation.spring.web.json.Json这个Json就星了，如果有新增的需要转换的类，也需要加一下

```java
registerTypeAdapter(Json.class, new JsonSerializer<Json>() {
  @Override
  public JsonElement serialize(Json json, Type typeOfSrc, JsonSerializationContext context) {
    if (json != null)
      return new JsonParser().parse(json.value());
    else
      return new JsonPrimitive("");
  }
}).create());
```

2、如果定义的是泛型，不返回

这边就需要在Controller返回的时候一定加上具体类型

```java
@PostMapping(value = "/api/query")
@ApiOperation(value = "query execution", notes = "query execution state")
public ListResult<EmployeeVO> query(
    @RequestBody @ApiParam(name = "param", value = "param", required = true) QueryVO param) {
  return ListResult.build(() -> externalControllerService.query(param));
}
```

#### 常用注解

@Api
用在请求的类上，表示对类的说明

@ApiOperation
用在请求类的方法上，说明方法的用途和作用

@ApiParam
可用在方法，参数和字段上，一般用在请求体参数上，描述请求体信息
就是注释对象

@ApiImplicitParams
用在请求的方法上，表示一组参数说明，里面是@ApiImplicitParam列表

@ApiImplicitParam
用在 @ApiImplicitParams 注解中，一个请求参数的说明
就是注释简单属性

@ApiResponses
用在请求的方法上，表示一组响应

@ApiResponse
用在 @ApiResponses 中，一般用于表达一个错误的响应信息

@ApiModel
用在实体类（模型）上，表示相关实体的描述。

@ApiModelProperty
用在实体类属性上，表示属性的相关描述。
主要用到example

#### 细节优化点

1、做成可以配置的，一个变量，生产环境用比较危险

2、选想要暴露的API，有各种方式，这边是匹配注解

```java
useDefaultResponseMessages(false).select().apis(RequestHandlerSelectors.withClassAnnotation(Api.class))
```

3、不用默认的UI访问路径，在swagger-ui.html前加 XXX/swagger-ui.html，为了前后端分离，后端有uri前缀，不然无法访问

解决方法，这种方式是把swagger的相关代码导入到项目, 修改index中的url, 相对于方式二繁琐了, 配置里面就是较为简单的设置，看注释
