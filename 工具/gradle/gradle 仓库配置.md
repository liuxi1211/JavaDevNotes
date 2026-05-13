针对你提出的三个问题，我来做一轮精准优化，下面逐一解答：

---

### 问题1：全局只配置了依赖的镜像仓库，那插件下载不了怎么办？

**核心原因**：Gradle 插件（如 `id 'com.android.application'` 或 `id 'org.springframework.boot'`）默认是通过 **`gradlePluginPortal()`** 仓库解析的，这个仓库通常也被镜像覆盖。如果你在全局 `init.gradle` 里只配置了 `allprojects.repositories` 里的 `maven { url '阿里云public' }`，这个配置**只对项目依赖（dependencies）生效，对插件管理（pluginManagement）无效**。

**解决方案**：需要在全局层面同时覆盖**插件仓库**。在 `~/.gradle/init.gradle`（或 `init.gradle.kts`）中使用 `settingsEvaluated` 钩子来配置 `pluginManagement`：

```groovy
// ~/.gradle/init.gradle
settingsEvaluated { settings ->
    settings.pluginManagement {
        repositories {
            // 阿里云 Gradle 插件专用镜像
            maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
            // 保留官方仓库作为备选
            gradlePluginPortal()
        }
    }
}

allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }
}
```

**说明**：`settingsEvaluated` 会在每个项目的 `settings.gradle` 被解析后执行，从而注入全局插件仓库配置。这样无论项目自身是否配置了 `pluginManagement`，都能确保插件下载走国内镜像。

**备选方案**（更简单，但需要每个项目配合）：在项目的 `settings.gradle` 中显式配置 `pluginManagement { repositories { ... } }`，并把镜像放在第一位。这是项目级配置，优先级高于全局钩子。

---

### 问题2：Gradle Home 默认位置是哪里？如果要配置 Home 位置和依赖保存位置怎么配置？

**默认位置**：

| 操作系统 | 路径 |
| :--- | :--- |
| Windows | `C:\Users\<用户名>\.gradle` |
| Linux / macOS | `~/.gradle` |

**如何自定义 Gradle Home**：

通过环境变量 `GRADLE_USER_HOME` 设置。例如在 `~/.bashrc` 或系统环境变量中：

```bash
export GRADLE_USER_HOME=/path/to/your/custom/gradle-home
```

设置后，整个 Gradle 用户目录（包括缓存、wrapper 分发、daemon 日志等）都会迁移到新路径。

**如何单独配置依赖缓存位置**（不改变整个 Gradle Home）：

Gradle 没有直接单独指定依赖缓存目录的官方参数，但可以通过**软链接**或修改 `init.gradle` 中的 `DependencyCache` 相关配置（不推荐）。更常见的方式是直接修改 `GRADLE_USER_HOME`，因为 Gradle 的目录结构是固定的：

- `caches/modules-2/files-2.1/` → 依赖 jar 包
- `caches/transforms-3/` → 转换后的 artifacts
- `wrapper/dists/` → Gradle Wrapper 分发包

如果你希望将**依赖缓存**移到另一个磁盘分区，可以这样做：

1. 设置 `GRADLE_USER_HOME=/fast-disk/gradle-home`
2. 原 `~/.gradle` 中其他非缓存内容（如 `gradle.properties`、`init.gradle`）通过软链接保留：  
   `ln -s /fast-disk/gradle-home/caches ~/.gradle/caches`

但**强烈建议直接使用统一的 `GRADLE_USER_HOME`**，简单可靠。

---

### 问题3：项目中配置的仓库地址和全局用户维度哪个优先级高？生效规则是什么？

**结论**：**项目级配置优先级高于全局用户级配置**，除非全局配置使用了强制覆盖手段。

**具体生效规则**：

| 配置层级 | 配置位置 | 生效时机 | 优先级 |
| :--- | :--- | :--- | :--- |
| 项目级 `repositories` | `build.gradle` 中的 `repositories {}` | 构建时直接读取 | **最高** |
| 项目级 `pluginManagement.repositories` | `settings.gradle` 中的 `pluginManagement { repositories {} }` | settings 解析时读取 | **最高** |
| 全局用户级 `allprojects.repositories` | `~/.gradle/init.gradle` 中的 `allprojects { repositories { ... } }` | 在项目配置之前注入 | 低于项目级（会被项目级覆盖/追加） |
| 全局用户级 `settingsEvaluated` 钩子注入的 `pluginManagement` | `~/.gradle/init.gradle` 中的 `settingsEvaluated` | 在 settings 解析后执行 | 如果有 `repositoriesMode` 或项目自身定义了 `pluginManagement`，则行为取决于合并规则 |

**详细说明**：

1. **对于 `dependencies` 中的普通依赖**：  
   Gradle 会合并所有仓库列表。具体顺序：  
   - 先添加项目级 `repositories` 中声明的仓库
   - 再添加全局 `init.gradle` 中通过 `allprojects` 注入的仓库  
   - **查找依赖时按列表顺序从前往后查找**，所以项目级写在最前面的仓库会优先被使用。  
   - 如果项目级没有配置任何仓库，则完全使用全局注入的仓库列表。

2. **对于 `pluginManagement.repositories`**：  
   - 如果项目 `settings.gradle` 中显式定义了 `pluginManagement { repositories { ... } }`，则全局钩子 `settingsEvaluated` 中追加的仓库只会**添加**到列表末尾（如果项目没有设置 `repositoriesMode` 为 `FAIL_ON_PROJECT_REPOS`）。  
   - 由于 Gradle 默认按照仓库声明顺序查找插件，项目级仓库在前面就会优先使用。  
   - 如果想强制全局覆盖项目级，可以在 `settingsEvaluated` 中使用 `repositories.clear()` 再添加，但这属于破坏性操作，不推荐。

**最佳实践**：

- **团队开发**：在项目的 `settings.gradle` 和 `build.gradle` 中明确配置所有仓库（包括镜像），并提交到版本控制，确保所有人一致。
- **个人环境**：使用 `init.gradle` 全局配置镜像作为保底，但项目级配置会覆盖它。想要全局强制镜像，需在 `init.gradle` 中使用 `repositories.clear()` 和 `add` 等操作，但这可能破坏某些项目的特殊配置（如内部私有仓库）。稳妥做法是只在全局配置速度快的镜像，并让项目级可覆盖。

---

如果你希望我针对上述三个问题，直接输出一个**整合后的、独立的新回答**（代替原来的长文），我可以为你整理成简洁的三点解答格式。请告诉我是否需要这样输出。