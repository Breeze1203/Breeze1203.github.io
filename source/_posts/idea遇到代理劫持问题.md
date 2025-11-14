---
title: idea遇到代理劫持问题
date: 2025-11-13 20:18:31
categories: Essay
---

最近我的 IDEA 因为 Clash 的问题，出现了各种奇奇怪的问题，就这些问题的解决做一个简单的记录。

1、You have JVM property "https.proxyHost" set to "127.0.0.1"
由于我在 Mac 上开了 Clash 代理软件，接管了系统代理，打开 IDEA 的 Appearance & Behavior --> System Settings --> HTTP Proxy 界面，提示 You have JVM property "https.proxyHost" set to "127.0.0.1"，解决方案就是：移除掉 Java 自带的 http 和 socket 代理，采用系统代理，选择 Help -> Edit Custom VM Options，增加如下的配置:
```shell
-Dhttp.proxyHost
-Dhttp.proxyPort
-Dhttps.proxyHost
-Dhttps.proxyPort
-DsocksProxyHost
-DsocksProxyPort
```
重启 IDEA 即可解决。

2、刷新 Maven 项目依赖，Build 控制台报 status: 502 Bad Gateway
某些依赖无法直接访问，这个只需要在 Maven 的 setting.xml 配置文件中，增加 HTTP 代理就行，让 Maven 强制走 Clash 代理，比如我的 Clash 的 HTTP 代理端口是 7890，则配置如下：
```xml
<proxies>
    <proxy>
        <id>clash proxy</id>
        <active>true</active>
        <protocol>http</protocol>
        <host>127.0.0.1</host>
        <port>7890</port>
    </proxy>
</proxies>

```




