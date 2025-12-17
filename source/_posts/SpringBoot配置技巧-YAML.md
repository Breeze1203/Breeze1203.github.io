---
title: SpringBoot配置技巧(YAML)
date: 2025-12-17 09:48:22
tags: [JAVA]
---

#### YAML锚点（复用通用配置段）
痛点：多个数据源配置几乎一样，只有 URL 不同，但你不得不把相同的配置复制好几遍。
解法：用 YAML 的锚点（&）和别名（*）来复用配置块。
```yaml
# 定义通用配置模板
common-datasource: &common-ds
  driver-class-name: com.mysql.cj.jdbc.Driver
  username: ${DB_USER:root}
  password: ${DB_PASSWORD:root}
  hikari:
    minimum-idle: 5
    maximum-pool-size: 20
    connection-timeout: 30000

spring:
  datasource:
    # 主库：继承通用配置，只改 URL
    primary:
      <<: *common-ds
      url: jdbc:mysql://master.pt.data:3306/pig

    # 从库：继承通用配置，只改 URL
    replica:
      <<: *common-ds
      url: jdbc:mysql://slave.pt.data:3306/pig
```
&common-ds 定义了一个锚点，*common-ds 引用它，<< 是合并键，表示把锚点的所有属性"铺开"到当前位置。
以后要改连接池大小？只改一处，两个数据源同时生效。复制粘贴？不存在的。
#### 敏感文件外部化
痛点：本地开发需要一些敏感配置（数据库密码、API Key），但这些配置不能提交到 Git。
解法：用 spring.config.import 导入本地 .env 文件。
```yaml
# application.yml
spring:
  config:
    # 从本地 .env 文件加载配置（不存在也不报错）
    import: optional:file:.env[.properties]
# 引用 .env 中的变量
app:
  api-key: ${MY_API_KEY}
  secret: ${MY_SECRET}
```
```
# .env 文件（加入 .gitignore）
MY_API_KEY=sk-xxxxxxxxxxxx
MY_SECRET=your-secret-here
```
optional: 前缀是重点。有这个前缀，文件不存在也不会报错。生产环境不需要这个文件（通过环境变量注入），本地开发通过 .env 文件提供。两边都能跑
#### 环境变量注入 + 安全默认值
痛点：用环境变量注入配置，但忘了设置环境变量，应用直接启动失败。
解法：用 ${VAR:default} 语法设置默认值。
```yaml
spring:
  datasource:
    # 优先读 DB_URL 环境变量，没有就用本地地址
    url: ${DB_URL:jdbc:mysql://localhost:3306/pig}
    username: ${DB_USER:root}
    password: ${DB_PASSWORD:root}

server:
  # 生产环境通过 SERVER_PORT 覆盖
  port: ${SERVER_PORT:8080}
```
这样设计的好处：开发环境直接启动，不用配任何环境变量；生产环境通过 K8s 的 env 或 ConfigMap 注入，覆盖默认值。
一套配置文件，到处都能跑。
#### 多Profile组合激活
痛点：生产环境要同时激活 prod、security、metrics 三个 Profile，启动命令很长
解法：用 Profile Groups 把多个 Profile 打包成一个
```yaml
# application.yml
spring:
  profiles:
    group:
      # 激活 production 时，自动激活这三个
      production:
        - proddb
        - security
        - metrics
      # 激活 development 时，自动激活这两个
      development:
        - devdb
        - swagger
```
启动命令从：
```shell
java -jar pig-gateway.jar --spring.profiles.active=prod,proddb,security,metrics
```
简化为：
```
java -jar pig-gateway.jar --spring.profiles.active=production
```
"生产环境包含哪些配置模块"这个知识，固化在配置文件里，而不是散落在运维脚本中
#### 属性交叉使用
痛点：服务名、版本号在配置文件里出现好几次，改一处漏一处。
解法：用 ${property.name} 交叉引用。
```shell
**pt:
  service:
    name: pt-gateway
    version: 4.0.0
  # 引用上面的值
  logging:
    path: /var/log/${pt.service.name}/
  metrics:
    prefix: ${pt.service.name}_${pt.service.version}
spring:
  application:
    # 也可以引用
    name: ${pt.service.name}**
```
改一处，全局生效。尤其是服务名、日志路径这种到处都要用的值，避免复制粘贴。
#### 单文件管理多环境配置
痛点：application-dev.yml、application-prod.yml、application-test.yml... 文件越来越多，改一个配置要改好几个文件。
解法：用 YAML 的多文档语法，在一个文件里管理所有环境。
```shell
# application.yml
# 通用配置（所有环境生效）
server:
  port: 8080
spring:
  application:
    name: pig-gateway
---
# 开发环境
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 9999
---
# 生产环境
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```
#### 条件导入外部配置
痛点：想支持客户自定义配置，但客户可能不提供这个文件，不提供就用默认值。
解法：用 optional: 前缀做条件导入。
```
spring:
  config:
    import:
      # 客户定制配置（可选）
      - optional:file:/etc/pig/custom-config.yml
      # 覆盖配置（可选）
      - optional:classpath:override.yml
```
如果 /etc/pt/custom-config.yml 存在，就加载它，覆盖默认配置；不存在，应用照常启动。
这个特性在做 SaaS 产品时特别有用：基础功能用默认配置，大客户可以通过挂载自定义配置文件来定制行为
#### 结构话配置+类型安全
痛点：用 @Value 注入配置，散落在各个类里，改个配置名要全局搜索；而且没有类型检查，写错了运行时才报错。
解法：用 @ConfigurationProperties 绑定到 Java Record。
```shell
pp:
  notification:
    retry:
      count: 3
      delay: 2s
      enabled: true
      channels:
        - sms
        - email
```
```java
package com.pig4cloud.pigx.common.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import java.time.Duration;
import java.util.List;

@ConfigurationProperties(prefix = "pig.notification.retry")
public record PigRetryProperties(
    int count,
    Duration delay,
    boolean enabled,
    List<String> channels
) {}
```
Java Record 天生不可变，线程安全。Duration 类型自动解析 "2s" 这样的时间表达式。IDE 还能自动补全配置项
#### 随机值生成
痛点：测试环境需要生成唯一 ID，或者想给每个实例一个随机端口避免冲突。
解法：用 ${random.*} 表达式
```shell
pig:
  instance-id: ${random.uuid}
  random-port: ${random.int[10000,65535]}
  session-seed: ${random.long}
server:
  # 开发环境随机端口，避免多实例冲突
  port: ${random.int[8000,9000]}
```
支持的格式：
- ${random.uuid} - UUID
- ${random.int} - 随机整数
- ${random.int(max)} - 0 到 max
- ${random.int[min,max]} - min 到 max
- ${random.long} - 随机长整数
有个坑：每次读取这个属性都会生成新值。如果你在两个 Bean 里注入同一个随机属性，它们会拿到不同的值。别问我怎么知道的。解决办法是把随机值绑定到一个 @ConfigurationProperties Bean，其他地方注入这个 Bean
#### 云平台自动感知
痛点：应用部署在 K8s 上时需要一些特殊配置（如优雅停机时间），但本地开发不需要。
解法：用 on-cloud-platform 条件激活。
```yaml
# 默认配置
spring:
  lifecycle:
    timeout-per-shutdown-phase: 10s
---
# 仅在 Kubernetes 环境生效
spring:
  config:
    activate:
      on-cloud-platform: kubernetes
  lifecycle:
    timeout-per-shutdown-phase: 30s
logging:
  level:
    root: INFO
```
Spring Boot 会自动检测运行环境（通过环境变量、文件系统特征等）。同一个 JAR 包，在本地跑用本地配置，部署到 K8s 自动切换成云上配置。
不用在启动脚本里传一堆参数了

注: [参考文章](https://mp.weixin.qq.com/s/78WgG3dT4kHAPMK9aQu8sw)（做大佬的搬运工，学习积累）