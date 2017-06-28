# Auto Generate API Document With Swagger

## 0x1 What is Swagger
Swagger is a simple yet powerful representation of your RESTful API. With the largest ecosystem of API tooling on the planet, thousands of developers are supporting Swagger in almost every modern programming language and deployment environment. With a Swagger-enabled API, you get interactive documentation, client SDK generation and discoverability. For details please refer: [http://swagger.io/](http://swagger.io)

## 0x2 How To
SpringFox is an open source Java library integrates Spring with swagger together to generate JSON API document. We can generate json format document by integrating source code with SpringFox's annotations. 
Swagger-ui is another open source tool for rendering API documents into human readable format in html web pages. We can use the two tools to generate document with service code together.

### 0x3 Configuration

#### Maven Dependencies

```xml
<!--swagger API Doc dependencies-->
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger2</artifactId>
  <version>2.5.0</version>
</dependency>
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger-ui</artifactId>
  <version>2.5.0</version>
</dependency>
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>16.0.1</version>
</dependency>
```

#### web.xml 

Add two additional servlet-mappings to include swagger resources.

```xml
<servlet-mapping>
  <servlet-name>SpringDispatcher</servlet-name>
  <url-pattern>/*</url-pattern>
</servlet-mapping>
<servlet-mapping>
  <servlet-name>SpringDispatcher</servlet-name>
  <url-pattern>/swagger-resources</url-pattern>
</servlet-mapping>
<servlet-mapping>
  <servlet-name>SpringDispatcher</servlet-name>
  <url-pattern>/v2/api-docs</url-pattern>
</servlet-mapping>

```

#### SpringDispatcher-Servlet.xml

```xml
<!--swagger beans-->
<mvc:annotation-driven/>
<context:annotation-config/>
<bean class="com.chris.platform.mkt.SwaggerConfig"/>
<!--swagger ui-->
<mvc:resources mapping="/webjars/**" location="classpath:/META-INF/resources/webjars/"/>
<mvc:resources mapping="/swagger-ui.html" location="classpath:/META-INF/resources/swagger-ui.html"/>
<mvc:default-servlet-handler/>
```

If there is unit test in your source code, please **don't forget** to add swagger beans to test's servlet. 

#### application.properties
Add a variable to control which stack could access the documents. 

```ini
swagger.active=dev
```

#### Swagger Configuration Class
Add a swagger configuration class to customize swagger's configuration, here is an example in mkt-service.

```java
/**
 * Created by chris on 16/8/21.
 */

@Configuration
@EnableWebMvc
@EnableSwagger2
@ComponentScan(basePackageClasses = {
        CouponMgtController.class,
        PointMgtController.class,
        PointBizController.class,
        LotteryMgtController.class,
        LotteryBizController.class
})
public class SwaggerConfig {

    @Value("${swagger.active}")
    private String swaggerActive;

    @Autowired
    private TypeResolver typeResolver;

    @Bean
    public Docket mktApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("mkt-service-api")
                .apiInfo(apiInfo())
                .enable(swaggerActive.equals("dev"))
                .select()
                .paths(mktPaths())
                .build()
                .forCodeGeneration(true);
    }

    private Predicate<String> mktPaths() {
        return or(
                regex("/mkt/coupon.*"),
                regex("/mkt/point.*"),
                regex("/mkt/lottery.*")
        );
    }

    @Bean
    public Docket lotteryApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("lottery-api")
                .apiInfo(apiInfo())
                .enable(swaggerActive.equals("dev"))
                .select()
                .paths(lotteryPaths())
                .build()
                .alternateTypeRules(
                        newRule(typeResolver.resolve(DeferredResult.class,
                                typeResolver.resolve(ResponseEntity.class, WildcardType.class)),
                                typeResolver.resolve(WildcardType.class)))
                .forCodeGeneration(true);
    }

    private Predicate<String> lotteryPaths() {
        return regex("/mkt/lottery.*");
    }

    @Bean
    public Docket couponMgtApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("coupon-mgt-api")
                .apiInfo(apiInfo())
                .enable(swaggerActive.equals("dev"))
                .select()
                .paths(couponPaths())
                .build()
                .enableUrlTemplating(true);
    }

    private Predicate<String> couponPaths() {
        return regex("/mkt/coupon.*");
    }

    @Bean
    public Docket PointApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .securitySchemes(newArrayList(new BasicAuth("test")))
                .groupName("point-api")
                .apiInfo(apiInfo())
                .enable(swaggerActive.equals("dev"))
                .select()
                .paths(pointPaths())
                .build();
    }

    private Predicate<String> pointPaths() {
        return regex("/mkt/point.*");
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Marketing Service API Document")
                .description("Marketing Service API Document, includes coupon APIs, point APIs and lottery APIs.")
                .termsOfServiceUrl("http://127.0.0.1:8080/mkt")
                .license("Apache License Version 2.0")
                .licenseUrl("https://github.com/springfox/springfox/blob/master/LICENSE")
                .contact(new Contact("chris", "", "panzhongbin@gmail.com"))
                .version("1.0")
                .build();
    }
}

```
If there is unit test code in your source code please don't forget to add the annotation **@WebAppConfiguration** to your test class. Then test class would look like:

```java
@Test
@WebAppConfiguration
@ContextConfiguration(locations = "classpath*:Test-servlet.xml")
public class TestCouponRedisCache extends AbstractTestNGSpringContextTests {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private List<Coupon> mockCouponList;

    @BeforeClass
    public void setUp() {
        mockCouponList = new LinkedList<>();
        Map<String, Object> args = new HashMap<>();
        args.put("couponId", 1);
        args.put("description", "天下熙熙皆为利来,天下攘攘皆为利往");
        mockCouponList.add(buildCoupon(args));
    }
}

```
### 0x4 Example

This example is my implementation of mkt-service. For the details of annotations provided by SpringFox please refer the official document. 

```java
// API Definition
@RequestMapping(value = "/qualifier/product", method = RequestMethod.GET, produces = "application/json")
@ApiOperation(value = "查询与产品相关的参与资格条件", tags = {"抽奖资格", "MIS API"})
public ResponseEntity<QualifierProdPageListResult> getQualifierProdList(@ApiParam(value = "请求ID", example = "1471847219974")
                                                                        @RequestParam(value = "X-Request-Id", required = false) String requestId,
                                                                        @ApiParam(value = "活动Id")
                                                                        @RequestParam(value = "activityId", required = false) Integer activityId,
                                                                        @ApiParam(value = "当前页号", defaultValue = "1")
                                                                        @RequestParam(value = "page", required = false, defaultValue = "1") Integer pageIdx,
                                                                        @ApiParam(name= "分页大小", defaultValue = "50", value= "分页大小, -1表示不分页")
                                                                        @RequestParam(value = "pageSize", required = false, defaultValue = "50") Integer pageSize) {}
 
 
// Model Definition
@ApiModel(value = "中奖纪录")
public class MktActLotteryJournal {
    @JsonIgnore
    @ApiModelProperty(value = "纪录ID")
    private Integer lotteryJournalId;

    @ApiModelProperty(value = "用户名")
    private String userName;

    @ApiModelProperty(value = "账号ID")
    private Integer accountId;

    @ApiModelProperty(value = "用户真实姓名")
    private String accountName;

    @ApiModelProperty(value = "抽奖活动Id")
    private Integer activityId;

    @ApiModelProperty(value = "抽奖活动标题")
    private String activityTitle;
    
    @ApiModelProperty(value = "奖品ID")
    private Integer prizeId;

    @ApiModelProperty(value = "奖品名")
    private String prizeName;

    @ApiModelProperty(value = "奖品价值")
    private BigDecimal prizeValue;

    @ApiModelProperty(value = "奖品唯一识别码")
    private String prizeUniqueCode;

    @JsonSerialize(using = DateTimeJsonSerializer.class)
    @ApiModelProperty(value = "中奖时间", notes = "格式:yyyy-MM-ddTHH:mm:ss")
    private Date createdAt;

    @JsonIgnore
    @ApiModelProperty(hidden = true)
    private String remark;
}

```
 
## 0x5 Drawback

Even SpringFox was used, developer still needs to write document info in source code. Seems it is hard to escape from writing document.

And another issue is that swagger-ui could not display Java generic types with the real data type for Java generic's type erasing. To display the generic type correctly we have to add the code as below sample.
Suppose we want to return a response with page list contains MktActLotteryJournal objects and other additional information. We need to write helper classes to get the point.

```java
/**
 * Created by chris on 16/8/22.
 */
@ApiModel(value = "分页结果列表")
public class PageListResult<T> {

    @ApiModelProperty(value = "数据列表", required = true)
    protected List<T> data;

    @ApiModelProperty(value = "分页信息", required = true)
    private Map<String, Object> paging;

    @ApiModelProperty(value = "错误码", required = true)
    private Integer error;

    @ApiModelProperty(value = "错误描述信息", notes = "在error不为0时有效")
    private String message;

    @ApiModelProperty(value = "请求ID回显示")
    private String requestId;

    public List<T> getData() {
        return data;
    }

    public void setData(List<T> data) {
        this.data = data;
    }

    public Map<String, Object> getPaging() {
        return paging;
    }

    public void setPaging(Map<String, Object> paging) {
        this.paging = paging;
    }

    public Integer getError() {
        return error;
    }

    public void setError(Integer error) {
        this.error = error;
    }

    public String getRequestId() {
        return requestId;
    }

    public void setRequestId(String requestId) {
        this.requestId = requestId;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}


// In controller define a concrete class to extends the PageListResult
private static class JournalPageListResult extends PageListResult<MktActLotteryJournal> {
    @Override
    public void setData(List<MktActLotteryJournal> data) {
        super.setData(data);
    }
}

```

After service running the API documents were generated and could access from _http://<service-address>:8080/swagger-ui.html._ For instance the mkt service APIs could be accessed via: http://192.168.1.100:8080/swagger-ui.html#



