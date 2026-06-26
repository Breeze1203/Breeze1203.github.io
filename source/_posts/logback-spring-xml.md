---
title: logback-spring.xml
date: 2026-06-26 14:51:17
categories: Essay
---

下面这 5 个工程级配置最核心：

```text
1. app.log     主业务日志
2. sql.log     SQL 日志
3. slow-sql.log 慢 SQL 日志
4. http.log    HTTP 请求日志
5. error.log   错误日志
```

## 一、推荐完整目录

```text
logs/${project.artifactId}/
├── app.log
├── sql.log
├── slow-sql.log
├── http.log
└── error.log
```

---

## 二、工程级 logback-spring.xml 模板

### 1. 公共变量

```xml
<property name="log.path" value="logs/${project.artifactId}"/>

<property name="FILE_LOG_PATTERN"
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{50} - %msg%n"/>
```

含义：

```text
log.path        日志根目录
%d              时间
%thread         当前线程名
%-5level        日志级别，左对齐，占 5 位
%X{traceId}     MDC 里的 traceId，做链路追踪用
%logger{50}     logger 名称，最长 50 个字符
%msg            日志正文
%n              换行
```

### 1. app.log：主业务日志

```xml
<appender name="APP_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}/app.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${log.path}/%d{yyyy-MM}/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>50MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>5GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>${FILE_LOG_PATTERN}</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

参数含义：

```text
RollingFileAppender      滚动文件输出器
file                     当前正在写入的日志文件
rollingPolicy            日志滚动策略
SizeAndTimeBasedRollingPolicy 按时间 + 文件大小滚动
fileNamePattern          历史日志文件命名规则
%d{yyyy-MM}              按月份建目录
%d{yyyy-MM-dd}           按天切日志
%i                       同一天内第几个分片
.gz                      自动压缩历史日志
maxFileSize              单个日志文件最大大小
maxHistory               最多保留多少天/月的历史日志
totalSizeCap             历史日志总大小上限
encoder                  日志编码器
pattern                  日志格式
charset                  文件编码
```

对应 logger：

```xml
<logger name="com.pig4cloud.pigx.fin" level="INFO" additivity="false">
    <appender-ref ref="APP_FILE"/>
    <appender-ref ref="ERROR_FILE"/>
</logger>
```

---

## 2. sql.log：MyBatis SQL 日志

需要把 MyBatis-Plus 改成走 Slf4j：

```yaml
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
```

SQL appender：

```xml
<appender name="SQL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}/sql.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${log.path}/%d{yyyy-MM}/sql.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>15</maxHistory>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{50} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

logger：

```xml
<logger name="com.pig4cloud.pigx.fin.mapper" level="DEBUG" additivity="false">
    <appender-ref ref="SQL_FILE"/>
</logger>
```

含义：

```text
logger name     指定哪个包的日志进入这个文件
level="DEBUG"   MyBatis SQL 通常是 DEBUG 级别
additivity=false 不再向 root 继续传递，避免 sql.log、app.log、console 重复输出
appender-ref    指定输出到哪个 appender
```

---

## 3. slow-sql.log：慢 SQL 日志

MyBatis 自带日志不太适合自动区分慢 SQL，推荐用 **P6Spy** 或自定义拦截器。

如果你用 P6Spy，可以设置：

```properties
# spy.properties
executionThreshold=1000
outagedetection=true
outagedetectioninterval=1
```

含义：

```text
executionThreshold=1000     超过 1000ms 的 SQL 才记录，单位毫秒
outagedetection=true        开启慢查询检测
outagedetectioninterval=1   慢查询检测间隔，单位秒
```

Logback appender：

```xml
<appender name="SLOW_SQL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}/slow-sql.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${log.path}/%d{yyyy-MM}/slow-sql.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>50MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>2GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%X{traceId}] %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

logger：

```xml
<logger name="p6spy" level="INFO" additivity="false">
    <appender-ref ref="SQL_FILE"/>
</logger>

<logger name="com.p6spy.engine.outage" level="INFO" additivity="false">
    <appender-ref ref="SLOW_SQL_FILE"/>
</logger>
```

---

## 4. http.log：HTTP 请求日志

这个需要你在代码里单独打日志，比如 Filter 或 AOP。

Logback appender：

```xml
<appender name="HTTP_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}/http.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${log.path}/%d{yyyy-MM}/http.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>15</maxHistory>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{traceId}] %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

logger：

```xml
<logger name="HTTP_LOG" level="INFO" additivity="false">
    <appender-ref ref="HTTP_FILE"/>
</logger>
```

Java 里这样打：

```java
private static final Logger HTTP_LOG = LoggerFactory.getLogger("HTTP_LOG");

HTTP_LOG.info("method={} uri={} status={} cost={}ms ip={}",
        method, uri, status, cost, ip);
```

---

## 5. error.log：错误日志

```xml
<appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}/error.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${log.path}/%d{yyyy-MM}/error.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>50MB</maxFileSize>
        <maxHistory>60</maxHistory>
        <totalSizeCap>5GB</totalSizeCap>
    </rollingPolicy>

    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{50} %file:%line - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>

    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>ERROR</level>
    </filter>
</appender>
```

关键是这个：

```xml
<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
    <level>ERROR</level>
</filter>
```

含义：

```text
ThresholdFilter  阈值过滤器
level=ERROR      只有 ERROR 及以上级别才进入 error.log
```

root：

```xml
<root level="INFO">
    <appender-ref ref="console"/>
    <appender-ref ref="APP_FILE"/>
    <appender-ref ref="ERROR_FILE"/>
</root>
```

---

### 三、最关键的 Logback 参数解释

### logger

```xml
<logger name="com.xxx.mapper" level="DEBUG" additivity="false">
```

```text
name        包名、类名，或自定义 logger 名称
level       日志级别：TRACE < DEBUG < INFO < WARN < ERROR
additivity  是否继续向父 logger/root 传递
```

`additivity=false` 很重要。
不加的话，SQL 可能同时出现在：

```text
sql.log
app.log
console
```

---

### appender

```xml
<appender name="SQL_FILE" class="...RollingFileAppender">
```

```text
name    appender 的名字，logger 通过 appender-ref 引用
class   输出器类型，比如控制台、文件、滚动文件
```

常用类型：

```text
ConsoleAppender        控制台
FileAppender           普通文件
RollingFileAppender    可滚动文件，最常用
AsyncAppender          异步日志，高并发项目常用
```

---

### rollingPolicy

```xml
<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
```

常用策略：

```text
TimeBasedRollingPolicy          只按时间滚动
SizeAndTimeBasedRollingPolicy   按时间 + 大小滚动，生产推荐
```

---

### fileNamePattern

```xml
<fileNamePattern>${log.path}/%d{yyyy-MM}/sql.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
```

```text
%d{yyyy-MM}       月份目录
%d{yyyy-MM-dd}    每天一个日志
%i                同一天文件过大时的序号
.gz               历史日志压缩
```

---

### encoder / pattern

```xml
<encoder>
    <pattern>%d [%thread] %-5level [%X{traceId}] %logger{50} - %msg%n</pattern>
</encoder>
```

常用占位符：

```text
%d                 时间
%level             日志级别
%-5level           左对齐日志级别
%thread            线程名
%logger            logger 名
%logger{50}        logger 名最多 50 个字符
%class             类名，性能较差
%method            方法名，性能较差
%file              文件名，性能较差
%line              行号，性能较差
%msg               日志内容
%n                 换行
%ex                异常堆栈
%wEx               Spring Boot 扩展异常格式
%X{traceId}        MDC traceId
```

生产环境不建议大量使用：

```text
%class
%method
%line
```

因为获取调用栈有性能损耗。

---

## 四、推荐最终策略

开发环境：

```text
console + app.log + sql.log + error.log
SQL DEBUG 打开
```

测试环境：

```text
app.log + sql.log + error.log + slow-sql.log
SQL DEBUG 可开
```

生产环境：

```text
app.log + error.log + slow-sql.log
普通 SQL 建议关闭或只短期开启
```

生产建议：

```xml
<logger name="com.pig4cloud.pigx.fin.mapper" level="INFO" additivity="false">
    <appender-ref ref="SQL_FILE"/>
</logger>
```
