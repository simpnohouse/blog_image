```
项目开发流程？
	需求分析：
	产品设计：axure、MockupCreator.exe 
	开发：
		技术选型、接口文档
				ER图：
				技术架构图：
		前端：   mock    vscode 、webstorm   
		后端：	  swagger、jmeter、postman        idea、eclipse(sts)
		联调
	测试：
	运维：部署运行    k8s  jenkins+docker
项目开发管理工具：
	禅道(有开源版)：




数据库设计？
异常类型？
接口文档？
常见的日志框架？
为什么要统一响应？
```



# 1、mp分页配置

## 1.1 配置mp分页查询拦截器

srb-base中引入依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
</dependency>
```

srb-base创建配置类

```java
//将数据库 mp mybatis相关的注解全部抽取到当前配置类中
@Configuration
//srb-core启动类上面的 MapperScan注解剪切到此处
@MapperScan(basePackages = {"com.atguigu.srb.core.mapper"})
public class MybatisPlusConfig {
    //将mp的分页拦截器注入到容器中
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
        MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
        paginationInnerInterceptor.setDbType(DbType.MYSQL);
        //设置分页合理化： 如果页码超过总页码返回最后一页  如果页码小于1返回第一页
        paginationInnerInterceptor.setOverflow(true);
        //将分页拦截器设置到mybatisPlusInterceptor
        mybatisPlusInterceptor.addInnerInterceptor(paginationInnerInterceptor);
        //乐观锁拦截器
        return mybatisPlusInterceptor;
    }

}
```

**srb-core**中启动类上指定扫描的包：必须扫描到自己项目中的组件 和 srb-base中新创建的配置类

```java
@ComponentScan(basePackages = {"com.atguigu.srb"})
```



## 1.2 自定义分页查询

单表分页查询不需要自定义，为了整合mp的分页 和mybatis的自定义sql的实现

### 1、将srb-core mapper包下的xml剪切到resources/mapper目录下

### 2、在application中配置mapper扫描路径

```yaml
mybatis-plus:
  # 配置mapper映射文件所在的包
	#  mapper-locations: classpath:/com/atguigu/srb/core/mapper/xml/*.xml
  # maven编译时默认会忽略java目录下的java以外的文件  不会编译到target/classes目录中
  mapper-locations: classpath:/mapper/xml/*.xml
```

### 3、自定义业务方法mapper实现分页查询

```java
//IntegralGradeService
void queryIntegralGradesByKey(IPage<IntegralGrade> page, Integer key);
```



```java
//IntegralGradeServiceImpl   
public void queryIntegralGradesByKey(IPage<IntegralGrade> page, Integer key) {
    //mapper的方法查询后 一般返回 行映射的对象 或者 对象集合、或者map    影响的行数
    List<IntegralGrade> list = baseMapper.selectIntegralGradesByKey(page,key);
    //将数据库查询到的分页的记录集合设置到分页对象中
    page.setRecords(list);

}
```

```java
//IntegralGradeMapper
//mp自定义方法  接收了page对象，mp如果配置了分页拦截器 它会自动根据page对象解析出分页条件拼接sql
List<IntegralGrade> selectIntegralGradesByKey(IPage<IntegralGrade> page, @Param("key") Integer key);

```

```xml
<!-- IntegralGradeMapper.xml -->
<select id="selectIntegralGradesByKey" resultType="com.atguigu.srb.core.pojo.entity.IntegralGrade">
    SELECT *
    FROM integral_grade
    WHERE integral_start> #{key}
</select>
```

出现异常：

```
ibatis.binding.BindingException
   mapper接口中的方法 找不到mapper映射文件中的实现
   > mapper映射文件和mapper接口不是绑定关系(mybatis必须能找到mapper映射文件的路径、mapper映射文件中namespace必须绑定mapper接口的全类名，文件的名称应该一样)
   > mapper映射文件需要实现mapper接口中的方法
   
```

解决：

```
将mapper包下的xml整个包 剪切到  resources目录下的mapper目录中

在srb-core的application文件中添加mapper-location的配置
  # maven编译时默认会忽略java目录下的java以外的文件  不会编译到target/classes目录中
  mapper-locations: classpath:/mapper/xml/*.xml
```



# 2、mp自动填充

数据库设计表时都会设计时间字段,新增和更新时都会修改创建或者更新时间

代码中操作时间的内容就冗余了

mp支持自动填充

## 2.1 在srb-base中创建填充处理器

```java
@Component
public class GlobalFillDateMetaObjectHandler implements MetaObjectHandler {
    //新增时触发：自动填充
    @Override
    public void insertFill(MetaObject metaObject) {
//        metaObject代表正在被mp执行新增的对象原数据对象
        //判断对象是否有createTime和updateTime属性  如果有自动设置属性值
        if(metaObject.hasSetter("createTime")){
            metaObject.setValue("createTime",new Date());
        }
        if(metaObject.hasSetter("updateTime")){
            metaObject.setValue("updateTime",new Date());
        }
    }
    //更新时触发：自动填充
    @Override
    public void updateFill(MetaObject metaObject) {
//        metaObject代表正在被mp执行更新的对象原数据对象
        if(metaObject.hasSetter("updateTime")){
            metaObject.setValue("updateTime",new Date());
        }
    }
}

```

## 2.2 在需要填充的javabean的属性上指定填充方式

```java
//IntegralGrade
@TableField(fill = FieldFill.INSERT)
private Date createTime;

@ApiModelProperty(value = "更新时间")
@TableField(fill = FieldFill.INSERT_UPDATE)
private Date updateTime;
//拷贝属性和上面的注解，ctrl+shift+r 全局搜索属性那一行 替换掉
```

## 2.3 以后新增更新时 会自动填充

如果使用的是对象新增 对象更新会触发自动填充，如果是对某些属性值进行操作 不会触发

```java
//会触发填充：
integralGradeService.updateById(integralGrade);
integralGradeService.save(integralGrade);
```

```java
//不能触发自动填充
integralGradeService.update(Wrappers.lambdaUpdate(IntegralGrade.class)
                            .set(IntegralGrade::getBorrowAmount , 1000)
                            .eq(IntegralGrade::getId , 1));
```



## 2.4 id精度丢失问题

```
如果后端id使用Long类型接收，长度超过16位时，后面的会四舍五入
雪花算法生成的id长度19位 索引以后会出现jackson转换long类型的id为json字符串时精度丢失
前端框架js接收到的长度超过16位的数字时也会精度丢失

解决：
	方案1：用String类型接收id
	方案2：给jackson配置转换器 id转为json之前先处理为字符串类型。
```

**srb-base**中创建配置类：全局处理jackson 长度可能超过16的类型的转换  转为字符串

```java
@Configuration
public class JacksonConfig {
    //jackson将对象转为json之前 将以下的属性转为json字符串
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer(){
        return jacksonObjectMapperBuilder -> {
            jacksonObjectMapperBuilder.serializerByType(Long.TYPE, ToStringSerializer.instance);
            jacksonObjectMapperBuilder.serializerByType(Long.class, ToStringSerializer.instance);
            jacksonObjectMapperBuilder.serializerByType(BigInteger.class, ToStringSerializer.instance);
            jacksonObjectMapperBuilder.serializerByType(BigDecimal.class, ToStringSerializer.instance);
        };
    }
}
```

配置后项目重启，查询的数据的雪花算法的id不会出现精度丢失

# 3、mp逻辑删除

默认已经支持了逻辑删除 删除的数据deleted字段改为了1

查询时 只会查询is_deleted字段为0的数据



# 4、swagger配置

## 4.1 注解

```
swagger的注解 只有在使用swagger测试时 才会起作用，不会影响项目的正常运行
Controller上使用的：
	生成接口文档
	@Api(tags="xx") 描述接口的作用的
	@ApiOpration(value="xxx")  介绍接口方法的作用
	@ApiParam(value="xxx")  介绍接口的一个形参的作用
Model(数据模型)上使用的：
	描述javabean的作用 每个属性的类型 含义....
	@ApiModel(value="xx")
	@ApiModelProperty(value="xx") 
```

## 4.2 分组

在srb-base中创建swagger配置类实现分组

### 4.2.1 先在srb-core中controller下创建api和admin包

api：存放会员访问的Controller模块   api下的Controller路径以/api开始

```
@RequestMapping("/api/integralGrade")
public class ApiIntegralGradeController
```

admin：存放管理员访问的Controller模块  admin下的Controller路径以/admin开始

```
@RequestMapping("/admin/integralGrade")
public class AdminIntegralGradeController 
```

分开控制编写 权限控制比较方便

### 4.2.2 创建swagger分组配置类

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {
    //扫描管理员controller模块的配置：
    @Bean
    public Docket adminApiConfig(){
        return new Docket(DocumentationType.SWAGGER_2)
            .groupName("adminApi")
            .apiInfo(adminApiInfo())
            .select()
            //只显示admin路径下的页面
            .paths(Predicates.and(PathSelectors.regex("/admin/.*")))
            .build();
    }
    private ApiInfo adminApiInfo(){
        return new ApiInfoBuilder()
            .title("尚融宝后台管理系统-API文档")
            .description("本文档描述了尚融宝后台管理系统接口")
            .version("1.0")
            .contact(new Contact("Atguigu", "http://atguigu.com", "atguigu@126.com"))
            .build();
    }
    @Bean
    public Docket webApiConfig(){
        return new Docket(DocumentationType.SWAGGER_2)
            .groupName("webApi")
            .apiInfo(webApiInfo())
            .select()
            //只显示api路径下的页面
            .paths(Predicates.and(PathSelectors.regex("/api/.*")))
            .build();
    }
    private ApiInfo webApiInfo(){
        return new ApiInfoBuilder()
            .title("尚融宝用户前台系统-API文档")
            .description("本文档描述了尚融宝用户前台系统接口")
            .version("1.0")
            .contact(new Contact("Atguigu", "http://atguigu.com", "atguigu@126.com"))
            .build();
    }
}
```

swagger控制台中可以切换分组查询组内的Controller





# 5、统一响应

## 5.1 统一响应类分析

```
以后web项目用户在访问时：
	响应状态码：5xx
		如果后端出现异常没有处理，会上抛给服务器，服务器抛出到页面中展示给用户
	响应状态码：200
		200状态码代表服务器没有抛出异常
		> 如果后端执行时业务正常没有任何问题 可以返回成功的数据
		> 如果后端业务逻辑任务有问题希望直接结束并告诉用户问题所在
			但是返回结果不代表操作成功，又不能将异常直接抛出
		
解决方案：
	当后端业务执行没有任何问题,可以返回一个成功的对象(成功的数据)
	当后端业务执行业务判断失败时，逻辑上认为失败时，可以返回一个失败的对象(描述失败)
	当服务器出现未捕获异常时，可以通过全局异常处理器处理异常，返回一个失败的对象(描述失败)
以上的解决方案处理后，后端返回的响应报文状态码都是200
	为了方便前端区分后端返回的是成功还是失败，可以在成功或失败的对象中通过属性描述成功还是失败
	可以将响应数据 抽取成一个类，描述响应结果：
	
class R{
  private Integer code; //响应对象状态码  0代表成功  1表示默认失败 其他代表失败  
  private String msg;//状态码的描述文本
  private Map<String,Object> data = new HashMap();//接收成功的响应数据(item,integralGrade)(items,list)
	//获取一个代表成功的对象
	public static R ok(){
		R r = new R();
	    r.setCode(0);
	    r.setMsg("success");
	    return r;
	}
	//获取一个代表失败的对象
	public static R error(){
		R r = new R();
	    r.setCode(1);
	    r.setMsg("未知错误");
	    return r;
	}
	//向map中存入成功的响应数据
	public R data(String key,Object val){
		this.data.put(key,val);
		return this;
	}
	//修改code的方法：支持链式调用
	public R code(Integer code){
		this.code = code;
		return this;
	}
	//修改msg的方法：支持链式调用
	public R msg(String msg){
		this.msg = msg;
		return this;
	}
}	
	
R.error().code(2).msg("账号错误");	
R.ok().data("item",obj);	
	
	
```

## 5.2 枚举类

```java
@Getter
public enum ResponseEnum {
    SUCCESS(0,"成功"),
    ERROR(-1, "服务器内部错误"),
    //-1xx 服务器错误
    BAD_SQL_GRAMMAR_ERROR(-101, "sql语法错误"),
    SERVLET_ERROR(-102, "servlet请求异常"), //-2xx 参数校验
    UPLOAD_ERROR(-103, "文件上传错误"),
    EXPORT_DATA_ERROR(-104, "数据导出失败"),
    PARAM_ERROR(-105, "参数校验失败"),
    //-2xx 参数校验
    BORROW_AMOUNT_NULL_ERROR(-201, "借款额度不能为空"),
    MOBILE_NULL_ERROR(-202, "手机号码不能为空"),
    MOBILE_ERROR(-203, "手机号码不正确"),
    PASSWORD_NULL_ERROR(204, "密码不能为空"),
    CODE_NULL_ERROR(205, "验证码不能为空"),
    CODE_ERROR(206, "验证码错误"),
    MOBILE_EXIST_ERROR(207, "手机号已被注册"),
    LOGIN_MOBILE_ERROR(208, "用户不存在"),
    LOGIN_PASSWORD_ERROR(209, "密码错误"),
    LOGIN_LOKED_ERROR(210, "用户被锁定"),
    LOGIN_AUTH_ERROR(-211, "未登录"),
    USER_BIND_IDCARD_EXIST_ERROR(-301, "身份证号码已绑定"),
    USER_NO_BIND_ERROR(302, "用户未绑定"),
    USER_NO_AMOUNT_ERROR(303, "用户信息未审核"),
    USER_AMOUNT_LESS_ERROR(304, "您的借款额度不足"),
    LEND_INVEST_ERROR(305, "当前状态无法投标"),
    LEND_FULL_SCALE_ERROR(306, "已满标，无法投标"),
    NOT_SUFFICIENT_FUNDS_ERROR(307, "余额不足，请充值"),
    PAY_UNIFIEDORDER_ERROR(401, "统一下单错误"),
    ALIYUN_SMS_LIMIT_CONTROL_ERROR(-502, "短信发送过于频繁"),//业务限流
    ALIYUN_SMS_ERROR(-503, "短信发送失败"),//其他失败
    WEIXIN_CALLBACK_PARAM_ERROR(-601, "回调参数不正确"),
    WEIXIN_FETCH_ACCESSTOKEN_ERROR(-602, "获取access_token失败"),
    WEIXIN_FETCH_USERINFO_ERROR(-603, "获取用户信息失败"),
    ;

    private Integer code;
    private String msg;

    private ResponseEnum(Integer code , String msg){
        this.code = code;
        this.msg = msg;
    }
}

```

## 5.3 响应类

```java
@Data
public class R {
    //响应对象状态码  0代表成功  1表示默认失败 其他代表失败
    private Integer code;
    //状态码的描述文本
    private String msg;
	//接收成功的响应数据(item,integralGrade)(items,list)
    private Map<String, Object> data = new HashMap();


    //获取一个代表成功的对象
    public static R ok() {
        R r = new R();
        r.setCode(ResponseEnum.SUCCESS.getCode());
        r.setMsg(ResponseEnum.SUCCESS.getMsg());
        return r;
    }

    //获取一个代表失败的对象
    public static R error() {
        R r = new R();
        r.setCode(ResponseEnum.ERROR.getCode());
        r.setMsg(ResponseEnum.ERROR.getMsg());
        return r;
    }
    //根据传入的枚举类对象 创建R对象
    public static R setResult(ResponseEnum responseEnum) {
        R r = new R();
        r.setCode(responseEnum.getCode());
        r.setMsg(responseEnum.getMsg());
        return r;
    }
    //向map中存入成功的响应数据
    public R data(String key, Object val) {
        this.data.put(key, val);
        return this;
    }

    //修改code的方法：支持链式调用
    public R code(Integer code) {
        this.code = code;
        return this;
    }

    //修改msg的方法：支持链式调用
    public R msg(String msg) {
        this.msg = msg;
        return this;
    }
}
```

## 5.4 修改接口使用响应类响应

```
item表示一个对象
items表示对象的List集合
page表示对象的分页数据
flag表示布尔值
```



# 6、统一日志配置

```
常见的日志框架：
	commons-loging: spring框架默认使用的
	log4j: mybatis默认使用的
	logback: springboot默认使用的
	slf4j: 日志门面  主要用来统一日志框架
	jul: 不用 java1.4后自带的
所有的日志框架功能基本都一样：
	控制日志输出级别
	持久化日志
	控制日志输出格式...
项目中使用的日志框架较多，又不能排除某些日志依赖，因为有些框架在使用它，如果每个日志框架API都学习 成本太高

	slf4j: 可以统一日志框架，以后只要使用slf4j的api就可以打印日志，而且所有框架不受影响 都可以被slf4j统一配置日志的级别 格式 持久划...
		slf4j提供了日志转换包(日志全类名、方法信息和要转换的日志包类一样)但是打印日志时slf4j可以调用logback的方法打印日志
		
```

在srb-core resources下创建     logback-spring.xml    配置文件，编写配置

​			需要修改：log.path 为自己的盘符路径

​			在项目的applictaion中配置： spring.profiles.active=dev

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration  scan="true" scanPeriod="10 seconds">
    <contextName>logback</contextName>
    <property name="log.path" value="F:/srb/core" />
    <!--控制台日志格式：彩色日志-->
    <!-- magenta:洋红 -->
    <!-- boldMagenta:粗红-->
    <!-- cyan:青色 -->
    <!-- white:白色 -->
    <!-- magenta:洋红 -->
    <property name="CONSOLE_LOG_PATTERN"
              value="%yellow(%date{yyyy-MM-dd HH:mm:ss}) |%highlight(%-5level) |%blue(%thread) |%blue(%file:%line) |%green(%logger) |%cyan(%msg%n)"/>
    <!--文件日志格式-->
    <property name="FILE_LOG_PATTERN"
              value="%date{yyyy-MM-dd HH:mm:ss} |%-5level |%thread |%file:%line |%logger |%msg%n" />
    <!--编码-->
    <property name="ENCODING"
              value="UTF-8" />
    <!--输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <!--日志级别-->
            <level>DEBUG</level>
        </filter>
        <encoder>
            <!--日志格式-->
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!--日志字符集-->
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>
    <!--输出到文件-->
    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志过滤器：此日志文件只记录INFO级别的-->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_info.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${log.path}/info/log-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
    </appender>
    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 日志过滤器：此日志文件只记录WARN级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_warn.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/warn/log-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
    </appender>
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 日志过滤器：此日志文件只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_error.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error/log-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
    </appender>
    <!-- 下面配置是否激活由application.yml中的spring.profiles.active的值决定 -->
    <!--开发环境-->
    <springProfile name="dev">
        <!--可以灵活设置此处，从而控制日志的输出-->
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="INFO_FILE" />
            <appender-ref ref="WARN_FILE" />
            <appender-ref ref="ERROR_FILE" />
        </root>
    </springProfile>
    <!--生产环境-->
    <springProfile name="pro">
        <root level="ERROR">
            <appender-ref ref="ERROR_FILE" />
        </root>
    </springProfile>
</configuration>
```







# 7、统一异常处理

## 7.1 异常处理的原因

```
用户访问后端接口时我们不希望将5xx抛出 页面不友好
前端异步请求时 后端希望统一响应，成功返回R成功对象 失败返回失败R对象

我们希望返回的响应报文状态码都是200
	前端根据R对象的code值判断后端处理成功还是失败
	成功解析data数据展示，失败获取msg给用户提示
```

异常测试

## 7.2 异常处理器

srb-base创建全局异常处理器

```java
@Slf4j
// 全局异常处理  处理后的结果转为json响应
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(value = Exception.class)
    public R exception(Exception e){ //形参类型需要和异常处理注解配置的类型一致
        //打印异常的堆栈消息(包含了代码那些行出现了什么问题)
        log.error("出现了异常: {}", ExceptionUtils.getStackTrace(e));
        return R.error();
    }
}
```

特定异常处理器

```java
@ExceptionHandler(value = BadSqlGrammarException.class)
public R exception(BadSqlGrammarException e){ //形参类型需要和异常处理注解配置的类型一致
    //打印异常的堆栈消息(包含了代码那些行出现了什么问题)
    log.error("出现了异常: {}", ExceptionUtils.getStackTrace(e));
    return R.setResult(ResponseEnum.BAD_SQL_GRAMMAR_ERROR);
}
```

## 7.3 自定义异常

```
编译时异常：
	编译时必须处理的异常
运行时异常：
	代码运行期间可能产生的异常叫做运行时异常
```

```java
@Getter
public class BusinessException extends RuntimeException{
    //用来接收将来发生异常时 希望返回的失败的R对象的code和msg
    private Integer code;
    private String msg;

    public BusinessException(Integer code,String msg){
        super(msg);
        this.code = code;
        this.msg = msg;
    }
    public BusinessException(ResponseEnum responseEnum){
        super(responseEnum.getMsg());
        this.code = responseEnum.getCode();
        this.msg = responseEnum.getMsg();
    }
}
```

## 7.4 自定义异常处理器+上层异常处理器

```java
@ExceptionHandler(value = BusinessException.class)
public R exception(BusinessException e){ //形参类型需要和异常处理注解配置的类型一致
    //打印异常的堆栈消息(包含了代码那些行出现了什么问题)
    log.error("出现了异常: {}", ExceptionUtils.getStackTrace(e));
    return R.error().code(e.getCode()).msg(e.getMsg());
}

@ExceptionHandler({
    NoHandlerFoundException.class,
    HttpRequestMethodNotSupportedException.class,
    HttpMediaTypeNotSupportedException.class,
    MissingPathVariableException.class,
    MissingServletRequestParameterException.class,
    TypeMismatchException.class,
    HttpMessageNotReadableException.class,
    HttpMessageNotWritableException.class,
    MethodArgumentNotValidException.class,
    HttpMediaTypeNotAcceptableException.class,
    ServletRequestBindingException.class,
    ConversionNotSupportedException.class,
    MissingServletRequestPartException.class,
    AsyncRequestTimeoutException.class
        })
public R handleServletException(Exception e) {
    log.error(e.getMessage(), e);
    //SERVLET_ERROR(-102, "servlet请求异常"),
    return R.error().msg(ResponseEnum.SERVLET_ERROR.getMsg()).code(ResponseEnum.SERVLET_ERROR.getCode());
}
```

## 7.5 测试

项目中对业务代码进行try-catch/或者业务判断 一旦出现异常或者业务不满足要求 抛出自定义异常对象并设置异常的code和msg描述异常信息



## 7.6 创建断言类优化业务判断抛出异常

srb-base中创建：

```java
public class Asserts {
    //对参数进行校验，是否为空  是否相等 是否为true/false  不满足要求抛出自定义类型的异常
    //断言obj不为空：如果obj为空抛出异常
    public static void assertNotNull(Object obj , ResponseEnum responseEnum){
        if(obj==null){
            throw new BusinessException(responseEnum);
        }
    }
    public static void assertNotNull(String str , ResponseEnum responseEnum ){
        if(str==null || str.length()==0){
            throw new BusinessException(responseEnum);
        }
    }

}
```

修改代码使用断言判断抛出异常：

```java
@PutMapping
public R updateById(@RequestBody IntegralGrade integralGrade) {
    Asserts.assertNotNull(integralGrade.getId(), ResponseEnum.PARAM_ERROR);
    return R.ok().data("flag", integralGradeService.updateById(integralGrade));
}
```

# 8、推送项目到远程仓库

初始化项目本地仓库

添加到暂存区+提交到本地仓库

创建空的远程仓库

推送本地仓库到远程仓库



srb远程仓库地址：

```
https://gitee.com/xgatguigu/sh230201-srb.git
```







```
1. 第一范式(1NF):每一列都是不可分割的原子数据项。即表中的每个列不可再分。
2. 第二范式(2NF):满足1NF,并且每个列都必须依赖于主键。即除主键列之外的每个列都必须完全依赖于主键。
3. 第三范式(3NF):满足2NF,并且每个非主键列都不能有传递依赖。即除主键列之外的每一列都必须直接依赖于主键,不能间接依赖。
三大范式的目的在于数据表的规范化设计,避免出现数据冗余与数据插入、删除、修改的异常。
```

