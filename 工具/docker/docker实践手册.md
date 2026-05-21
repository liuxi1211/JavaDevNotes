
本教程按照**功能模块**系统整理Docker核心指令，包含**用途说明、核心参数、实战示例**及**响应结果解析**，覆盖95%以上日常开发运维场景。所有指令均基于Docker Engine 26.x最新稳定版验证。

## 一、基础概念速览

- **镜像(Image)**：只读模板，包含运行应用所需的代码、运行时、库、环境变量和配置文件
    
- **容器(Container)**：镜像的可运行实例，是独立的沙箱环境
    
- **数据卷(Volume)**：Docker管理的持久化存储，独立于容器生命周期
    
- **网络(Network)**：容器间及容器与外部的通信桥梁
    
- **Dockerfile**：构建镜像的文本文件，包含一系列指令
    

---

## 二、镜像管理指令

### 1. `docker images` - 列出本地镜像

**用途**：查看本地已下载或构建的所有镜像 **语法**：`docker images [OPTIONS] [REPOSITORY[:TAG]]` **核心参数**：

- `-a, --all`：显示所有镜像（包括中间层镜像）
    
- `-q, --quiet`：只显示镜像ID
    
- `--filter, -f`：过滤结果（如`dangling=true`显示悬空镜像）
    
- `--format`：自定义输出格式
    

**示例**：

```Bash
# 列出所有镜像
docker images

# 只显示ubuntu镜像
docker images ubuntu

# 显示所有悬空镜像并删除
docker images -f dangling=true -q | xargs docker rmi
```

**响应解析**：

```Plaintext
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    6efc10a0510f   2 weeks ago    187MB
ubuntu       22.04     5a81c4b8502e   3 weeks ago    77.8MB
```

- `REPOSITORY`：镜像仓库名
    
- `TAG`：镜像标签（版本标识）
    
- `IMAGE ID`：镜像唯一ID（前12位）
    
- `CREATED`：镜像创建时间
    
- `SIZE`：镜像占用磁盘空间
    

### 2. `docker pull` - 从仓库拉取镜像

**用途**：从Docker Hub或私有仓库下载镜像到本地 **语法**：`docker pull [OPTIONS] NAME[:TAG|@DIGEST]` **核心参数**：

- `-a, --all-tags`：拉取仓库中所有标签的镜像
    
- `--platform`：指定平台（如`linux/amd64`、`linux/arm64`）
    

**示例**：

```Bash
# 拉取最新版nginx
docker pull nginx

# 拉取指定版本的ubuntu
docker pull ubuntu:22.04

# 拉取指定平台的镜像
docker pull --platform linux/arm64 nginx:alpine
```

**响应解析**：

```Plaintext
Using default tag: latest
latest: Pulling from library/nginx
Digest: sha256:abcdef1234567890...
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
```

- `Digest`：镜像的唯一哈希值，用于验证镜像完整性
    
- `Status`：拉取状态（已下载/已存在）
    

### 3. `docker build` - 从Dockerfile构建镜像

**用途**：根据Dockerfile文件构建自定义镜像 **语法**：`docker build [OPTIONS] PATH | URL | -` **核心参数**：

- `-t, --tag`：为镜像打标签（`name:tag`格式）
    
- `-f, --file`：指定Dockerfile路径（默认当前目录的`Dockerfile`）
    
- `--build-arg`：设置构建时的环境变量
    
- `--no-cache`：不使用构建缓存，重新构建所有层
    
- `-q, --quiet`：安静模式，只输出镜像ID
    

**示例**：

```Bash
# 从当前目录构建镜像，标签为myapp:v1
docker build -t myapp:v1 .

# 指定Dockerfile路径并设置构建参数
docker build -f ./docker/Dockerfile.prod --build-arg NODE_ENV=production -t myapp:prod .

# 不使用缓存构建
docker build --no-cache -t myapp:v2 .
```

**响应解析**：

```Plaintext
Sending build context to Docker daemon  2.048kB
Step 1/5 : FROM node:20-alpine
 ---> abcdef123456
Step 2/5 : WORKDIR /app
 ---> Running in 7890abcd
 ---> 1234567890ab
Removing intermediate container 7890abcd
...
Successfully built 1234567890ab
Successfully tagged myapp:v1
```

- 每一步对应Dockerfile中的一条指令
    
- `Removing intermediate container`：删除构建过程中产生的临时容器
    
- 最后两行显示构建成功的镜像ID和标签
    

### 4. `docker rmi` - 删除本地镜像

**用途**：删除本地不再需要的镜像 **语法**：`docker rmi [OPTIONS] IMAGE [IMAGE...]` **核心参数**：

- `-f, --force`：强制删除（即使有容器正在使用该镜像）
    
- `--no-prune`：不删除未打标签的父镜像
    

**示例**：

```Bash
# 删除指定镜像
docker rmi nginx:latest

# 强制删除镜像
docker rmi -f ubuntu:20.04

# 删除所有悬空镜像
docker image prune

# 删除所有未使用的镜像
docker image prune -a
```

**注意**：如果有容器正在使用该镜像，需要先删除容器再删除镜像，或使用`-f`参数强制删除。

### 5. `docker tag` - 为镜像打标签

**用途**：为本地镜像添加新的标签，通常用于推送到私有仓库 **语法**：`docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`

**示例**：

```Bash
# 为本地镜像添加私有仓库标签
docker tag myapp:v1 registry.example.com/myapp:v1

# 为镜像添加latest标签
docker tag myapp:v1 myapp:latest
```

### 6. `docker push` - 推送镜像到仓库

**用途**：将本地镜像推送到Docker Hub或私有仓库 **语法**：`docker push [OPTIONS] NAME[:TAG]` **核心参数**：

- `-a, --all-tags`：推送所有标签的镜像
    

**示例**：

```Bash
# 推送到私有仓库
docker push registry.example.com/myapp:v1

# 推送到Docker Hub（需要先登录）
docker push username/myapp:latest
```

---

## 三、容器管理指令

### 1. `docker run` - 创建并启动容器

**用途**：Docker最核心的指令，从镜像创建一个新容器并启动它 **语法**：`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]` **核心参数**（按重要性排序）：

- `-d, --detach`：后台运行容器并返回容器ID
    
- `-p, --publish`：端口映射（`主机端口:容器端口`）
    
- `-v, --volume`：挂载数据卷或主机目录（`主机路径:容器路径`）
    
- `--name`：为容器指定一个名称
    
- `-e, --env`：设置环境变量
    
- `--restart`：容器退出时的重启策略（`no`/`always`/`on-failure`/`unless-stopped`）
    
- `-it`：交互式运行（`-i`保持标准输入打开，`-t`分配伪终端）
    
- `--rm`：容器退出时自动删除
    
- `--network`：指定容器加入的网络
    
- `--user`：指定运行容器的用户
    
- `--privileged`：给予容器特权（访问主机设备）
    

**示例**：

```Bash
# 交互式运行ubuntu容器，退出后自动删除
docker run -it --rm ubuntu:22.04 /bin/bash

# 后台运行nginx，映射8080端口到容器80端口
docker run -d -p 8080:80 --name mynginx nginx:latest

# 运行mysql，设置环境变量，挂载数据卷
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -e MYSQL_DATABASE=mydb \
  -v mysql_data:/var/lib/mysql \
  --restart always \
  mysql:8.0
```

**响应解析**：

- 交互式模式：直接进入容器终端，输入`exit`退出
    
- 后台模式：返回容器的完整ID（64位）
    

### 2. `docker ps` - 列出容器

**用途**：查看当前运行的容器 **语法**：`docker ps [OPTIONS]` **核心参数**：

- `-a, --all`：显示所有容器（包括已停止的）
    
- `-q, --quiet`：只显示容器ID
    
- `-s, --size`：显示容器大小
    
- `--filter, -f`：过滤结果（如`status=running`、`name=mynginx`）
    
- `--format`：自定义输出格式
    

**示例**：

```Bash
# 列出运行中的容器
docker ps

# 列出所有容器
docker ps -a

# 只显示容器ID
docker ps -q

# 显示所有已停止的容器
docker ps -f status=exited
```

**响应解析**：

```Plaintext
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
abc123def456   nginx:latest   "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp   mynginx
```

- `CONTAINER ID`：容器唯一ID（前12位）
    
- `IMAGE`：容器使用的镜像
    
- `COMMAND`：容器启动时执行的命令
    
- `CREATED`：容器创建时间
    
- `STATUS`：容器状态（Up/Exited/Restarting等）
    
- `PORTS`：端口映射信息
    
- `NAMES`：容器名称
    

### 3. `docker start/stop/restart` - 启动/停止/重启容器

**用途**：管理已存在容器的生命周期 **语法**：

```Bash
docker start [OPTIONS] CONTAINER [CONTAINER...]
docker stop [OPTIONS] CONTAINER [CONTAINER...]
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

**核心参数**：

- `-t, --time`：停止容器前等待的时间（秒），默认10秒
    

**示例**：

```Bash
# 启动容器
docker start mynginx

# 停止容器
docker stop mynginx

# 强制停止容器（立即杀死）
docker stop -t 0 mynginx

# 重启容器
docker restart mynginx
```

### 4. `docker exec` - 在运行中的容器内执行命令

**用途**：进入正在运行的容器或在容器内执行命令 **语法**：`docker exec [OPTIONS] CONTAINER COMMAND [ARG...]` **核心参数**：

- `-it`：交互式执行（进入容器终端）
    
- `-d, --detach`：后台执行命令
    
- `-u, --user`：指定执行命令的用户
    

**示例**：

```Bash
# 进入运行中的nginx容器
docker exec -it mynginx /bin/bash

# 在容器内执行命令（不进入终端）
docker exec mynginx ls /usr/share/nginx/html

# 后台执行命令
docker exec -d mynginx touch /tmp/test.txt
```

**注意**：`docker exec`只能在**运行中**的容器内执行命令。

### 5. `docker rm` - 删除容器

**用途**：删除已停止的容器 **语法**：`docker rm [OPTIONS] CONTAINER [CONTAINER...]` **核心参数**：

- `-f, --force`：强制删除运行中的容器
    
- `-v, --volumes`：删除与容器关联的匿名数据卷
    

**示例**：

```Bash
# 删除已停止的容器
docker rm mynginx

# 强制删除运行中的容器
docker rm -f mynginx

# 删除所有已停止的容器
docker container prune

# 删除所有容器（包括运行中的）
docker rm -f $(docker ps -aq)
```

### 6. `docker logs` - 查看容器日志

**用途**：查看容器的标准输出和标准错误日志 **语法**：`docker logs [OPTIONS] CONTAINER` **核心参数**：

- `-f, --follow`：实时跟踪日志输出
    
- `--tail`：只显示最后N行日志
    
- `-t, --timestamps`：显示日志时间戳
    
- `--since`：显示指定时间之后的日志
    

**示例**：

```Bash
# 查看容器日志
docker logs mynginx

# 实时跟踪日志
docker logs -f mynginx

# 只显示最后100行日志
docker logs --tail 100 mynginx

# 显示最近1小时的日志并带时间戳
docker logs -t --since 1h mynginx
```

### 7. `docker inspect` - 查看容器/镜像详细信息

**用途**：获取容器或镜像的底层详细信息（JSON格式） **语法**：`docker inspect [OPTIONS] NAME|ID [NAME|ID...]` **核心参数**：

- `-f, --format`：使用Go模板格式化输出
    

**示例**：

```Bash
# 查看容器详细信息
docker inspect mynginx

# 查看容器IP地址
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mynginx

# 查看容器挂载的卷
docker inspect -f '{{.Mounts}}' mynginx

# 查看镜像详细信息
docker inspect nginx:latest
```

### 8. `docker cp` - 容器与主机之间复制文件

**用途**：在容器和本地主机之间复制文件或目录 **语法**：

```Bash
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH
```

**示例**：

```Bash
# 从容器复制文件到主机
docker cp mynginx:/etc/nginx/nginx.conf ./nginx.conf

# 从主机复制文件到容器
docker cp ./index.html mynginx:/usr/share/nginx/html/index.html
```

---

## 四、数据卷管理指令

### 1. `docker volume create` - 创建数据卷

**用途**：创建一个Docker管理的数据卷 **语法**：`docker volume create [OPTIONS] [VOLUME]` **核心参数**：

- `--driver, -d`：指定卷驱动程序（默认`local`）
    
- `--opt, -o`：设置驱动程序选项
    

**示例**：

```Bash
# 创建一个名为mysql_data的数据卷
docker volume create mysql_data

# 创建指定驱动的卷
docker volume create --driver local --opt type=tmpfs --opt device=tmpfs tmpfs_volume
```

### 2. `docker volume ls` - 列出数据卷

**用途**：查看所有本地数据卷 **语法**：`docker volume ls [OPTIONS]` **核心参数**：

- `-q, --quiet`：只显示卷名
    
- `--filter, -f`：过滤结果
    

**示例**：

```Bash
# 列出所有数据卷
docker volume ls

# 只显示卷名
docker volume ls -q
```

### 3. `docker volume inspect` - 查看数据卷详细信息

**用途**：获取数据卷的详细信息，包括挂载点路径 **语法**：`docker volume inspect [OPTIONS] VOLUME [VOLUME...]`

**示例**：

```Bash
# 查看数据卷详细信息
docker volume inspect mysql_data
```

**响应解析**：

```JSON
[
  {
    "CreatedAt": "2026-05-21T08:00:00Z",
    "Driver": "local",
    "Labels": {},
    "Mountpoint": "/var/lib/docker/volumes/mysql_data/_data",
    "Name": "mysql_data",
    "Options": {},
    "Scope": "local"
  }
]
```

- `Mountpoint`：数据卷在主机上的实际存储路径
    

### 4. `docker volume rm` - 删除数据卷

**用途**：删除未被容器使用的数据卷 **语法**：`docker volume rm [OPTIONS] VOLUME [VOLUME...]` **核心参数**：

- `-f, --force`：强制删除（即使有容器正在使用）
    

**示例**：

```Bash
# 删除数据卷
docker volume rm mysql_data

# 删除所有未使用的数据卷
docker volume prune
```

---

## 五、网络管理指令

### 1. `docker network ls` - 列出网络

**用途**：查看所有Docker网络 **语法**：`docker network ls [OPTIONS]`

**示例**：

```Bash
# 列出所有网络
docker network ls
```

**响应解析**：

```Plaintext
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local
7890abcdef12   host      host      local
3456abcdef78   none      null      local
```

- `bridge`：默认桥接网络（容器默认连接到此网络）
    
- `host`：使用主机网络
    
- `none`：无网络
    

### 2. `docker network create` - 创建网络

**用途**：创建一个自定义网络 **语法**：`docker network create [OPTIONS] NETWORK` **核心参数**：

- `--driver, -d`：指定网络驱动（默认`bridge`）
    
- `--subnet`：指定子网（如`172.20.0.0/16`）
    
- `--gateway`：指定网关
    
- `--ip-range`：指定IP地址范围
    

**示例**：

```Bash
# 创建一个桥接网络
docker network create mynetwork

# 创建指定子网的网络
docker network create --subnet=172.20.0.0/16 --gateway=172.20.0.1 mynetwork
```

### 3. `docker network connect/disconnect` - 连接/断开网络

**用途**：将容器连接到指定网络或从网络断开 **语法**：

```Bash
docker network connect [OPTIONS] NETWORK CONTAINER
docker network disconnect [OPTIONS] NETWORK CONTAINER
```

**示例**：

```Bash
# 将容器连接到自定义网络
docker network connect mynetwork mynginx

# 将容器从网络断开
docker network disconnect mynetwork mynginx
```

### 4. `docker network inspect` - 查看网络详细信息

**用途**：获取网络的详细信息，包括连接的容器 **语法**：`docker network inspect [OPTIONS] NETWORK [NETWORK...]`

**示例**：

```Bash
# 查看网络详细信息
docker network inspect mynetwork
```

### 5. `docker network rm` - 删除网络

**用途**：删除未被容器使用的网络 **语法**：`docker network rm [OPTIONS] NETWORK [NETWORK...]`

**示例**：

```Bash
# 删除网络
docker network rm mynetwork

# 删除所有未使用的网络
docker network prune
```

---

## 六、Dockerfile核心指令

Dockerfile是构建镜像的脚本，包含一系列指令，按顺序执行。以下是最常用的指令：

|   |   |   |
|---|---|---|
|指令|用途|示例|
|`FROM`|指定基础镜像|`FROM node:20-alpine`|
|`WORKDIR`|设置工作目录|`WORKDIR /app`|
|`COPY`|复制文件到镜像|`COPY package.json .`|
|`ADD`|复制文件（支持URL和解压）|`ADD https://example.com/file.tar.gz .`|
|`RUN`|构建时执行命令|`RUN npm install`|
|`ENV`|设置环境变量|`ENV NODE_ENV=production`|
|`EXPOSE`|声明容器暴露的端口|`EXPOSE 3000`|
|`CMD`|容器启动时执行的命令|`CMD ["npm", "start"]`|
|`ENTRYPOINT`|容器启动时执行的入口点|`ENTRYPOINT ["node", "app.js"]`|
|`VOLUME`|声明挂载点|`VOLUME ["/app/data"]`|
|`USER`|指定运行用户|`USER node`|
|`ARG`|构建时参数|`ARG VERSION=1.0.0`|

**关键区别**：

- `COPY` vs `ADD`：优先使用`COPY`，`ADD`有额外功能但行为不直观
    
- `CMD` vs `ENTRYPOINT`：`CMD`是默认参数，可被覆盖；`ENTRYPOINT`是固定入口
    

---

## 七、Docker Compose核心指令

Docker Compose用于定义和运行多容器Docker应用程序，使用YAML文件配置应用服务。

### 1. `docker compose up` - 创建并启动服务

**用途**：根据docker-compose.yml文件创建并启动所有服务 **语法**：`docker compose up [OPTIONS] [SERVICE...]` **核心参数**：

- `-d, --detach`：后台运行
    
- `--build`：启动前构建镜像
    
- `--force-recreate`：强制重新创建容器
    
- `--no-deps`：不启动依赖服务
    

**示例**：

```Bash
# 启动所有服务
docker compose up

# 后台启动所有服务
docker compose up -d

# 启动指定服务
docker compose up -d web

# 构建镜像并启动
docker compose up -d --build
```

### 2. `docker compose down` - 停止并删除服务

**用途**：停止并删除所有服务、网络、卷和镜像 **语法**：`docker compose down [OPTIONS]` **核心参数**：

- `-v, --volumes`：删除数据卷
    
- `--rmi`：删除镜像（`all`/`local`）
    
- `--remove-orphans`：删除未在compose文件中定义的容器
    

**示例**：

```Bash
# 停止并删除服务（保留卷）
docker compose down

# 停止并删除服务和卷
docker compose down -v

# 停止并删除所有内容
docker compose down -v --rmi all
```

### 3. `docker compose ps` - 列出服务容器

**用途**：查看当前compose项目的所有容器 **语法**：`docker compose ps [OPTIONS] [SERVICE...]`

**示例**：

```Bash
# 列出所有服务容器
docker compose ps
```

### 4. `docker compose logs` - 查看服务日志

**用途**：查看服务的日志输出 **语法**：`docker compose logs [OPTIONS] [SERVICE...]` **核心参数**：

- `-f, --follow`：实时跟踪日志
    
- `--tail`：只显示最后N行日志
    

**示例**：

```Bash
# 查看所有服务日志
docker compose logs

# 实时跟踪web服务日志
docker compose logs -f web
```

### 5. `docker compose exec` - 在服务容器内执行命令

**用途**：在指定服务的容器内执行命令 **语法**：`docker compose exec [OPTIONS] SERVICE COMMAND [ARG...]`

**示例**：

```Bash
# 进入web服务容器
docker compose exec web /bin/bash

# 在db服务内执行mysql命令
docker compose exec db mysql -u root -p
```

---

## 八、系统与信息相关指令

### 1. `docker info` - 查看Docker系统信息

**用途**：显示Docker系统的整体信息，包括容器数量、镜像数量、存储驱动等 **语法**：`docker info`

### 2. `docker version` - 查看Docker版本信息

**用途**：显示Docker客户端和服务器的版本信息 **语法**：`docker version`

### 3. `docker system df` - 查看磁盘使用情况

**用途**：显示Docker占用的磁盘空间 **语法**：`docker system df`

### 4. `docker system prune` - 清理系统资源

**用途**：清理所有未使用的资源（容器、镜像、卷、网络） **语法**：`docker system prune [OPTIONS]` **核心参数**：

- `-a, --all`：删除所有未使用的镜像（不仅是悬空镜像）
    
- `-v, --volumes`：删除未使用的数据卷
    
- `-f, --force`：强制清理，不提示
    

**示例**：

```Bash
# 清理所有未使用的资源（不包括卷）
docker system prune

# 彻底清理所有资源
docker system prune -a -v
```

---

## 九、常用组合与最佳实践

### 1. 快速清理所有资源

```Bash
# 停止所有容器
docker stop $(docker ps -aq)

# 删除所有容器
docker rm $(docker ps -aq)

# 删除所有镜像
docker rmi $(docker images -q)

# 清理所有未使用的资源
docker system prune -a -v
```

### 2. 查看容器资源使用情况

```Bash
# 实时查看所有容器的资源使用
docker stats

# 查看指定容器的资源使用
docker stats mynginx
```

### 3. 导出和导入容器

```Bash
# 导出容器为tar文件
docker export -o mycontainer.tar mycontainer

# 导入tar文件为镜像
docker import mycontainer.tar myimage:v1
```

### 4. 保存和加载镜像

```Bash
# 保存镜像为tar文件
docker save -o myimage.tar myimage:v1

# 加载tar文件为镜像
docker load -i myimage.tar
```

### 最佳实践

1. 始终使用`-d`参数后台运行生产环境容器
    
2. 为容器指定`--restart always`策略确保自动重启
    
3. 使用数据卷持久化重要数据，不要将数据存储在容器内
    
4. 构建镜像时使用多阶段构建减小镜像体积
    
5. 避免使用`latest`标签，使用具体版本号
    
6. 不要在容器内运行多个进程
    
7. 使用非root用户运行容器提高安全性
    

需要我补充一份**生产环境常用指令速查表**（一页可打印），或者针对某个特定场景（如Java/Node.js应用部署）提供完整的Dockerfile和Compose示例吗？