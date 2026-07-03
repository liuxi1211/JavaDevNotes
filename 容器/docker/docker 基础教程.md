# Docker 实战完整教程：常用指令与参数速查手册

  

## 一、Docker 核心概念

  

Docker 是基于容器化技术的应用部署工具，核心包含三大要素：

  

- **镜像（Image）**：只读的应用模板，包含运行应用所需的代码、环境、依赖

- **容器（Container）**：镜像的运行实例，可读写，相互隔离

- **仓库（Registry）**：存放镜像的远程服务，如 Docker Hub

  

---

  

## 二、镜像操作指令

  

### 1. docker search — 搜索镜像

  

```bash

docker search [OPTIONS] TERM

```

  

**常用参数：**

- `--limit N`：限制返回结果数量，默认25

- `--filter stars=N`：筛选收藏数大于等于N的镜像

- `--filter is-official=true`：只显示官方镜像

  

**示例：**

```bash

docker search --filter stars=100 --filter is-official=true nginx

```

  

### 2. docker pull — 拉取镜像

  

```bash

docker pull [OPTIONS] NAME[:TAG|@DIGEST]

```

  

**常用参数：**

- `-a, --all-tags`：拉取仓库所有标签的镜像

- `--platform linux/amd64`：指定平台架构

  

**示例：**

```bash

docker pull nginx:latest

docker pull --platform linux/arm64 mysql:8.0

```

  

### 3. docker images / docker image ls — 查看本地镜像

  

```bash

docker images [OPTIONS] [REPOSITORY[:TAG]]

```

  

**常用参数：**

- `-a, --all`：显示所有镜像（含中间层镜像）

- `-q, --quiet`：只显示镜像ID

- `--format "{{.Repository}}:{{.Tag}}"`：自定义输出格式

- `--filter dangling=true`：筛选悬空镜像（无标签的镜像）

  

**示例：**

```bash

docker images -q  # 只输出所有镜像ID，常用于批量删除

```

  

### 4. docker rmi / docker image rm — 删除镜像

  

```bash

docker rmi [OPTIONS] IMAGE [IMAGE...]

```

  

**常用参数：**

- `-f, --force`：强制删除（即使有容器依赖）

- `--no-prune`：不删除未打标签的父镜像

  

**示例：**

```bash

docker rmi nginx:latest

docker rmi -f $(docker images -q)  # 批量删除所有镜像

```

  

### 5. docker build — 构建镜像

  

```bash

docker build [OPTIONS] PATH | URL | -

```

  

**常用参数：**

- `-t, --tag name:tag`：指定镜像名称和标签

- `-f, --file Dockerfile`：指定 Dockerfile 路径

- `--build-arg KEY=VALUE`：传递构建时变量

- `--no-cache`：不使用缓存，重新构建

- `--platform linux/amd64,linux/arm64`：多架构构建

- `-m, --memory 4g`：限制构建内存

  

**示例：**

```bash

docker build -t myapp:v1.0 .

docker build -f ./docker/Dockerfile.prod -t myapp:prod --build-arg ENV=production .

```

  

### 6. docker tag — 给镜像打标签

  

```bash

docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

```

  

**示例：**

```bash

docker tag myapp:v1.0 registry.example.com/myapp:v1.0

```

  

### 7. docker push — 推送镜像到仓库

  

```bash

docker push [OPTIONS] NAME[:TAG]

```

  

**常用参数：**

- `-a, --all-tags`：推送所有标签

  

**示例：**

```bash

docker push registry.example.com/myapp:v1.0

```

  

### 8. docker save / docker load — 镜像导入导出

  

```bash

# 导出镜像为 tar 文件

docker save -o output.tar IMAGE [IMAGE...]

  

# 从 tar 文件加载镜像

docker load -i input.tar

```

  

**示例：**

```bash

docker save -o nginx.tar nginx:latest

docker load -i nginx.tar

```

  

---

  

## 三、容器操作指令

  

### 1. docker run — 创建并启动容器

  

这是最核心、参数最多的命令。

  

```bash

docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

```

  

**核心常用参数：**

  

| 参数 | 说明 | 示例 |

|------|------|------|

| `-d, --detach` | 后台运行容器 | `docker run -d nginx` |

| `--name` | 指定容器名称 | `--name my-nginx` |

| `-p, --publish` | 端口映射 主机:容器 | `-p 8080:80` |

| `-P` | 随机映射所有暴露端口 | `docker run -P nginx` |

| `-v, --volume` | 挂载数据卷/目录 | `-v /data:/usr/share/nginx/html` |

| `--mount` | 更高级的挂载方式 | `type=bind,src=/data,dst=/app` |

| `-e, --env` | 设置环境变量 | `-e MYSQL_ROOT_PASSWORD=123456` |

| `--env-file` | 从文件加载环境变量 | `--env-file .env` |

| `--network` | 指定网络 | `--network my-net` |

| `--ip` | 指定容器IP（需自定义网络） | `--ip 172.20.0.10` |

| `--restart` | 重启策略 | `--restart always` |

| `--rm` | 容器停止后自动删除 | `docker run --rm alpine` |

| `-i, --interactive` | 保持标准输入打开 | 常与 `-t` 配合 |

| `-t, --tty` | 分配伪终端 | `-it` 进入交互模式 |

| `-w, --workdir` | 指定工作目录 | `-w /app` |

| `-u, --user` | 指定运行用户 | `-u root` |

| `--privileged` | 授予容器特权 | `--privileged` |

| `--memory, -m` | 限制内存 | `-m 512m` |

| `--cpus` | 限制CPU核心数 | `--cpus 1.5` |

| `--hostname` | 设置容器主机名 | `--hostname web01` |

| `--add-host` | 添加hosts映射 | `--add-host db:192.168.1.10` |

  

**经典示例：**

```bash

# 后台运行 Nginx，映射8080端口，挂载目录，自动重启

docker run -d \

  --name my-nginx \

  -p 8080:80 \

  -v /opt/nginx/html:/usr/share/nginx/html \

  -v /opt/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \

  --restart always \

  nginx:latest

  

# 交互式进入 Ubuntu 容器

docker run -it --rm ubuntu:22.04 /bin/bash

```

  

### 2. docker ps — 查看容器列表

  

```bash

docker ps [OPTIONS]

```

  

**常用参数：**

- `-a, --all`：显示所有容器（含停止的）

- `-q, --quiet`：只显示容器ID

- `-n N`：显示最近创建的N个容器

- `-l, --latest`：显示最新创建的容器

- `-s, --size`：显示容器大小

- `--format`：自定义输出格式

  

**示例：**

```bash

docker ps -aq  # 所有容器ID，常用于批量操作

```

  

### 3. docker start/stop/restart — 启停容器

  

```bash

docker start CONTAINER [CONTAINER...]

docker stop CONTAINER [CONTAINER...]

docker restart CONTAINER [CONTAINER...]

```

  

**常用参数：**

- `-t, --time N`：stop/restart 等待N秒后强制停止，默认10秒

  

**示例：**

```bash

docker stop my-nginx

docker restart -t 5 my-nginx

docker start $(docker ps -aq)  # 启动所有容器

```

  

### 4. docker rm — 删除容器

  

```bash

docker rm [OPTIONS] CONTAINER [CONTAINER...]

```

  

**常用参数：**

- `-f, --force`：强制删除运行中的容器

- `-v, --volumes`：同时删除关联的数据卷

  

**示例：**

```bash

docker rm my-nginx

docker rm -f $(docker ps -aq)  # 强制删除所有容器

docker rm -v $(docker ps -aq -f status=exited)  # 删除所有已停止容器及卷

```

  

### 5. docker exec — 在运行容器中执行命令

  

```bash

docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

```

  

**常用参数：**

- `-i`：保持输入

- `-t`：分配终端

- `-d`：后台执行

- `-u`：指定用户

- `-w`：指定工作目录

  

**示例：**

```bash

# 进入容器交互终端（最常用）

docker exec -it my-nginx /bin/bash

  

# 直接执行命令

docker exec my-nginx nginx -s reload

```

  

### 6. docker logs — 查看容器日志

  

```bash

docker logs [OPTIONS] CONTAINER

```

  

**常用参数：**

- `-f, --follow`：实时跟踪日志输出

- `--tail N`：显示最后N行

- `-t, --timestamps`：显示时间戳

- `--since 时间`：从指定时间开始的日志

  

**示例：**

```bash

docker logs -f --tail 100 my-nginx

docker logs -t --since="2024-01-01T00:00:00" my-nginx

```

  

### 7. docker inspect — 查看容器/镜像详细信息

  

```bash

docker inspect [OPTIONS] NAME|ID [NAME|ID...]

```

  

**常用参数：**

- `--format, -f`：使用 Go 模板格式化输出

- `-s, --size`：显示总文件大小（仅容器）

- `--type container|image`：指定查看类型

  

**示例：**

```bash

# 查看容器IP

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-nginx

  

# 查看挂载目录

docker inspect -f '{{json .Mounts}}' my-nginx

```

  

### 8. docker cp — 容器与主机间复制文件

  

```bash

# 主机 -> 容器

docker cp SRC_PATH CONTAINER:DEST_PATH

  

# 容器 -> 主机

docker cp CONTAINER:SRC_PATH DEST_PATH

```

  

**示例：**

```bash

docker cp ./index.html my-nginx:/usr/share/nginx/html/

docker cp my-nginx:/etc/nginx/nginx.conf ./backup/

```

  

### 9. docker top / stats — 查看容器进程与资源

  

```bash

docker top CONTAINER  # 查看容器内进程

docker stats [OPTIONS] [CONTAINER...]  # 实时查看资源占用

```

  

**stats 常用参数：**

- `-a, --all`：显示所有容器（默认只显示运行中）

- `--no-stream`：只输出一次，不持续刷新

- `--format`：自定义格式

  

---

  

## 四、数据卷（Volume）操作

  

### 1. docker volume create — 创建数据卷

  

```bash

docker volume create [OPTIONS] [VOLUME]

```

  

**常用参数：**

- `-d, --driver local`：指定驱动，默认 local

- `--opt, -o`：驱动选项，如指定挂载路径

  

**示例：**

```bash

docker volume create mysql-data

docker volume create -o type=none -o device=/opt/mysql -o o=bind mysql-data

```

  

### 2. docker volume ls — 列出数据卷

  

```bash

docker volume ls [OPTIONS]

```

  

### 3. docker volume inspect — 查看数据卷详情

  

```bash

docker volume inspect VOLUME [VOLUME...]

```

  

### 4. docker volume rm — 删除数据卷

  

```bash

docker volume rm VOLUME [VOLUME...]

```

  

### 5. docker volume prune — 清理未使用的数据卷

  

```bash

docker volume prune [OPTIONS]

```

- `-f, --force`：不提示确认

  

---

  

## 五、网络操作

  

### 1. docker network ls — 列出网络

  

```bash

docker network ls [OPTIONS]

```

  

### 2. docker network create — 创建网络

  

```bash

docker network create [OPTIONS] NETWORK

```

  

**常用参数：**

- `--driver, -d bridge`：网络驱动（bridge/host/none/overlay）

- `--subnet`：子网网段，如 `172.20.0.0/16`

- `--gateway`：网关地址

- `--ip-range`：分配IP范围

  

**示例：**

```bash

docker network create --driver bridge --subnet 172.20.0.0/16 my-net

```

  

### 3. docker network connect/disconnect — 连接/断开网络

  

```bash

docker network connect NETWORK CONTAINER

docker network disconnect NETWORK CONTAINER

```

  

**示例：**

```bash

docker network connect my-net my-nginx

```

  

### 4. docker network inspect — 查看网络详情

  

```bash

docker network inspect NETWORK [NETWORK...]

```

  

### 5. docker network rm / prune — 删除/清理网络

  

```bash

docker network rm NETWORK

docker network prune  # 删除所有未使用的网络

```

  

---

  

## 六、Dockerfile 常用指令详解

  

### 基础指令

  

| 指令 | 作用 | 示例 |

|------|------|------|

| `FROM` | 指定基础镜像 | `FROM node:20-alpine` |

| `LABEL` | 镜像元数据 | `LABEL maintainer="admin@example.com"` |

| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |

| `ENV` | 设置环境变量 | `ENV NODE_ENV=production` |

| `ARG` | 构建时变量 | `ARG VERSION=1.0` |

| `RUN` | 构建时执行命令 | `RUN npm install` |

| `COPY` | 复制文件到镜像 | `COPY . .` |

| `ADD` | 复制（支持URL/自动解压） | `ADD app.tar.gz /app/` |

| `EXPOSE` | 声明暴露端口 | `EXPOSE 80 443` |

| `CMD` | 容器启动默认命令 | `CMD ["npm", "start"]` |

| `ENTRYPOINT` | 容器入口程序 | `ENTRYPOINT ["docker-entrypoint.sh"]` |

| `VOLUME` | 声明挂载点 | `VOLUME ["/data"]` |

| `USER` | 指定运行用户 | `USER node` |

| `HEALTHCHECK` | 健康检查 | `HEALTHCHECK CMD curl -f http://localhost/` |

  

### 关键区别说明

  

1. **COPY vs ADD**

   - COPY 只做简单文件复制，推荐优先使用

   - ADD 额外支持 URL 下载和 tar 自动解压，但行为不直观

  

2. **CMD vs ENTRYPOINT**

   - CMD：容器启动的默认命令和参数，可被 `docker run` 覆盖

   - ENTRYPOINT：容器的固定入口程序，参数可追加

   - 最佳实践：ENTRYPOINT 定程序，CMD 给默认参数

  

3. **RUN vs CMD**

   - RUN：镜像构建阶段执行，产生新的镜像层

   - CMD：容器启动时执行，每个容器只执行一次

  

### Dockerfile 最佳实践示例

  

```dockerfile

# 多阶段构建：构建阶段

FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=production

  

# 运行阶段

FROM node:20-alpine

WORKDIR /app

ENV NODE_ENV=production

USER node

COPY --from=builder /app/node_modules ./node_modules

COPY . .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]

```

  

---

  

## 七、Docker Compose 常用命令

  

Docker Compose 用于编排多容器应用，配置文件为 `docker-compose.yml`。

  

### 核心命令

  

```bash

# 启动所有服务（后台）

docker compose up -d

  

# 启动并构建镜像

docker compose up -d --build

  

# 停止并删除容器、网络

docker compose down

  

# 停止并删除容器、网络、镜像、数据卷

docker compose down -v --rmi all

  

# 查看服务状态

docker compose ps

  

# 查看日志

docker compose logs -f --tail=100

  

# 重启服务

docker compose restart [SERVICE]

  

# 在服务中执行命令

docker compose exec SERVICE COMMAND

  

# 拉取服务依赖的镜像

docker compose pull

  

# 查看配置

docker compose config

```

  

### docker-compose.yml 核心配置项

  

```yaml

version: '3.8'

  

services:

  web:

    image: nginx:latest        # 镜像

    build: ./nginx             # 从 Dockerfile 构建

    container_name: web-nginx  # 容器名

    ports:                      # 端口映射

      - "80:80"

    volumes:                    # 数据卷

      - ./html:/usr/share/nginx/html

      - nginx-log:/var/log/nginx

    environment:                # 环境变量

      - TZ=Asia/Shanghai

    env_file:                   # 环境变量文件

      - .env

    networks:                   # 所属网络

      - frontend

    depends_on:                 # 依赖顺序

      - app

    restart: always             # 重启策略

    healthcheck:                # 健康检查

      test: ["CMD", "curl", "-f", "http://localhost"]

      interval: 30s

      timeout: 10s

      retries: 3

  

volumes:

  nginx-log:

  

networks:

  frontend:

    driver: bridge

```

  

---

  

## 八、系统运维与清理命令

  

### 1. docker system df — 查看磁盘占用

  

```bash

docker system df  # 概览

docker system df -v  # 详细信息

```

  

### 2. docker system prune — 一键清理无用资源

  

```bash

docker system prune [OPTIONS]

```

  

**常用参数：**

- `-a, --all`：删除所有未使用镜像（不只是悬空镜像）

- `-f, --force`：不提示确认

- `--volumes`：同时清理未使用的数据卷

  

**示例：**

```bash

docker system prune -af --volumes  # 深度清理，释放全部空闲资源

```

  

### 3. docker info — 查看 Docker 系统信息

  

```bash

docker info

```

  

### 4. docker version — 查看版本

  

```bash

docker version

```

  

---

  

## 九、高频速查表

  

### 镜像操作

  

```bash

docker search 镜像名          # 搜索

docker pull 镜像名:标签       # 拉取

docker images                 # 查看

docker rmi 镜像ID             # 删除

docker build -t 名:标签 .     # 构建

docker save -o 文件.tar 镜像  # 导出

docker load -i 文件.tar       # 导入

```

  

### 容器操作

  

```bash

docker run -d --name 名 -p 宿:容 -v 宿:容 镜像  # 启动

docker ps -a                  # 查看所有

docker start/stop/restart 名  # 启停

docker rm -f 名               # 删除

docker exec -it 名 /bin/bash  # 进入

docker logs -f --tail 100 名  # 日志

docker inspect 名             # 详情

docker cp 宿路径 名:容路径     # 复制文件

```

  

### 清理三件套

  

```bash

docker system prune -f        # 清理悬空资源

docker volume prune -f        # 清理未用卷

docker image prune -af        # 清理未用镜像

```

  

---

  

## 十、最佳实践总结

  

1. **镜像构建**

   - 优先使用 Alpine 基础镜像减小体积

   - 采用多阶段构建，只保留运行时必要文件

   - 合并 RUN 指令减少镜像层数

   - 将变化少的指令放在上层，充分利用构建缓存

  

2. **容器运行**

   - 一个容器只跑一个主进程

   - 使用 `--restart always` 保证服务高可用

   - 数据必须挂载到外部 volume 或宿主机目录

   - 不要在容器中存储持久化数据

  

3. **安全建议**

   - 容器内不以 root 用户运行，使用 USER 指令切换

   - 只映射必要端口，不使用 `--privileged` 特权模式

   - 生产环境使用固定版本镜像，不使用 latest

   - 定期更新基础镜像修复漏洞

  

---

  

需要我针对某个具体场景（比如部署 WordPress、Java 应用、MySQL 主从等）给出完整的实战示例吗？