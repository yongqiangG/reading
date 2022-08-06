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

