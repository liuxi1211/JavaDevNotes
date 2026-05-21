
---

## 案例一：Nginx 容器反向代理局域网 MySQL + Redis

### 前提条件

1. 局域网内已部署MySQL（IP：`192.168.3.100`，端口：`3306`）
    
2. 局域网内已部署Redis（IP：`192.168.3.101`，端口：`6379`）
    
3. Docker主机可以正常访问上述两个服务
    
4. 关闭Docker主机防火墙或开放对应端口
    

### 核心说明

MySQL和Redis使用**TCP协议**，不能用Nginx的HTTP模块代理，必须使用**stream模块**。官方Nginx镜像已默认包含stream模块。

---

### 步骤1：拉取Nginx镜像

```Bash
docker pull nginx:1.25-alpine
```

### 步骤2：创建Nginx配置文件

在主机上创建配置目录和文件：

```Bash
mkdir -p /opt/nginx/conf
vim /opt/nginx/conf/nginx.conf
```

写入以下配置内容：

```Nginx
# 全局配置
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# 事件模块
events {
    worker_connections 1024;
}

# TCP代理模块（核心）
stream {
    # 定义上游MySQL服务器
    upstream mysql_backend {
        server 192.168.3.100:3306;
        # 多节点可配置负载均衡
        # server 192.168.3.102:3306 backup;
    }

    # 定义上游Redis服务器
    upstream redis_backend {
        server 192.168.3.101:6379;
    }

    # MySQL代理服务
    server {
        listen 3306; # 容器监听3306端口
        proxy_pass mysql_backend;
        proxy_timeout 300s; # 超时时间
        proxy_connect_timeout 10s; # 连接超时
    }

    # Redis代理服务
    server {
        listen 6379; # 容器监听6379端口
        proxy_pass redis_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
    }
}

# HTTP模块（可选，用于静态资源或HTTP代理）
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
}
```

### 步骤3：启动Nginx容器

使用**host网络模式**（推荐，性能最好，无需端口映射，直接使用主机网络）：

```Bash
docker run -d \
  --name nginx-proxy \
  --network host \
  -v /opt/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
  -v /opt/nginx/logs:/var/log/nginx \
  --restart always \
  nginx:1.25-alpine
```

**参数说明**：

- `--network host`：容器直接使用主机网络，性能无损耗
    
- `-v`：挂载配置文件和日志目录到主机，方便修改和查看
    
- `--restart always`：容器异常退出或主机重启后自动启动
    

### 步骤4：验证代理是否成功

#### 验证MySQL代理

在任意局域网机器上使用MySQL客户端连接Docker主机的3306端口：

```Bash
mysql -h 192.168.3.200 -u root -p
# 192.168.3.200是Docker主机的IP地址
```

#### 验证Redis代理

在任意局域网机器上使用Redis客户端连接Docker主机的6379端口：

```Bash
redis-cli -h 192.168.3.200 -p 6379
```

### 步骤5：查看日志和排错

```Bash
# 查看Nginx运行日志
docker logs -f nginx-proxy

# 进入容器检查配置
docker exec -it nginx-proxy /bin/sh
nginx -t # 检查配置文件语法
```

---

## 案例二：SpringBoot 项目从构建到生产部署全流程

### 前提条件

1. 本地已安装JDK 17+和Maven 3.8+
    
2. 有一个简单的SpringBoot项目（本文使用SpringBoot 3.2.x）
    
3. 项目已配置连接局域网的MySQL和Redis（使用案例一中的Nginx代理地址）
    

---

### 步骤1：准备SpringBoot项目

#### 1.1 项目结构

```Plaintext
my-springboot-app/
├── src/
│   └── main/
│       ├── java/
│       │   └── com/
│       │       └── example/
│       │           └── MyApp.java
│       └── resources/
│           └── application.yml
├── pom.xml
└── Dockerfile
```

#### 1.2 配置application.yml

```YAML
server:
  port: 8080

spring:
  # 连接Nginx代理的MySQL
  datasource:
    url: jdbc:mysql://192.168.3.200:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

  # 连接Nginx代理的Redis
  data:
    redis:
      host: 192.168.3.200
      port: 6379
      password: ""
      database: 0
```

#### 1.3 编写测试接口

```Java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }

    @GetMapping("/")
    public String hello() {
        return "Hello SpringBoot + Docker!";
    }

    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

### 步骤2：编写多阶段构建Dockerfile

在项目根目录创建`Dockerfile`：

```Dockerfile
# 第一阶段：构建应用
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /app

# 复制pom.xml并下载依赖（利用Docker缓存）
COPY pom.xml .
RUN mvn dependency:go-offline

# 复制源代码并构建
COPY src ./src
RUN mvn clean package -DskipTests

# 第二阶段：运行应用（使用精简JRE镜像）
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# 安装必要工具（可选）
RUN apk add --no-cache tzdata

# 设置时区
ENV TZ=Asia/Shanghai

# 从构建阶段复制jar包
COPY --from=builder /app/target/*.jar app.jar

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**多阶段构建优势**：最终镜像只包含JRE和应用jar包，体积可减小80%以上。

### 步骤3：构建Docker镜像

在项目根目录执行：

```Bash
docker build -t my-springboot-app:v1 .
```

构建完成后查看镜像：

```Bash
docker images | grep my-springboot-app
```

### 步骤4：运行SpringBoot容器

```Bash
docker run -d \
  --name my-app \
  -p 8080:8080 \
  --restart always \
  my-springboot-app:v1
```

### 步骤5：验证应用是否正常运行

#### 查看容器状态

```Bash
docker ps | grep my-app
```

#### 查看应用日志

```Bash
docker logs -f my-app
```

#### 测试接口访问

```Bash
curl http://localhost:8080
# 输出：Hello SpringBoot + Docker!

curl http://localhost:8080/health
# 输出：OK
```

### 步骤6：使用Docker Compose简化部署

在项目根目录创建`docker-compose.yml`：

```YAML
version: '3.8'

services:
  app:
    build: .
    image: my-springboot-app:v1
    container_name: my-app
    ports:
      - "8080:8080"
    environment:
      # 覆盖application.yml中的配置
      - SPRING_DATASOURCE_URL=jdbc:mysql://192.168.3.200:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=123456
      - SPRING_DATA_REDIS_HOST=192.168.3.200
    restart: always
    volumes:
      - ./logs:/app/logs
```

使用Compose启动应用：

```Bash
# 构建镜像并后台启动
docker compose up -d --build

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f app
```

### 步骤7：集成到Nginx反向代理（可选）

修改案例一中的Nginx配置，添加HTTP反向代理：

```Nginx
http {
    # ... 其他配置 ...

    server {
        listen 80;
        server_name app.example.com;

        location / {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

重启Nginx容器：

```Bash
docker restart nginx-proxy
```

现在可以通过`http://app.example.com`访问SpringBoot应用。

---

### 常见问题与解决方法

1. **容器无法访问局域网服务**：
    
    1. 检查Docker主机防火墙是否开放端口
        
    2. 使用`--network host`模式替代bridge模式
        
    3. 验证Docker主机能否ping通局域网服务
        
2. **SpringBoot连接数据库失败**：
    
    1. 检查数据库地址、端口、用户名密码是否正确
        
    2. 确认数据库允许远程访问
        
    3. 查看应用日志获取详细错误信息
        
3. **Nginx TCP代理失败**：
    
    1. 确保配置在`stream`块中，而非`http`块
        
    2. 检查Nginx配置语法：`nginx -t`
        
    3. 查看Nginx错误日志：`/opt/nginx/logs/error.log`
        
