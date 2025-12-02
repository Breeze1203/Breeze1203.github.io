---
title: elasticsearch与kibana的配置
date: 2025-12-01 21:09:40
tags: [Elasticsearch]
---
#### 下载合适版本的[es](https://www.elastic.co/downloads/past-releases/elasticsearch-8-13-0)与[kibana](https://www.elastic.co/downloads/past-releases/kibana-8-13-0)

#### 启动es、浏览器访问localhost:9102
```shell
./bin/elasticsearch -d
```
让你输入用户名和密码，默认用户elastic，密码启动日志控制台会打印
#### 重置用户密码（忘记密码）
```shell
./bin/elasticsearch-reset-password -u elastic --url https://127.0.0.1:9200 
```
因为我是本地访问，所以指定url为 https://127.0.0.1:9200 

#### kibana连接es
##### 先生成一个专门的用户供Kibana 后台程序用来连接
```shell
./bin/elasticsearch-reset-password -u kibana_system
```

##### 编辑kibana配置文件
```yaml
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "Yk+Rq6cmIa9DCrsM6=Lr"
elasticsearch.ssl.certificateAuthorities: [ "/Users/pt/Downloads/elasticsearch-8.13.0/config/certs/http_ca.crt" ]
```
##### 启动kibana
```shell
nohup ./bin/kibana &
```
##### 查看日志文件里面的访问链接
```shell
tail -f nohup.log
```
##### 浏览器登录kibana
```yaml
user: elastic
password: 密码
```

#### 避免参数混淆
- server.ssl.enabled 作用：是否开启 HTTPS 默认值：false (关闭)。 如果设置为 true：你就必须通过 https://localhost:5601 访问，原来的 http:// 就无法访问了。

- server.ssl.certificate  作用：指定 公钥证书 文件的路径（.crt 或 .pem）。 这是 Kibana 的“身份证”，发给浏览器看的，证明它是安全的。

- server.ssl.key 作用：指定 私钥 文件的路径（.key）比喻：这是 Kibana 的“私章”，用来解密浏览器发来的数据，必须严格保密


- elasticsearch.ssl.certificateAuthorities 作用：指定 CA 根证书 文件的路径（通常是 http_ca.crt）。 角色：这是 Kibana 用来验证 Elasticsearch 身份的“信任名单”。

- elasticsearch.ssl.certificate 作用：指定 客户端公钥证书 的路径。 角色：这是 Kibana 发给 Elasticsearch 看的“员工工牌”（身份证明）。 详解：用于双向认证（mTLS）。当 Elasticsearch 设置了“不许用密码登录，只许刷卡进入”这种高安全级别策略时，Kibana 就必须出示这个文件来证明自己是谁。

- elasticsearch.ssl.key 作用：指定 客户端私钥 的路径。 角色：这是 Kibana 的“私章”或“指纹”。 详解：它必须和上面的 elasticsearch.ssl.certificate 成对使用。用来证明刚才出示的那个“员工工牌”确实是 Kibana 自己的，而不是捡来的或偷来的。


