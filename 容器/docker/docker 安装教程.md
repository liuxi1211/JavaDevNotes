
## 一、前置说明

### 支持的主流系统版本

- **Ubuntu**：20.04 LTS / 22.04 LTS / 24.04 LTS（推荐生产环境）
- **CentOS/RHEL**：7.x / 8.x / 9 Stream
- **Windows**：10/11 专业版/企业版（需开启 Hyper-V）
- **macOS**：10.15+（Intel/Apple Silicon 均支持）

### 安装包组成

Docker 官方标准安装包含核心组件：

- `docker-ce`：Docker 社区版引擎核心
- `docker-ce-cli`：命令行客户端
- `containerd.io`：容器运行时
- `docker-buildx-plugin`：镜像构建插件
- `docker-compose-plugin`：容器编排插件（替代旧版 `docker-compose` 二进制）

---

## 二、Ubuntu 系统标准安装（生产环境推荐）

### 2.1 卸载系统自带旧版本

系统默认仓库的旧版 Docker 包名为 `docker.io`、`docker-engine`，先卸载避免冲突：

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### 2.2 安装依赖并配置国内软件源

使用阿里云镜像源，解决官方源下载慢、超时问题。

```bash
# 1. 更新软件包索引
sudo apt update

# 2. 安装基础依赖
sudo apt install -y ca-certificates curl gnupg lsb-release

# 3. 创建密钥存储目录
sudo install -m 0755 -d /etc/apt/keyrings

# 4. 导入 Docker 官方 GPG 密钥（国内镜像源）
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 5. 添加 Docker 软件源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.3 安装 Docker Engine

```bash
# 更新源索引
sudo apt update

# 安装最新稳定版 Docker 全家桶
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> 如需安装指定版本，可先查询可用版本：
> 
> ```bash
> apt-cache madison docker-ce
> # 指定版本安装示例：sudo apt install docker-ce=5:27.0.3-1~ubuntu.22.04~jammy
> ```

### 2.4 启动服务并设置开机自启

```bash
# 启动 Docker
sudo systemctl start docker

# 设置开机自启（生产环境必开）
sudo systemctl enable docker

# 查看服务状态
sudo systemctl status docker
```

### 2.5 验证安装

```bash
# 查看版本
docker --version
# 输出示例：Docker version 27.0.3, build 7d4bcd8

# 运行测试镜像（验证核心功能正常）
sudo docker run hello-world
```

输出 `Hello from Docker!` 即表示安装成功。

---

## 三、CentOS / RHEL 系统标准安装

### 3.1 卸载旧版本

```bash
sudo yum remove -y docker docker-client docker-client-latest docker-common \
                  docker-latest docker-latest-logrotate docker-logrotate \
                  docker-engine
```

### 3.2 配置国内软件源

```bash
# 安装 yum 工具包
sudo yum install -y yum-utils

# 添加阿里云 Docker 源
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 3.3 安装 Docker

```bash
# 安装最新稳定版
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.4 启动与验证

```bash
# 启动并开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 验证
sudo docker --version
sudo docker run hello-world
```

---

## 四、Linux 快速一键安装（测试/开发环境）

适合快速部署、不需要精细配置的场景，国内网络自动使用镜像源加速：

```bash
# 官方脚本 + 阿里云镜像加速
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

执行完成后，手动启动服务并设置开机自启即可，步骤同上文。

---

## 五、安装后必做优化配置

### 5.1 配置国内镜像加速器（解决拉取超时）

这是国内环境必做配置，解决 `docker pull`、`docker search` 超时问题。

```bash
# 创建/修改 Docker 配置文件
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.ccs.tencentyun.com"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

> 额外配置说明：限制容器日志大小，避免日志占满磁盘，生产环境强烈建议配置。

重启 Docker 使配置生效：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证配置生效
docker info | grep -A 5 "Registry Mirrors"
```

### 5.2 非 root 用户运行 Docker（免 sudo）

默认只有 root 用户和 docker 组用户可操作 Docker，配置后普通用户无需每次加 `sudo`。

```bash
# 创建 docker 用户组（安装时通常已自动创建）
sudo groupadd docker

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER
```

**生效方式**：

- 方式1：退出当前终端，重新登录服务器
- 方式2：执行 `newgrp docker` 临时刷新组权限

验证：直接执行 `docker ps` 不报错即生效。

> ⚠️ 安全提示：docker 组用户等价于 root 权限，生产环境谨慎开放。

### 5.3 验证 Docker Compose

新版 Docker 已内置 Compose 插件，无需单独安装：

```bash
docker compose version
# 输出示例：Docker Compose version v2.28.1
```

旧版 `docker-compose`（横杠）命令已废弃，统一使用 `docker compose`（空格）。

---

## 六、离线安装方法（内网/无外网环境）

适用于完全不能联网的服务器，提前在有网机器下载安装包，再拷贝到目标服务器安装。

### 6.1 Ubuntu 离线安装

1. **下载安装包**：前往 [阿里云 Docker 镜像站](https://mirrors.aliyun.com/docker-ce/linux/ubuntu/dists/)，下载对应系统版本的 `.deb` 包，需要下载：
    
    - `containerd.io_xxx.deb`
    - `docker-ce-cli_xxx.deb`
    - `docker-ce_xxx.deb`
    - `docker-compose-plugin_xxx.deb`
2. **安装**：将所有 deb 包上传到服务器，执行：
    
    ```bash
    sudo dpkg -i *.deb
    ```
    

### 6.2 CentOS 离线安装

1. 下载对应版本的 `.rpm` 安装包
2. 上传后执行：
    
    ```bash
    sudo yum localinstall -y *.rpm
    ```
    

安装完成后，启动服务、配置镜像加速器步骤同在线安装。

---

## 七、完全卸载 Docker

### Ubuntu 系统

```bash
# 1. 卸载软件包
sudo apt purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 2. 删除所有数据（镜像、容器、卷、配置）
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
```

### CentOS 系统

```bash
# 1. 卸载软件包
sudo yum remove -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 2. 删除数据
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
```

---

## 八、Windows / macOS 安装（开发环境）

### Windows 10/11

1. **前置要求**：开启 Hyper-V 功能（控制面板 → 程序 → 启用或关闭 Windows 功能 → 勾选 Hyper-V）
2. **下载安装包**：前往 [Docker 官网](https://www.docker.com/products/docker-desktop/) 下载 `Docker Desktop for Windows`
3. 双击安装包，按向导完成安装，重启电脑即可
4. 启动 Docker Desktop，在设置 → Docker Engine 中添加镜像加速器配置，与 Linux 一致

### macOS

1. 下载 `Docker Desktop for Mac`（区分 Intel/Apple Silicon 芯片）
2. 拖拽安装到应用程序，启动即可
3. 同样在设置中配置镜像加速器提升速度

---

## 九、常见安装问题排查

1. **`apt update` 报错签名验证失败** 重新导入 GPG 密钥，检查密钥文件权限是否正确。
    
2. **启动 Docker 失败，提示 `Failed to start Docker Application Container Engine`** 检查 `/etc/docker/daemon.json` 语法是否正确，JSON 格式不允许末尾多余逗号。
    
3. **非 root 用户执行报错 `permission denied`** 确认用户已加入 docker 组，且已重新登录生效；或检查 `/var/run/docker.sock` 权限。
    

需要我补充某个特定系统版本的安装细节，或者 Docker 生产环境加固（安全、资源限制）的配置吗？