
本教程基于 Windows 10 1809 及以上版本（内置 OpenSSH 客户端）编写，全程使用系统原生 PowerShell 操作，无需额外安装 Git Bash 即可完成，同时兼容 Git 客户端场景。

## 一、前置准备：启用 OpenSSH 客户端

Windows 10 1809 及以上版本已内置 OpenSSH 客户端，仅需手动启用；若版本过低，建议先升级系统，或安装 Git for Windows（自带 SSH 工具集）。

### 方法 1：图形界面启用（推荐新手）

1. 按下 `Win + I` 打开系统设置，依次进入 **应用 → 可选功能**
    
2. 点击页面顶部 **查看功能**（或 “添加功能”），在搜索框输入 `OpenSSH`
    
3. 在结果中勾选 **OpenSSH 客户端**，点击「下一步」→「安装」
    
4. 安装完成后重启终端即可生效
    

### 方法 2：PowerShell 命令启用（管理员权限）

1. 按下 `Win + R`，输入 `powershell`，按下 `Ctrl + Shift + Enter` 以管理员身份打开 PowerShell
    
2. 执行安装命令：
    

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

3. 验证安装是否成功：
    

```powershell
ssh -V
```

若输出版本号（如 `OpenSSH_for_Windows_9.2p1`），则说明安装成功

## 二、生成 SSH 密钥对

### 1. 先检查是否已有密钥

避免覆盖原有密钥，先执行命令查看：

```powershell
ls ~/.ssh
```

若输出中已有 `id_ed25519`、`id_ed25519.pub` 等密钥文件，可直接跳过生成步骤；若无，继续下一步。

### 2. 生成新的密钥对

推荐使用安全性更高的 ed25519 算法，执行以下命令（替换为你的 GitHub 注册邮箱）：

```powershell
ssh-keygen -t ed25519 -C "201402321@qq.com"
```

- 若系统不支持 ed25519（极老版本），使用 RSA 兼容方案：
    
    ```powershell
    ssh-keygen -t rsa -b 4096 -C "your_github_email@example.com"
    ```
    

执行命令后，按提示操作：

1. **密钥保存路径**：直接按回车，使用默认路径 `C:\Users\你的用户名.ssh\id_ed25519`
    
2. **设置密钥密码（Passphrase）**：可选，输入密码后每次使用密钥都需验证，大幅提升安全性；无需密码则直接按两次回车
    

生成成功后，会在 `.ssh` 目录下得到两个核心文件：

- `id_ed25519`：**私钥文件，严禁泄露、外传**
    
- `id_ed25519.pub`：公钥文件，后续需配置到 GitHub 中
    

## 三、启动 SSH Agent 并添加私钥

SSH Agent 用于管理私钥，避免重复输入密码，需先配置服务并添加密钥。

1. **以管理员身份打开 PowerShell**，执行以下命令，设置 ssh-agent 服务自动启动并立即运行：
    

```powershell
# 设置服务开机自动启动
Set-Service -Name ssh-agent -StartupType Automatic
# 启动服务
Start-Service ssh-agent
```

2. **关闭管理员终端，重新打开普通权限 PowerShell**，执行命令添加私钥到 Agent：
    

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

若设置了密钥密码，此时输入密码即可完成添加

## 四、复制公钥并添加到 GitHub 账户

### 1. 复制公钥内容

PowerShell 中执行以下命令，一键复制公钥到剪贴板：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

也可手动打开 `C:\Users\你的用户名.ssh\id_ed25519.pub`，用记事本打开，**完整复制整行内容**（不要遗漏开头的 `ssh-ed25519`，也不要添加多余换行、空格）

### 2. 配置到 GitHub 账户

1. 浏览器登录你的 GitHub 账户，点击右上角头像 → 选择 **Settings（设置）**
    
2. 左侧菜单栏找到并点击 **SSH and GPG keys**
    
3. 点击右上角绿色的 **New SSH key** 按钮
    
4. 填写配置：
    
    1. **Title**：给密钥起一个标识名，比如 `Windows10-办公电脑`，方便区分设备
        
    2. **Key type**：保持默认 `Authentication key`
        
    3. **Key**：粘贴刚才复制的完整公钥内容
        
5. 点击 **Add SSH key**，完成添加（可能需要验证 GitHub 账户密码）
    

## 五、测试 SSH 连接

回到 PowerShell，执行以下命令测试与 GitHub 的 SSH 连通性：

```powershell
ssh -T git@github.com
```

- 首次连接会提示 `Are you sure you want to continue connecting (yes/no/[fingerprint])?`，输入 `yes` 回车即可
    
- 若输出 `Hi 你的GitHub用户名! You've successfully authenticated, but GitHub does not provide shell access.`，说明配置成功！
    

## 六、Git 客户端配套配置（可选）

若你使用 Git 进行代码提交、克隆等操作，还需配置 Git 全局账户信息，确保提交记录与 GitHub 账户匹配：

1. 配置全局用户名和邮箱（替换为你的 GitHub 信息）：
    

```powershell
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub注册邮箱"
```

2. 仓库远程地址切换为 SSH（可选）： 若已有仓库使用 HTTPS 地址，可切换为 SSH 地址，后续无需重复输入账号密码：
    

```powershell
# 进入仓库目录后执行
git remote set-url origin git@github.com:你的用户名/仓库名.git
```

## 七、常见问题排查

1. **ssh 命令提示 “不是内部或外部命令”**
    
    1. 确认已按步骤启用 OpenSSH 客户端，安装完成后重启终端；
        
    2. 若系统版本过低，直接安装 Git for Windows，使用 Git Bash 执行所有 SSH 命令。
        
2. **测试连接提示** **`Permission denied (publickey)`**
    
    1. 检查公钥是否完整、正确地粘贴到 GitHub，无多余空格 / 换行；
        
    2. 确认私钥已通过 `ssh-add` 成功添加到 ssh-agent，执行 `ssh-add -l` 可查看已添加的密钥；
        
    3. 检查私钥文件权限，Windows 需确保只有当前管理员账户能访问该文件（右键私钥文件 → 属性 → 安全，删除无关用户权限）。
        
3. **连接超时、无法访问 GitHub**
    
    1. 检查网络是否能正常访问 GitHub 官网；
        
    2. 若使用代理，需在 `.ssh` 目录下新建 `config` 文件，添加代理配置（示例）：
        
        ```Plain
        Host github.com
          HostName github.com
          User git
          ProxyCommand connect -S 127.0.0.1:10808 %h %p
        ```
        
4. **每次使用都要输入密钥密码**
    
    1. 确认 ssh-agent 服务已正常启动，且已执行 `ssh-add` 添加私钥；
        
    2. 检查服务启动类型是否为 Automatic，确保开机自动运行。