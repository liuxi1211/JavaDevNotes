
**说明**：Miniforge3 是 conda-forge 社区维护的轻量级 Conda 发行版，**默认集成 Mamba**（Conda 的多线程加速替代品，速度提升 3-10 倍）。本手册优先推荐使用 `mamba` 命令，同时标注等效 `conda` 命令。

## 一、基础信息与版本查看

|   |   |   |   |
|---|---|---|---|
|命令|作用|参数说明|示例|
|`mamba --version`|查看 Mamba 版本|无|`mamba --version`|
|`conda --version`|查看 Conda 版本|无|`conda --version`|
|`mamba info`|显示详细环境信息|无|`mamba info`|
|`mamba info --envs`|列出所有虚拟环境|`--envs` 仅显示环境列表|`mamba info --envs`|
|`mamba info <package>`|查看指定包的详细信息|包名|`mamba info numpy`|

## 二、安装与卸载

### 2.1 下载与安装

```bash
# Linux/macOS 自动下载对应架构最新版
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"

# 交互式安装（推荐）
bash Miniforge3-$(uname)-$(uname -m).sh

# 静默安装（服务器/CI 环境）
bash Miniforge3-$(uname)-$(uname -m).sh -b -p $HOME/miniforge3
# -b: 批处理模式，不显示协议和提示
# -p: 指定安装路径
```

### 2.2 更新 Miniforge

```bash
# 更新 Conda 和 Mamba 到最新版
mamba update -n base conda mamba -y

# 或下载新版安装包并更新
bash Miniforge3-Linux-x86_64.sh -u
# -u: 更新现有安装，保留环境和配置
```

### 2.3 完全卸载

```bash
# 1. 反向初始化 shell 配置
conda init --reverse

# 2. 关闭当前终端，打开新终端

# 3. 删除 Miniforge 安装目录
rm -rf $(conda info --base)

# 4. 删除剩余配置文件
rm -rf ~/.condarc ~/.conda ~/.continuum
```

## 三、环境管理（核心）

### 3.1 基本操作

|   |   |   |   |
|---|---|---|---|
|命令|作用|参数说明|示例|
|`mamba create -n <env_name> [python=version] [packages]`|创建新环境|`-n/--name`: 环境名 `python=3.10`: 指定 Python 版本 `packages`: 预装包列表 `-y`: 自动确认|`mamba create -n myai python=3.10 numpy pandas -y`|
|`conda activate <env_name>`|激活环境|环境名|`conda activate myai`|
|`mamba deactivate`|退出当前环境|无|`mamba deactivate`|
|`conda env list`|列出所有环境|无|`conda env list`|
|`mamba env remove -n <env_name>`|删除环境|`-n/--name`: 环境名 `-y`: 自动确认|`mamba env remove -n old_env -y`|
|`conda remove -n <env_name> --all`|等效删除环境命令|`--all`: 删除环境内所有包|`conda remove -n old_env --all -y`|

### 3.2 环境复制与重命名

```bash
# 复制环境
mamba create --clone <source_env> -n <new_env>
# 示例：mamba create --clone myai -n myai_backup

# 重命名环境（通过复制+删除实现）
mamba create --clone old_name -n new_name
mamba env remove -n old_name -y
```

### 3.3 环境导出与复现

```bash
# 导出当前环境配置（包含所有依赖）
conda env export > environment.yml

# 导出环境（不包含构建哈希，跨平台兼容性更好）
conda env export --no-builds > environment.yml

# 导出指定环境
conda env export -n myai > myai_env.yml

# 从配置文件创建环境
mamba env create -f environment.yml

# 更新现有环境（根据配置文件）
mamba env update -f environment.yml --prune
# --prune: 删除配置文件中未列出的包
```

## 四、包管理（核心）

### 4.1 基本操作

|   |   |   |   |
|---|---|---|---|
|命令|作用|参数说明|示例|
|`mamba install <package>`|安装包|包名 `=version`: 指定版本 `-c <channel>`: 指定频道 `-y`: 自动确认|`mamba install tensorflow=2.15.0 -c pytorch -y`|
|`mamba install <pkg1> <pkg2> ...`|安装多个包|空格分隔包名|`mamba install pandas matplotlib scikit-learn`|
|`mamba update <package>`|更新单个包|包名|`mamba update numpy`|
|`mamba update --all`|更新当前环境所有包|`--all`: 更新所有包|`mamba update --all -y`|
|`mamba remove <package>`|卸载包|包名|`mamba remove matplotlib -y`|
|`mamba uninstall <package>`|等效卸载命令|包名|`mamba uninstall matplotlib -y`|
|`mamba list`|列出当前环境已安装的包|无|`mamba list`|
|`mamba list <package>`|查看指定包的安装信息|包名|`mamba list numpy`|

### 4.2 包搜索与查询

```bash
# 搜索包
mamba search <package>
# 示例：mamba search pytorch

# 搜索指定版本范围的包
mamba search "python>=3.8,<3.12"

# 查看包的可用版本
mamba search numpy --info
```

## 五、配置管理

### 5.1 基本配置命令

```bash
# 查看所有配置
conda config --show

# 查看指定配置项
conda config --show channels

# 添加配置项
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/

# 设置配置项（覆盖原有值）
conda config --set ssl_verify true

# 删除配置项
conda config --remove channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/

# 恢复默认配置
conda config --reset
```

### 5.2 国内镜像源配置（加速下载）

```bash
# 中科大源（推荐）
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes

# 上海交大源
conda config --add channels https://mirrors.sjtug.sjtu.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.sjtug.sjtu.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.sjtug.sjtu.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
```

## 六、缓存与清理

```bash
# 清理未使用的包缓存
conda clean -p -y
# -p: 删除未被任何环境使用的包

# 清理所有下载的包缓存
conda clean -t -y
# -t: 删除所有压缩包

# 全面清理（推荐定期执行）
conda clean -a -y
# -a: 清理所有缓存（包、索引、临时文件等）

# 查看缓存占用空间
conda clean --dry-run -a
# --dry-run: 模拟执行，不实际删除文件
```

## 七、高级技巧与排障

### 7.1 环境运行命令

```bash
# 在指定环境中运行命令（无需激活）
mamba run -n myai python script.py

# 在指定环境中启动 Jupyter Lab
mamba run -n myai jupyter lab
```

### 7.2 包冲突解决

```bash
# 使用 Mamba 解决冲突（比 Conda 快得多）
mamba install <package>

# 强制重新安装包
mamba install --force-reinstall <package>

# 忽略依赖检查（谨慎使用）
mamba install --no-deps <package>
```

### 7.3 常见问题排查

```bash
# 检查环境变量
echo $PATH

# 查看 Conda 配置文件位置
conda config --show-sources

# 重置 base 环境
mamba update -n base --all -y

# 修复损坏的环境
mamba install --revision 0
# 回滚到环境创建时的状态
```

## 八、Mamba vs Conda 命令对照表

|   |   |   |   |
|---|---|---|---|
|操作|Conda 命令|Mamba 命令|提速效果|
|安装包|`conda install pkg`|`mamba install pkg`|3-5 倍|
|更新包|`conda update pkg`|`mamba update pkg`|5-10 倍|
|搜索包|`conda search pkg`|`mamba search pkg`|10-20 倍|
|创建环境|`conda create -n env`|`mamba create -n env`|3-5 倍|
|解决冲突|`conda install pkgA pkgB`|`mamba install pkgA pkgB`|10-100 倍|

**注意**：环境激活 / 退出命令 `conda activate`/`conda deactivate` 是 Conda 特有的，Mamba 没有对应的命令，直接使用 Conda 命令即可。

需要我补充一份**常用环境配置模板**（包含 Python 3.10/3.11/3.12 的基础数据科学、深度学习环境），方便你直接复制使用吗？