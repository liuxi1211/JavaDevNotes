# 🖥️ Chrome 调试笔记：如何捕获新标签页的初始网络请求？

### 1. 问题现象 (The Problem)

在 **A 页面**打开开发者工具（F12），点击链接跳转到 **B 页面**（新标签页）。

- **痛点**：B 页面加载时的所有初始接口（如 HTML 文档、首屏 JS/CSS）在 Network 面板中一片空白。
- **误区**：以为勾选了 `Preserve log` 就能保留记录，但实际上旧标签页看不到新标签页的请求，新标签页的 DevTools 根本没打开。

### 2. 核心原理 (Why it happens)

- **进程隔离**：每个标签页是独立的渲染进程。新标签页默认不会继承旧标签页的 DevTools 状态。
- **Preserve log 的真实作用**：它仅在**同一个 DevTools 实例**中生效（即单页应用刷新或路由跳转时）。如果新标签页的 DevTools 从未被打开过，就没有“实例”来保存日志。
- **结论**：必须让新标签页**物理上自动打开 DevTools**，才能记录从 0ms 开始的请求。

---

### 3. 解决方案 (Solutions)

#### ✅ 方案一：开启“弹窗自动打开 DevTools”（最推荐，一劳永逸）

**适用场景**：日常开发，点击 `target="_blank"` 链接或 `window.open` 弹窗。

1. 在任意页面打开 F12。
2. 点击右上角 **三个点 (⋮)** -> **Settings**。
3. 找到 **Global** (全局) 部分。
4. 勾选 **✅ Auto-open DevTools for popups** (为弹出窗口自动打开 DevTools)。
5. **关键**：源页面的 DevTools 必须保持打开状态，此设置才生效。

> **效果**：点击链接跳转新标签页时，新页面会“自带”打开的 DevTools，且 Network 面板会自动勾选 `Preserve log`，完美保留跳转链路。

#### 🚀 方案二：命令行启动参数（强制全局生效）

**适用场景**：深度调试，需要所有新标签页（包括非弹出窗口）都打开 DevTools。

- **Windows**:
    
    ```bash
    bashstart chrome --auto-open-devtools-for-tabs
    ```
    
- **macOS**:
    
    ```bash
    bashopen -a "Google Chrome" --args --auto-open-devtools-for-tabs
    ```
    
- **Linux**:
    
    ```bash
    bashgoogle-chrome --auto-open-devtools-for-tabs
    ```
    

> **注意**：关闭浏览器后失效，下次需重新执行命令。

#### 🛠️ 方案三：手动“预判” + 外部抓包（兜底方案）

**适用场景**：无法修改浏览器设置，或需要抓取 HTTPS/移动端/重定向链的完整数据。

1. **手动操作**：
    - 在点击链接**前**，先在当前页按 `F12`。
    - 点击链接后，**极速**切换到新标签页并按 `F12`。
    - _缺点_：可能会错过最开始的 1-2 个重定向请求。
2. **外部工具（专业首选）**：
    - 使用 **Charles**、**Fiddler** 或 **Proxyman**。
    - **原理**：作为系统代理拦截所有流量，不依赖浏览器 DevTools 是否打开。
    - **优势**：跨域、跨标签页、甚至跨设备（手机抓包）都能完整记录。

#### 💣 方案四：代码断点（Debugger）

**适用场景**：调试跳转前的特定逻辑，确认接口返回正确后再跳转。

```javascript
javascript// 在触发跳转的代码前插入
debugger; 
window.location.href = '/new-page';
```

- 代码执行到此处会暂停，此时可在 Network 面板查看当前所有请求详情，按 `F8` 继续执行才会跳转。

---

### 4. 避坑指南 (FAQ)

|问题|解释|
|---|---|
|**Preserve log 没用？**|它只保留**已打开**的 DevTools 窗口的日志。新标签页没开 DevTools，它就没法保留。|
|**勾选了 Auto-open 还是没反应？**|检查**源页面**的 DevTools 是否处于关闭状态？该功能需要源页面 DevTools 打开作为“触发器”。|
|**为什么 Firefox 更好用？**|Firefox 的 `Persist Logs` 功能在跨标签页时表现更稳定，如果 Chrome 实在无法满足，可临时切换 Firefox 调试。|

---

### 5. 极简操作清单 (Cheat Sheet)

> **日常开发 SOP**：

1. 打开 Chrome -> 按 `F12`。
2. 右上角 **Settings** -> 勾选 **Auto-open DevTools for popups**。
3. **保持 F12 打开**，去点击页面链接。
4. 新标签页打开 -> DevTools 自动弹出 -> 点击 **Network** -> 勾选 **Preserve log**。
5. 开始调试 🎉。

> **紧急情况 SOS**：

- 如果不小心关掉了新标签页的 DevTools，且没开自动打开：
    - 立刻按 `F12` 打开。
    - 在 Network 面板点击 **Preserve log**（虽然抓不到之前的，但能保留之后的）。
    - **或者**：直接打开 **Charles**，看外部抓包记录（最稳）。

---

**Tag**: `Chrome` `DevTools` `Debug` `Network` `Frontend`