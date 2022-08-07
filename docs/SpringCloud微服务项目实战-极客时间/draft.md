# 优惠券系统单体向微服务的演进

## maven依赖管理

### parent标签
通过顶层parent标签指定spring-boot-starter-parent版本，SpringBoot组件的版本信息就会被自动带到子模块，避免了组件之间版本的兼容性问题

### packaging标签
1. jar
2. war
3. pom（表示当前模块是一个boss，不包含任何实现，只负责定义版本和整合子模块）

### dependencymanagement标签
类似parent标签，负责声明版本并向下传递版本信息，子模块就不需要再指定version

### 抽象模式
将通用的部分封装在抽象类，具体的个性部分延迟到子类实现，再定义接口，对外暴露通用方法。
例如mq消息消费

### 工厂模式
通过工厂模式和接口，尽可能对上层业务屏蔽其底层业务复杂度。这样子底层的修改对上层就是无感知的，这也是开闭思想的原则。

### 数据冗余与数据异构
以空间换时间。通过将一份数据冗余或者异构到多处，提升查询效率。

### 包扫描声明
在多模块项目重，组件扫描通过声明包名来扫描到其他模块的包。


## Nacos

### 数据模型
namespace：可以作为环境或者多租户的隔离
group：可以作为环境或者多租户的隔离
dataId/service：服务名称

例如：prod.A.orderService

### 保障高可用的两个方向
1. 避单点故障
2. 故障恢复

### nacos的部署
#### k8s部署方式
1. 构建镜像
2. yaml文件

yaml文件示例：
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
spec:
  serviceName: nacos
  replicas: 1
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      containers:
        - name: nacos
          imagePullPolicy: IfNotPresent
          image: nacos/nacos-server
          resources:
            requests:
              memory: "512Mi"
              cpu: "200m"
          ports:
            - containerPort: 8848
          env:
            - name: MYSQL_DATABASE_NUM
              value: "0"
            - name: MODE
              value: "standalone"
  selector:
    matchLabels:
      app: nacos
---
apiVersion: v1
kind: Service
metadata:
  name: nacos-service
  labels:
    name: nacos-service
spec:
  type: NodePort
  ports:
  - port: 8848
    protocol: TCP
    targetPort: 8848
    nodePort: 32018
  selector:
    app: nacos
```

#### nacos配置项
1. 指定数据源为MySQL，默认为嵌入式数据库
2. 指定DB实例数
3. 修改JDBC连接串
4. nacos数据库初始化

#### Spring集成nacos
> 注意事项

SpringBoot、SpringCloud、SpringCloudAlibaba三者的版本需要严格匹配。
例如：Spring Cloud 2020.0.1、Spring Cloud Alibaba2021.1 和 Spring Boot 2.4.2

#### 顶层dependenceManagement定义springCloud和springCloudAlibaba版本
```
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>2020.0.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2021.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
```

#### nacos自动装配
当引入Naocs依赖之后，默认自动装配。

#### nacos重要配置项
```
spring:
  cloud:
    nacos:
      discovery:
        # 可以配置多个，逗号分隔
        server-addr: ip:port
        # 默认就是application name，一般不用配置
        service: coupon-template-serv
        # nacos客户端向服务端发送心跳的时间间隔，时间单位其实是ms
        heart-beat-interval: 5000
        # 服务端没有接受到客户端心跳请求就将其设为不健康的时间间隔，默认为15s
        # 注：推荐值该值为15s即可，如果有的业务线希望服务下线或者出故障时希望尽快被发现，可以适当减少该值
        heart-beat-timeout: 15000
        # [注意] 这个IP地址如果更换网络后变化，会导致服务调用失败，建议先不要设置
        # ip: 172.20.7.228
        # 元数据部分 - 可以自己随便定制
        metadata:
          mydata: abc
        # 客户端在启动时是否读取本地配置项(一个文件)来获取服务列表
        # 注：推荐该值为false，若改成true。则客户端会在本地的一个文件中保存服务信息，当下次宕机启动时，会优先读取本地的配置对外提供服务。
        naming-load-cache-at-start: false
        # 创建不同的集群
        cluster-name: Cluster-A
        # 命名空间ID，Nacos通过不同的命名空间来区分不同的环境，进行数据隔离，
        # 服务消费时只能消费到对应命名空间下的服务。
        # [注意]需要在nacos-server中创建好namespace，然后把id copy进来
        namespace: dev
        # [注意]两个服务如果存在上下游调用关系，必须配置相同的group才能发起访问
        group: Coupon-svc
        # 向注册中心注册服务，默认为true
        # 如果只消费服务，不作为服务提供方，倒是可以设置成false，减少开销
        register-enabled: true
        # 类似长连接监听服务端信息变化的功能
        watch:
          enabled: true
        watch-delay: 30000
```
### nacos的namespace和group
实现环境隔离+多租户隔离

不同group的服务无法访问

#### 其他用法
1. 利用group不同的服务无法访问，部署新服务完成线上测试。旧的group不会将流量打到新的group，完成线上测试。
2. 单元封闭，优先请求本地同机房的服务。不同区域的机房使用不同的group。

### nacos服务发现底层实现
客户端nacos client主动轮询从服务端拉取地址列表、group分组。

该定时任务由UpdateTask实现。

## loadbalance
### 负载均衡方式
1. 网关负载均衡，通过网关多一次请求转发
2. 客户端负载均衡

### 如何将loadbalance注入httpClient
1. 在client插入过滤器
2. 在请求之前通过过滤器和策略使用客户端的负载均衡策略

### 自定义loadbalance规则实现金丝雀发布
1. 实现重写负载均衡策略，通过请求头的流量打标匹配metadata中存在相同值的instance
2. 请求的客户端传入指定的请求头
3. 添加nacos元数据

## OpenFeign
### 如何实现远程调用
1. 项目启动，通过enableFeignClient开始加载FeignClient组件
2. FeignClientRegistrar通过扫描feignClient来构造出FeignClientFactoryBean
3. 通过FeignClientFactoryBean解析出请求路径、降级方法
4. 生成动态代理类，实现远程http调用

### 使用feignClient
有服务提供方统一实现，消费方直接extend服务并@FeignClient即可

FeignClient的负载均衡loadBalance是懒加载的，注意增加connectTimeout+retry

### feign日志级别
Full与Basic性能差异较大

```
@Bean
Logger.Level feignLogger() {
    return Logger.Level.FULL;
}
```

### feign超时设置
可以针对特定的调用服务设置不同的连接时间和读取时间

### feign的降级
相比较sentinel实现更加轻量，依赖于hystrix，但是需要移除ribbon组件，避免和loadBalance冲突。
```
<!-- hystrix组件，专门用来演示OpenFeign降级 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>2.2.10.RELEASE</version>
    <exclusions>
        <!-- 移除Ribbon负载均衡器，避免冲突 -->
            <exclusion>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-netflix-ribbon</artifactId>
            </exclusion>
    </exclusions>
</dependency>
```

1. 降级类
2. 降级工厂，可以拿到错误原因

## 配置中心

### 常见场景
1. 业务开关
2. 业务规则，例如经常变动的文案信息
3. 灰度发布。将特定配置项推送到特定ip的机器，完成生产测试

### 使用Nacos Config
> 引入依赖

```
<!-- 添加Nacos Config配置项 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!-- 读取bootstrap文件 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>

```

> 相关配置

```
spring:
  application:
    name: coupon-customer-serv
  cloud:
    nacos:
      config:
        server-addr: ip:port
        file-extension: yml
        # prefix: 文件名前缀，默认是spring.application.name
        # 如果没有指定命令空间，则默认命令空间为PUBLIC
        namespace: dev
        # 如果没有配置Group，则默认值为DEFAULT_GROUP
        group: DEFAULT_GROUP
        # 从Nacos读取配置项的超时时间
        timeout: 5000
        # 长轮训超时时间，长轮询可以降低多次请求带来的网络开销，服务端hold住请求一段时间，当有配置变更再返回
        config-long-poll-timeout: 1000
        # 重试时间
        config-retry-time: 100000
        # 长轮训重试次数
        max-retry: 3
        # 开启监听和自动刷新
        refresh-enabled: true
        # Nacos的扩展配置项，数字越大优先级越高
        enable-remote-sync-config: true
        # 从多个文件读取配置信息
        extension-configs:
          # 文件名
          - dataId: coupon-config.yml
            # 分组
            group: EXT_GROUP
            refresh: true
          # 可以添加其他节点
          - dataId: rabbitmq-config.yml
            group: EXT_GROUP
            refresh: true
```

> 项目中

@refreshScope+@value

## 服务容错
### 降级与熔断
降级：请求失败的planB

熔断：短时间内达到一个错误阈值，接下来的请求直接走降级逻辑，一段时间后尝试恢复

### 限流模型
- 快速失败
- 预热模型，慢慢拉高限流阈值
- 排队模型，达到阈值后不立刻失败，而是先放到队列

### sentinel的工作模型
责任链的设计模式，通过slotChain将各样的规则和统计串起来。

在sentinel中一切皆是资源。

### 接入sentinel
1. 部署sentinel服务
2. 引入pom依赖

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

3. 配置文件配置sentinel服务地址
```
spring:
  cloud:
    sentinel:
      transport:
        # sentinel api端口，默认8719
        port: 8719
        # dashboard地址
        dashboard: localhost:8080
```

4. 默认会为API的Uri路径生成一个资源

5. 自定义使用
```
@GetMapping("/getBatch")
@SentinelResource(value = "getTemplateInBatch", blockHandler = "getTemplateInBatch_block")
public Map<Long, CouponTemplateInfo> getTemplateInBatch(@RequestParam("ids") Collection<Long> ids) {
    ...
}

public Map<Long, CouponTemplateInfo> getTemplateInBatch_block(
Collection<Long> ids, BlockException exception) {
    log.info("接口被限流");
    return Maps.newHashMap();
}
```

### 流控方式
1. 直接流控
2. 关联流控，例如两个服务共享DB连接数
3. 链路流控，例如两个链路中的共享节点

### 使用注意
- 默认blockhandler只处理blockException
- 可以通过指定fallback指定处理所有运行期抛出的异常

### 时间窗口与熔断
在进入熔断逻辑之后一段时间，会进入半开状态，这时候如果一次请求成功，就退出熔断状态，否则继续保持熔断状态。

### 如何持久化sentinel规则
1. sentinel将限流规则保存到nacos
2. 应用从nacos读取限流规则
3. sentinel重启后从nacos加载限流规则


## 链路追踪

### sleuth的实现
1. traceId
2. spanId
3. parentSpanId

> 耗时统计

1. 客户端发起请求时间
2. 服务端接收到请求时间
3. 服务端返回请求时间
4. 客户端接收到请求返回时间

### 使用sleuth
1. pom引入
```
<!-- Sleuth依赖项 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```
2. 配置采样比例
```
spring:
  sleuth:
    sampler:
      # 采样率的概率，100%采样
      probability: 1.0
      # 每秒采样数字最高为100
      rate: 1000
```
3.
使用zipkin可视化，在zipkin和应用之间可以增加kafka等消息中间件提高消息的送达率和传递效率
4. 如果采用了rabbitmq作为中间件，zipkin启动增加rabbitmq参数，引入stream依赖
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

设置绑定相关队列

```
spring:
  zipkin:
    sender:
      type: rabbit
    rabbitmq:
      addresses: 127.0.0.1:5672
      queue: zipkin
```
5. 为zipkin设置数据源，默认保存在缓存中，可切换成es、MySQL、cassandra

## 日志检索
### ELK
- logstash：日志采集器，简单的初步过滤
- es：日志存储和分词
- kibana：可视化界面

### 使用elk
1. 配置logstach的输入输出源
2. 应用对接elk

pom引入
```
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.0.1</version>
</dependency>
```
logback-spring配置文件
```
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>DEBUG</level>
    </filter>
<!-- 日志输出编码 -->
    <encoder>
        <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        <charset>utf8</charset>
    </encoder>
</appender>
<appender name="logstash"
class="net.logstash.logback.appender.LogstashTcpSocketAppender">
<!-- 这是Logstash的连接方式 -->
<destination>127.0.0.1:5044</destination>
<!-- 日志输出的JSON格式 -->
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEnco
<providers>
<timestamp>
<timeZone>UTC</timeZone>
</timestamp>
<pattern>
<pattern>
{
"severity": "%level",
"service": "${applicationName:-}",
"trace": "%X{traceId:-}",
"span": "%X{spanId:-}",
"pid": "${PID:-}",
"thread": "%thread",
"class": "%logger{40}",
"rest": "%message"
}
</pattern>
</pattern>
</providers>
</encoder>
</appender>
```

### elk的高可用
不要将应用与logstash直连，而是将日志罗盘写入文件，避免日志丢失。

然后通过filebeat之类的工具将日志传输到logstash。如果logstash正忙，它会自动降低读取文件的速度。

## 网关
### 网关
第一道网关VIP，DNS解析成VIP对应到一个服务集群，为了避免单点故障，这里可以将DNS映射到两个VIP，形成主备。

第二道网关Nginx，可能是多层Nginx

第三道网关微服务网关，SpringGateway，zuul

### 微服务网关的好处
1. 服务的统一切面，可定制性强
2. 高扩展性，服务集群在缩扩容时，网关可以通过服务注册中心无感知获取节点的变动。如果没有微服务网关，可能缩扩容就需要手动维护VIP对应的节点列表

### gateway的路由规则
1. 路由
2. 谓语，是否满足路由条件
3. 过滤器，全局过滤器和局部过滤器

### 加载路由的方式
1. java hardcode
```
@Bean
public RouteLocator declare(RouteLocatorBuilder builder) {
    return builder.routes()
    .route("id-001", route -> route
    .path("/geekbang/**")
    .uri("http://time.geekbang.org")
    ).route(route -> route
    .path("/test/**")
    .uri("http://www.test.com")
    ).build();
}
```
2. yaml hardcode
3. 动态路由（nacos）

### 谓语
> 限定请求方式

```
.route("id-001", route -> route
.path("/geekbang/**")
.and().method(HttpMethod.GET, HttpMethod.POST)
.uri("http://time.geekbang.org")

```

> 验证请求参数，但是不建议在网关做太多的业务相关参数的校验，顶多做一些鉴权和登录状态的判断

```
.route("id-001", route -> route
// 验证cookie
.cookie("myCookie", "regex")
// 验证header
.and().header("myHeaderA")
.and().header("myHeaderB", "regex")
// 验证param
.and().query("paramA")
.and().query("paramB", "regex")
.and().remoteAddr("远程服务地址")
.and().host("pattern1", "pattern2")
)
```

> 时间谓语，可用于秒杀

```
.route("id-001", route -> route
// 在指定时间之前
.before(ZonedDateTime.parse("2022-12-25T14:33:47.789+08:00"))
// 在指定时间之后
.or().after(ZonedDateTime.parse("2022-12-25T14:33:47.789+08:00"))
// 或者在某个时间段以内
.or().between(
ZonedDateTime.parse("起始时间"),
ZonedDateTime.parse("结束时间"))
```

> 可以自定义实现谓语

### 实际使用
最常用的谓语一般是path，其他大部分谓语一般不需要使用。如果需要鉴权或者登陆状态校验，一般使用gateway的过滤器。

### 网关设置跨域
前后端谓语不同的根域名下，前端通过nodejs访问了后端，这时候就发生了跨域请求，浏览器根据同源策略，先探测一下后端是否支持跨域和受信，否则就阻止本次访问。

网关配置
```
server:
port: 30000
spring:
# 分布式限流的Redis连接
redis:
host: localhost
port: 6379
cloud:
nacos:
# Nacos配置项
discovery:
server-addr: localhost:8848
heart-beat-interval: 5000
heart-beat-timeout: 15000
cluster-name: Cluster-A
namespace: dev
group: myGroup
register-enabled: true
gateway:
discovery:
locator:
# 创建默认路由，以"/服务名称/接口地址"的格式规则进行转发
# Nacos服务名称本来就是小写，但Eureka默认大写
enabled: true
lower-case-service-id: true
# 跨域配置
globalcors:
cors-configurations:
'[/**]':
# 授信地址列表
allowed-origins:
- "http://localhost:10000"
- "https://www.geekbang.com"
# cookie, authorization认证信息
expose-headers: "*"
allowed-methods: "*"
allow-credentials: true
allowed-headers: "*"
# 浏览器缓存时间
max-age: 1000
```

```
@Configuration
public class RoutesConfiguration {
@Bean
public RouteLocator declare(RouteLocatorBuilder builder) {
return builder.routes()
.route(route -> route
.path("/gateway/coupon-customer/**")
.filters(f -> f.stripPrefix(1))
.uri("lb://coupon-customer-serv")
).route(route -> route
.order(1)
.path("/gateway/template/**")
.filters(f -> f.stripPrefix(1))
.uri("lb://coupon-template-serv")
).route(route -> route
.path("/gateway/calculator/**")
.filters(f -> f.stripPrefix(1))
.uri("lb://coupon-calculation-serv")
).build();
}
}
```
stripPrefix可以去除第一层路径

### 网关+redis+lua限流
1. 自定义限流维度和限流值（每秒产生令牌数量+令牌桶容量）
2. 添加限流
```
.route(route -> route.path("/gateway/coupon-customer/**")
.filters(f -> f.stripPrefix(1)
.requestRateLimiter(limiter-> {
limiter.setKeyResolver(hostAddrKeyResolver);
limiter.setRateLimiter(customerRateLimiter);
// 限流失败后返回的HTTP status code
limiter.setStatusCode(HttpStatus.BANDWIDTH_LIMIT_EXCEEDED);
}
)
)
.uri("lb://coupon-customer-serv")
```

### gateway+nacos实现动态路由

## 消息驱动
### 消息单播
消息只被消费一次
### 消息广播
消息被所有消费者消费一次

场景：热点数据的广播，令所有服务执行热点逻辑，例如构建本地缓存，而非远程缓存。

### 延迟消息
- 订单确认
- 自动取消未确认订单

### 消费异常
1. 设置消息消费重试次数
2. 重新丢回队列
3. 设置消费降级策略，例如通知人工介入
4. 死信队列，待故障修复后再将消息丢回原始队列，或者针对死信队列设置一个消费监听。

## 分布式事务
再复杂的分布式事务最终都要回到本地事务。

### XA协议

### 2PC
1. 协调者下发准备指令
2. 都准备好之后下发提交指令，如果失败，回滚。

问题：准备阶段一直占用DB连接，不适合高并发，通过长事务保证一致性。

### seata服务搭建
1. 更改持久化模式和DB连接方式，执行数据库初始化脚本
2. 将seata服务注册到nacos

### at模式
1. TM事务发起者向TC申请全局XID，RM开启事务并记录undoLog
2. 通过seata组件传递XID，下一个RM也开始事务并记录undoLog
3. 由第一个发起事务的TM角色决定commit或者rollback
4. TC异步发起commit或者rollback，清空undoLog

### 使用seata
1. 引入pom
```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```
2. 使用seata数据源代理，拥有生成undolog和注册事务分支的功能
```
@Configuration
public class SeataConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DruidDataSource druidDataSource() {
        return new DruidDataSource();
    }
    @Bean("dataSource")
    @Primary
    public DataSource dataSourceDelegation(DruidDataSource druidDataSource) {
        return new DataSourceProxy(druidDataSource);
    }
}
```
3. seata配置项
```
spring:
cloud:
alibaba:
seata:
tx-service-group: seata-server-group
seata:
application-id: coupon-customer-serv
registry:
type: nacos
nacos:
application: seata-server
server-addr: localhost:8848
namespace: dev
group: myGroup
cluster: default
service:
vgroup-mapping:
seata-server-group: default
```
4. 使用示例
```
// GlobalTranscational标记了这是一个分布式事务，任何异常都会导致回滚
@DeleteMapping("template")
@GlobalTransactional(name = "coupon-customer-serv", rollbackFor = Exception.class)
public void deleteCoupon(@RequestParam("templateId") Long templateId) {
    customerService.deleteCouponTemplate(templateId);
}

// 使用OpenFeign接口发起远程调用使优惠券模板失效，然后再注销本地用户身上的优惠券，这时候如果抛出一个异常，可以将远程优惠券模板服务的数据库更新回滚掉
@Override
@Transactional
public void deleteCouponTemplate(Long templateId) {
    templateService.deleteTemplate(templateId);
    couponDao.deleteCouponInBatch(templateId, CouponStatus.INACTIVE);
    // 模拟分布式异常
    throw new RuntimeException("AT分布式事务挂球了");
}
```

### seata总结
非必要不要上分布式事务！

seata的at模式基本无侵入，属于最简单的分布式事务一致性方案

对于简单的本地业务，建议直接使用传统的事务+日志补偿+批量服务的方式即可。

### TCC模式

#### 三阶段
try：锁定资源。新增lock字段记录资源锁定状态。

confirm：执行业务操作，并解锁资源。

concel：人工回滚，解锁资源

#### 使用TCC demo
0. 注册TCC
```
@LocalTCC
public interface CouponTemplateServiceTCC extends CouponTemplateService {
@TwoPhaseBusinessAction(
name = "deleteTemplateTCC",
commitMethod = "deleteTemplateCommit",
rollbackMethod = "deleteTemplateCancel"
)
void deleteTemplateTCC(@BusinessActionContextParameter(paramName = "id") L

void deleteTemplateCommit(BusinessActionContext context);

void deleteTemplateCancel(BusinessActionContext context);
}
```
1. 锁定资源
```
@Override
@Transactional
public void deleteTemplateTCC(Long id) {
    CouponTemplate filter = CouponTemplate.builder()
    .available(true)
    .locked(false)
    .id(id)
    .build();
    CouponTemplate template = templateDao.findAll(Example.of(filter))
    .stream().findFirst()
    .orElseThrow(() -> new RuntimeException("Template Not Found"));
    template.setLocked(true);
    templateDao.save(template);
}
```
2. 执行使得优惠券模板失效
```
@Override
@Transactional
public void deleteTemplateCommit(BusinessActionContext context) {
    Long id = Long.parseLong(context.getActionContext("id").toString());
    CouponTemplate template = templateDao.findById(id).get();
    template.setLocked(false);
    template.setAvailable(false);
    templateDao.save(template);
    log.info("TCC committed");
}
```
3. cancel方法
```
@Override
@Transactional
public void deleteTemplateCancel(BusinessActionContext context) {
    Long id = Long.parseLong(context.getActionContext("id").toString());
    Optional<CouponTemplate> templateOption = templateDao.findById(id);
    if (templateOption.isPresent()) {
    CouponTemplate template = templateOption.get();
    template.setLocked(false);
    templateDao.save(template);
    }
    log.info("TCC cancel");
}
```
#### 空回滚问题
try阶段由于网络原因不可执行或者执行失败，直接执行了cancel逻辑。

因此在cancel阶段需要判断try是否执行成功。

一种方式是根据资源是否被锁定来判断，另一种比较完善的方式是引入独立的事务控制表，try阶段将xid和事务分支id落库，如果cancel阶段没有查询到xid则表示try阶段未执行。

#### 倒悬问题
当try由于网络原因或者网关阻塞，先执行了cancel操作，而后又执行了try操作，造成资源被长期锁定。

如果引入了独立的事务控制表，可以在执行try之前检查二阶段的执行状态，如果已经回滚，那么直接跳过try执行。


## 主链路分析

### 漏斗模型
- 用户获取，导流端。导购、入会
- 用户激活（转化+留存）。活动、营销
- 收益端。下单

从上往下，流量递减，流量质量递增。

### 主链路特征
1. 业务完整性
2. 转化率重因子
3. 流端占比
4. 现金水库
