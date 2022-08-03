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
