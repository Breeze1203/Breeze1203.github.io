---
title: 一文讲清TLS
date: 2025-10-18 20:09:22
categories: SSL/TLS
---

### 分清两种场景 (TLS 与 mTLS)
**基础：** 无论哪种场景，**创建我们自己的 CA**。这个 CA 是所有信任的根源。
####  步骤 1: 创建 CA (信任的根基)
```shell
openssl req -x509 -newkey rsa:4096 -days 365 -nodes  -keyout ca-key.pem -out ca_cert.pem -subj "/C=US/ST=CA/L=Mountain View/O=gRPC-Study/CN=ca"
```
  * ca-key.pem (CA私钥): **最高机密**，用来签发证书。
  * ca_cert.pem (CA证书): **公开**，分发给所有人，用来验证证书。
### 场景 A：标准 TLS (服务器单向认证)
**目标：** 客户端验证服务器的身份，确保连接是加密的、且没有连错服务器。
#### 步骤 2: 生成服务器证书
```shell
# 生成服务器私钥
openssl genrsa -out server_key.pem 4096
```
```shell
# 生成服务器 CSR
openssl req -new -key server_key.pem -out server.csr -subj "/C=US/ST=CA/L=Mountain View/O=gRPC-Study/CN=localhost"
```
### CA 签发服务器证书
```shell
openssl x509 -req -in server.csr -CA ca_cert.pem -CAkey ca-key.pem -CAcreateserial -out server_cert.pem -days 365 -addext "subjectAltName = DNS:localhost,IP:127.0.0.1" 
```
**原因：** 现代 TLS 客户端会忽略CN=localhost 字段，转而只检查 subjectAltName (SAN) 字段。如果你不加这个，gRPC 客户端会报错，认为证书对 localhost 无效。
#### 场景 A 的配置与握手
  * **服务器需要：**
    1.  server_key.pem (自己的私钥)
    2.  server_cert.pem (自己的证书)
  * **客户端需要：**
    1.  ca_cert.pem (用来验证服务器)
  * **握手流程 (单向)：**
    1.  客户端连接服务器。
    2.  服务器出示 server_cert.pem。
    3.  客户端使用 ca_cert.pem 验证 server_cert.pem 是由受信任的 CA 签发的。
    4.  客户端检查证书上的 subjectAltName (SAN) 是否包含它所连接的地址 (例如 localhost)。
    5.  验证通过，握手继续，建立加密连接。
### 场景 B：双向认证 (mTLS) (这才是文章想说但没说清的)
**目标：** 客户端验证服务器，并且服务器也要反过来验证客户端的身份。
**前提：** 你已经完成了步骤 CA （步骤一）和服务器证书（步骤二）。
#### 步骤 3: 生成客户端证书 (这是原文缺失的部分)
我们现在使用同一个 CA，为客户端也签发一个“身份证”。
```shell
#生成客户端私钥
openssl genrsa -out client_key.pem 4096
```
###  生成客户端 CSR
注意：CN 可以是用户名、服务名等，用于标识客户端
```shell
openssl req -new -key client_key.pem -out client.csr -subj "/C=US/ST=CA/O=gRPC-Study/CN=my-grpc-client"
```
### CA 签发客户端证书
```shell
openssl x509 -req -in client.csr -CA ca_cert.pem -CAkey ca-key.pem -CAcreateserial -out client_cert.pem -days 365
```
#### 场景 B 的配置与握手
  * **服务器需要：**
    1.  server_key.pem (自己的私钥)
    2.  server_cert.pem (自己的证书)
    3.  ca_cert.pem (用来**验证客户端**)
  * **客户端需要：**
    1.  ca_cert.pem (用来**验证服务器**)
    2.  client_key.pem (自己的私钥)
    3.  client_cert.pem (自己的证书)
  * **握手流程 (双向)：**
    1.  【客户端验证服务器】(同场景 A 的 1-4 步)
    2.  ...验证通过后...
    3.  **【服务器验证客户端】**：服务器向客户端说：“请出示你的证书。”
    4.  客户端将自己的 client_cert.pem 和 client_key.pem 发送给服务器。
    5.  服务器使用自己持有的 ca_cert.pem 来验证 client_cert.pem 是否由受信任的 CA 签发。
    6.  (可选) 服务器还可以检查客户端证书的 CN (例如 CN=my-grpc-client)，以进行更细粒度的访问控制。
    7.  双方验证均通过，建立加密连接。
### 总结

| 文件名 | 文件类型 | 作用 | 是否敏感 |
| :--- | :--- | :--- | :--- |
| **ca-key.pem** | CA 私钥 | **最高机密。** CA 的权力来源，用于**签发**所有证书（服务器和客户端）。 | **极敏感** |
| **ca_cert.pem** | CA 证书 | **公开。** 信任的根基。分发给**服务器和客户端**，用于**验证**对方证书。 | 公开 |
| ca_cert.srl | CA 证书序列 | CA 的记账本。自动生成，防止序列号冲突。 | 不敏感 |
| | | | |
| server.csr | 服务器CSR | 申请表。不重要，可删除。 | 不敏感 |
| **server_key.pem** | 服务器私钥 | **服务器的机密。** 用于证明服务器身份，解密信息。 | **极敏感** |
| **server_cert.pem**| 服务器证书 | **公开。** 服务器的“身份证”，发给客户端，由 ca_cert.pem 验证。 | 公开 |
| | | | |
| client.csr | 客户端CSR | 申请表。不重要，可删除。 | 不敏感 |
| **client_key.pem** | 客户端私钥 | **客户端的机密。** 用于证明客户端身份。 | **极敏感** |
| **client_cert.pem**| 客户端证书 | **公开。** 客户端的“身份证”，发给服务器，由 ca_cert.pem 验证。 | 公开 |
