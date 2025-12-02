---
title: nginx基础操作
date: 2025-12-02 11:31:17
tags: nginx
---

### 一、Root vs Alias

这是 Nginx 最大的坑，90% 的 404 错误都源于此。

#### 1. Root (追加模式)

**口诀**：`root` 只是指定了根目录，Nginx 会把请求的 URL **完整地拼接到** root 指定的路径后面。

* **配置**：
  ```nginx
  location /info/ {
      root /usr/local/html;
  }
  ```
* **请求**：`GET /info/a.js`
* **实际寻找文件**：`/usr/local/html` + `/info/a.js` =  **`/usr/local/html/info/a.js`**
* **场景**：通常用于根目录 `/` 或者你的文件夹结构和 URL 完全一致的情况。

#### 2. Alias (替换模式)

**口诀**：`alias` 是替身，Nginx 会**丢掉** location 匹配到的那部分 URL，只把剩下的拼接到 alias 路径后面。

* **配置**：
  ```nginx
  location /info/ {
      # 注意：使用 alias 时，目录后面必须加 /
      alias /usr/local/html/;
  }
  ```
* **请求**：`GET /info/a.js`
* **实际寻找文件**：(去掉 /info/) -\> 剩下 `a.js` -\> 拼接到 alias -\> **`/usr/local/html/a.js`**
* **场景**：前端项目部署在子路径（比如 `VITE_BASE=/info`），但硬盘上的文件夹名不叫 `info`（比如叫 `dist`）时，**必须用 alias**。

#### 3\. 一图对比

| 指令 | URL 请求 | 配置路径 | 最终文件路径 | 你的情况适合谁？ |
| :--- | :--- | :--- | :--- | :--- |
| **root** | `/info/img.png` | `/data/dist` | `/data/dist/info/img.png` | ❌ 硬盘 dist 里没有 info 文件夹 |
| **alias** | `/info/img.png` | `/data/dist/` | `/data/dist/img.png` | ✅ 刚好对应 dist 根目录 |

-----

### 二、 常用操作命令 (Mac/Linux)

在终端操作 Nginx 进程的命令。

| 命令 | 解释 | 什么时候用 |
| :--- | :--- | :--- |
| **`nginx -t`** | **检查配置文件语法** | **最常用！** 每次改完配置必点，告诉你哪里写错了。 |
| **`nginx -s reload`** | **平滑重启** | 改完配置且 `nginx -t` 通过后，让配置生效。 |
| `nginx -s stop` | 停止服务 | 彻底关掉 Nginx。 |
| `brew services start nginx` | (Mac专用) 启动服务 | 开机自启或后台运行。 |
| `brew services restart nginx` | (Mac专用) 重启服务 | 整个进程杀掉重启（比 reload 暴力）。 |

-----

### 三、 常用配置指令 (nginx.conf)

#### 1\. `location` 匹配规则

Nginx 怎么决定用哪个块来处理请求？

* `location = /abc`：**精确匹配**。只有完全等于 `/abc` 才匹配。优先级最高。
* `location /abc`：**前缀匹配**。只要是以 `/abc` 开头的都算（如 `/abc/def`）。
* `location /`：**兜底匹配**。如果上面都没匹配上，就进这里。

#### 2\. `proxy_pass` 反向代理

解决跨域，转发接口到后端。这里也有一个**类似 root/alias 的坑**（斜杠陷阱）。

* **写法 A（不带斜杠）：**

  ```nginx
  location /api/ {
      proxy_pass http://localhost:8081; 
  }
  ```

    * 请求：`/api/user`
    * 转发给后端：`http://localhost:8081/api/user` (**保留**了 /api 前缀)

* **写法 B（带斜杠）：**

  ```nginx
  location /api/ {
      proxy_pass http://localhost:8081/; 
  }
  ```

    * 请求：`/api/user`
    * 转发给后端：`http://localhost:8081/user` (**去掉**了 /api 前缀)
    * **注意**：如果你后端接口路径本身不带 `/api`，通常用这种写法。

#### 3\. `try_files` (SPA 必配)

Vue/React 项目刷新 404 的救命稻草。

```nginx
try_files $uri $uri/ /index.html;
```

**翻译**：

1.  **`$uri`**：先找找有没有这个文件？（比如 `a.js`，有就返回）
2.  **`$uri/`**：再找找有没有这个目录？（比如 `assets/`，有就找下面的 index）
3.  **`/index.html`**：都找不到？那肯定是前端路由（Route），直接把首页 `index.html` 发给浏览器，让前端 JS 自己去解析路由。

-----

### 四、 总结： `dist` 部署模板

基于我的的踩坑经历，这是最标准的 **Vue/Vite 项目部署到子路径** 的 Nginx 模板：

```nginx
server {
    listen       8080;
    server_name  localhost;

    # 1. 精确匹配：处理不带斜杠的访问，自动加斜杠
    location = /info {
        return 301 /info/;
    }

    # 2. 通用匹配：处理资源文件
    location /info/ {
        # 使用 alias 指向 dist 物理路径 (注意结尾斜杠)
        alias  /Users/root1/Desktop/dist/;
        
        # 首页文件
        index  index.html;
        
        # 兜底策略：注意最后回退的路径必须带 /info/ 前缀
        try_files $uri $uri/ /info/index.html;
    }
}

 server {
     listen 8081;
     server_name debug;
     # 访问路径以/admin-api开头的转发到http://127.0.0.1:8081/admin-api
     location /admin-api {
               proxy_pass http://127.0.0.1:8081;
               proxy_set_header Host $host:$server_port;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_set_header Cookie $http_cookie;
               proxy_pass_header Set-Cookie;
               proxy_redirect off;
              }
        }
```

-----

### 实际生产环境搭建

在实际生产环境中，**绝对不会**把几十个域名的配置全塞进一个 `nginx.conf` 里。那样做不仅难以维护，而且一旦改错一个标点符号，所有网站都会挂掉。

解决办法就是：**模块化拆分**，使用 **`include`** 指令。

#### 第一步：在 `nginx.conf` 中引入外部文件夹

打开主配置文件（`/opt/homebrew/etc/nginx/nginx.conf`）。

通常 Homebrew 安装的 Nginx 默认已经在 `http { ... }` 块的**最后一行**写好了引入配置。如果没有，请手动加上：

```nginx
http {
    # ... 前面的一堆 gzip, log_format 等通用配置保持不变 ...

    # 🚀 核心指令：引入 servers 文件夹下的所有 .conf 结尾的文件
    # 注意：路径可以是相对路径（相对于 nginx.conf 所在目录），也可以是绝对路径
    include servers/*.conf; 
}
```

*注意：Homebrew 版 Nginx 默认可能已经创建了一个 `servers` 文件夹，和 `nginx.conf` 同级。*


#### 第二步：创建独立的配置文件

假设你有两个项目：一个是你的**信息管理系统 (Info)**，一个是**电商后台 (Shop)**。

1.  **进入 servers 目录**：

    ```bash
    cd /opt/homebrew/etc/nginx/servers
    ```

2.  **创建第一个文件 `info.conf`**：
    *(名字随便起，最好以域名或项目名命名，必须以 .conf 结尾)*

    ```nginx
    # info.conf 文件内容
    server {
        listen       8080;
        server_name  localhost;
        
        location /info/ {
            alias  /Users/root1/Desktop/dist-info/;
            index  index.html;
            try_files $uri $uri/ /info/index.html;
        }
    }
    ```

3.  **创建第二个文件 `shop.conf`**：

    ```nginx
    # shop.conf 文件内容
    server {
        listen       8081; # 比如这个项目跑在 8081
        server_name  shop.local;
        
        location / {
            root   /Users/root1/Desktop/dist-shop/;
            index  index.html;
        }
    }
    ```


#### 第三步：清理主配置文件

现在，你可以把 `nginx.conf` 里面原有的那些庞大的 `server { ... }` 代码块**全部删掉**（或者注释掉），只保留全局配置和 `include` 指令。

**瘦身后的 `nginx.conf` 会变得非常清爽：**

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout  65;

    # 只剩这一行干活的，其他具体配置全在文件夹里
    include servers/*.conf; 
}
```

#### 第四步：测试与重载

当你新增了一个 `blog.conf` 或者修改了 `info.conf` 后，只需要像往常一样操作：

1.  **检查所有子文件语法**：
    `nginx -t`
    *(它会自动扫描 include 进来的所有文件并检查错误)*

2.  **重载生效**：
    `nginx -s reload`

-----

###  进阶技巧：企业级管理方案
所谓的\*\*“企业级管理方案”\*\*，核心目标是：**在不删除配置文件的情况下，能够快速地开启或关闭某个网站，并且最大程度复用公共配置。**

业界（尤其是 Linux 发行版如 Ubuntu/Debian）最标准的做法是采用 **`sites-available`（库存目录）** 和 **`sites-enabled`（生效目录）** 配合 **软链接（Symbolic Link）** 的管理模式。

以下是在Linux 上搭建这套体系：


#### 一、 核心架构图解

把 Nginx 想象成一个**商场**：
1.  **`sites-available` (仓库)**：里面存放着所有商铺的装修图纸（配置文件）。不管你写了多少个，只要放在这里，Nginx **都不会看**，不会生效。
2.  **`sites-enabled` (展示厅)**：这里存放着**指向**仓库图纸的“快捷方式”（软链接）。Nginx **只读取**这里的配置。
3.  **开关逻辑**：
    * **上线**：创建一个软链接，把图纸从仓库映射到展示厅。
    * **下线**：把展示厅里的软链接删掉（仓库里的原文件还在，随时可以恢复）。



#### 二、 搭建步骤 (以 Mac Homebrew 为例)

假设你的 Nginx 根目录在 `/opt/homebrew/etc/nginx`（如果是 Linux 通常在 `/etc/nginx`）。

##### 1\. 创建目录结构

打开终端，执行：

```bash
cd /opt/homebrew/etc/nginx

# 1. 创建两个文件夹
mkdir sites-available
mkdir sites-enabled

# 2. 创建一个存放公共片段的文件夹 (稍后讲)
mkdir snippets
```

#### 2\. 修改主配置 `nginx.conf`

打开 `nginx.conf`，将之前的 `include servers/*.conf;` 删掉，换成指向 `sites-enabled`：

```nginx
http {
    # ... 其他配置 ...

    # 🚀 核心：只引入 sites-enabled 目录下的文件
    include sites-enabled/*.conf;
}
```


#### 三、 工作流演示

#### 1\. 新建项目配置（入库）

比如你要配置一个 `cms.conf`，文件必须建在 **available** 目录：

```bash
# 或者是 code /opt/homebrew/etc/nginx/sites-available/cms.conf
vim sites-available/cms.conf 
```

内容如下：

```nginx
server {
    listen 8080;
    server_name cms.local;
    root /var/www/cms;
}
```

此时你运行 `nginx -t`，或者重启 Nginx，**这个网站是不会生效的**，因为它还在仓库里。

#### 2\. 启用项目（上线 / 创建软链）

这是最关键的一步。我们需要在 `enabled` 目录创建一个指向 `available` 的软链接。

**⚠️ 注意：创建软链时，必须使用【绝对路径】，否则 Nginx 会报错！**

```bash
# 语法：ln -s <源文件绝对路径> <目标链接路径>
ln -s /opt/homebrew/etc/nginx/sites-available/cms.conf /opt/homebrew/etc/nginx/sites-enabled/
```

此时，`sites-enabled` 里就有了一个 `cms.conf` 的快捷方式。
执行 `nginx -s reload`，网站上线。

#### 3\. 停用项目（下线）

比如这个项目要维护，暂时关闭，不需要删文件，只需要删链接：

```bash
rm sites-enabled/cms.conf
```

执行 `nginx -s reload`，网站下线。原配置文件还在 `sites-available` 里躺着，安全无忧。

-----

#### 四、 进阶：复用配置 (Snippets)

企业级管理的另一个痛点是：**重复代码太多**。
比如每个 HTTPS 的网站都要配一遍 SSL证书路径，或者每个 API 都要配一遍跨域（CORS）。

我们可以用 **`snippets`（片段）** 来解决。

#### 1\. 抽取公共配置

在 `snippets` 目录下新建一个 `cors.conf`：

```nginx
# /opt/homebrew/etc/nginx/snippets/cors.conf

# 允许跨域的通用配置
add_header 'Access-Control-Allow-Origin' '*';
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
add_header 'Access-Control-Allow-Headers' 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

if ($request_method = 'OPTIONS') {
    return 204;
}
```

#### 2\. 在各个项目中引用

现在回到你的 `sites-available/cms.conf`，只需要一行代码：

```nginx
server {
    listen 8080;
    server_name cms.local;
    
    location /api/ {
        # 🚀 一行引入，所有跨域配置生效
        include snippets/cors.conf;
        
        proxy_pass http://localhost:9000;
    }
}
```

-----

### 五、 总结：一套标准的企业级操作规范

| 动作 | 你的操作 |
| :--- | :--- |
| **新增网站** | 在 `sites-available` 新建 `.conf` 文件 |
| **测试配置** | 无论什么时候，手滑之前先 `nginx -t` |
| **上线网站** | `ln -s` 做软链到 `sites-enabled`，然后 `reload` |
| **下线网站** | `rm` 掉 `sites-enabled` 里的软链，然后 `reload` |
| **修改通用配置** | 修改 `snippets` 里的文件，所有引用的网站同时生效 |

