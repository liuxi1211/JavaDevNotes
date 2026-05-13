## 一、Gradle 入门基础

### 1.1 构建工具概述
- 构建工具的作用与演进（Make → Ant → Maven → Gradle）
- Gradle 的核心优势：灵活性、性能、DSL、增量构建
- Gradle 与 Maven 对比
- Gradle 版本发布线与选择建议

### 1.2 安装与环境配置
- 手动安装（从官网下载 ZIP，配置环境变量）
- 使用 SDKMAN! / Homebrew / Chocolatey 安装
- 验证安装：`gradle -v`
- JVM 版本要求与 `JAVA_HOME` 配置
- `GRADLE_USER_HOME` 介绍（缓存、Wrapper 存储位置）

### 1.3 Gradle Wrapper
- Wrapper 的作用与意义
- 生成 Wrapper 文件：`gradle wrapper`
- `gradle/wrapper/gradle-wrapper.properties` 详解
- `gradlew` / `gradlew.bat` 脚本解读
- 项目中使用 Wrapper 的最佳实践（提交到版本控制）

### 1.4 快速上手：第一个 Gradle 项目
- `gradle init` 初体验
- 项目类型选择（application / library / 基本项目）
- 选择 DSL（Groovy / Kotlin）
- 选择测试框架（JUnit / TestNG 等）
- 生成的项目结构解读
- 使用 `gradle tasks` 查看可用任务
- 运行构建、测试、`gradle run`

---

## 二、构建脚本核心

### 2.1 构建脚本基础
- `settings.gradle(.kts)` 与 `build.gradle(.kts)` 的作用
- `Project` 对象与 `Settings` 对象
- Groovy DSL 基础语法回顾（可选，若不熟）
- Kotlin DSL 基础（推荐，尤其对 Kotlin 开发者）
- 脚本中的 API 本质：委托、闭包/扩展函数

### 2.2 Task（任务）
- 任务定义：`task` / `tasks.register` 与 `tasks.create`
- 延迟配置（`register`） vs 立即创建（`create`）
- 任务行为：`doLast` / `doFirst`
- 任务类型：`Copy`、`Delete`、`Exec`、`Zip` 等内置类型
- 自定义任务类（继承 `DefaultTask`）
- 任务的输入与输出（`@Input`, `@OutputFile` 等）
- 增量构建与 UP-TO-DATE 原理
- 任务依赖：`dependsOn`、`mustRunAfter`、`shouldRunAfter`、`finalizedBy`
- 任务配置避免（Task Configuration Avoidance）

### 2.3 插件（Plugins）
- 插件的作用
- 插件分类：
  - 核心插件（例如 `java`、`application`）
  - 社区插件（通过 Gradle Plugin Portal 引入）
  - 脚本插件（`apply from`）
- 插件引入方式：
  - `plugins { id("...") }` DSL（推荐）
  - 旧式 `apply plugin`
- 常用插件：
  - `java` / `java-library`
  - `application`
  - `groovy` / `kotlin`
  - `maven-publish`
  - `idea` / `eclipse`

### 2.4 依赖管理
- 依赖图谱的概念
- 依赖声明：`implementation`, `api`, `compileOnly`, `runtimeOnly`, `testImplementation` 等配置
- `java-library` 插件中的 `api` 与 `implementation` 区别
- 仓库配置：`repositories { mavenCentral(); google(); mavenLocal() }`
- 声明依赖：模块坐标（group:artifact:version）
- 动态版本与快照版本
- 依赖解析与冲突解决策略
- `dependencies` 锁与版本锁定（`dependencyLocking`）
- 版本目录（Version Catalogs）：`libs.versions.toml` 的使用
- 强制版本与排除传递性依赖
- BOM 导入（如 Spring Boot、Jackson BOM）
- 查看依赖树：`gradle dependencies`

---

## 三、Gradle 构建生命周期

### 3.1 三大阶段
- **Initialization 阶段**：解析 `settings.gradle`，创建项目层级
- **Configuration 阶段**：配置所有 `Project` 对象，执行构建脚本中的配置代码
- **Execution 阶段**：执行用户请求的任务

### 3.2 生命周期监听
- `gradle.buildStarted` / `settingsEvaluated` / `projectsLoaded`
- `gradle.projectsEvaluated`
- `gradle.taskGraph.whenReady`
- `listener` 场景：构建前检查、参数校验、指标收集

### 3.3 配置延迟与避免
- 尽早使用 `tasks.register` 代替 `tasks.create`
- 仅在需要时解析 Provider
- 使用 `gradle.taskGraph.whenReady` 动态决定任务依赖

---

## 四、多项目构建

### 4.1 多项目结构
- 根项目与子模块
- `settings.gradle` 中的 `include` / `includeBuild`
- 项目路径与物理目录的关系
- 多项目示例：`app`、`lib`、`common`

### 4.2 子项目配置
- 根脚本中 `allprojects` 与 `subprojects` 的使用
- 更优实践：使用 `configure(subprojects) { ... }` 或约定插件
- 跨项目任务依赖：`evaluationDependsOn` 的注意点

### 4.3 组合构建（Composite Builds）
- 场景：多来源项目联调、替换依赖为本地项目
- `includeBuild` 声明
- 依赖替换：`substitute(module("...")).using(project("..."))`

### 4.4 构建逻辑复用
- 传统方式：`buildSrc`
- 现代推荐：约定插件（Convention Plugins），写在 `buildSrc` 或独立项目
- Precompiled script plugins（`*.gradle.kts` 在 `buildSrc` 中）
- 引入外部共享约定（发布为二进制插件）

---

## 五、深入 Task 与增量构建

### 5.1 输入输出模型
- `@Input`、`@InputFile`、`@InputFiles`
- `@OutputFile`、`@OutputFiles`、`@OutputDirectory`
- `@Nested` 处理复杂输入
- `@Internal` 标记非增量属性
- 开发任务时遵守增量原则

### 5.2 Provider API
- `Provider<T>` 与 `Property<T>`
- 延迟解析：`map`、`flatMap`、`zip`
- 与命令行选项对接：`@Option`
- 与扩展（Extensions）和插件的交互

### 5.3 Worker API（并发执行）
- 工作隔离与类加载
- `WorkerExecutor` 的使用
- 并行任务执行的优势与适用场景

---

## 六、编写自定义插件

### 6.1 插件开发基础
- 实现 `Plugin<Project>` 接口
- 插件 ID 与 `META-INF/gradle-plugins`
- 简单插件注册在 `buildSrc` 中
- 使用扩展（Extension）让用户配置插件

### 6.2 独立插件项目
- 多模块插件工程
- 插件测试：
  - 手动测试
  - 自动化功能测试（`gradleTestKit`）
- 发布插件到 Gradle Plugin Portal 或私有仓库

### 6.3 高级扩展 DSL
- 嵌套扩展对象
- 使用 `NamedDomainObjectContainer` 创建容器型配置（如 sourceSets）
- 与 `Provider` 配合，实现懒加载 DSL

---

## 七、测试、质量与发布

### 7.1 测试集成
- Java 测试：`test` 任务与 JUnit 平台配置
- 并行测试执行：`maxParallelForks`
- 测试分组、过滤与体系报告
- 测试日志与 TestKit

### 7.2 代码质量
- 集成 Checkstyle / PMD / SpotBugs
- 集成 JaCoCo 代码覆盖率
- 将质量任务加入构建生命周期

### 7.3 发布与部署
- `maven-publish` 插件使用
- 发布到 Maven Local / 私有 Nexus / Maven Central
- 签名与 PGP 配置（`signing` 插件）
- 使用 `project.version` 与 CI 版本号管理

---

## 八、性能优化与最佳实践

### 8.1 构建扫描（Build Scan）
- 生成构建扫描报告：`--scan`
- 分析性能瓶颈：任务耗时、依赖解析、配置时间

### 8.2 构建缓存
- 本地构建缓存：`org.gradle.caching=true`
- 远程构建缓存（与 CI 共用）
- 哪些任务适合缓存

### 8.3 配置缓存（Configuration Cache）
- 概念：跳过配置阶段直接执行
- 启用方式与要求
- 常见问题排查（非稳定外部输入、绝对路径等）

### 8.4 其他加速技巧
- 并行构建：`org.gradle.parallel=true`
- 按需配置：`org.gradle.configureondemand=true`
- 使用 `-x` 跳过不需要的任务
- 守护进程 JVM 参数调优
- 依赖解析策略优化（减少动态版本）

### 8.5 常见问题与排错
- `--debug` / `--info` / `--stacktrace` 的用途
- 依赖冲突解决方法：`failOnVersionConflict`、`resolutionStrategy`
- 类冲突与版本对齐（`platform`）
- 锁版本以增强可重复性

---

## 九、与工具和 CI/CD 集成

### 9.1 IDE 集成
- IntelliJ IDEA 中的 Gradle 集成原理
- 原生 BSP 与 Gradle Build Server
- 加速 IDE 同步的技巧
- 常见问题：导入慢、索引占用、无法识别 DSL

### 9.2 CI/CD 集成
- GitHub Actions / GitLab CI / Jenkins 中使用 Gradle
- 缓存 Gradle 依赖与 Wrapper（actions/setup-gradle 等）
- 构建缓存复用
- 并行执行与矩阵构建
- 生成报告 & 构建扫描发布到 CI 同一页面

### 9.3 Gradle 与 Kotlin Multiplatform
- Kotlin 多平台项目 Gradle 配置概览
- 源集结构与目标平台
- 多平台依赖声明方式

### 9.4 Gradle 与 Android
- Android Gradle Plugin (AGP) 基础
- 版本兼容性
- 构建变体与风味管理
- Android 专门依赖配置

---

## 十、综合实战建议

- 实战 1：创建标准 Java 多模块项目（`app`、`domain`、`infrastructure`）
- 实战 2：编写一个代码生成插件，根据 YAML 生成实体类
- 实战 3：搭建发布流水线，将库发布到 Maven Central
- 实战 4：将 Maven 存量项目迁移到 Gradle
- 实战 5：为内部团队编写可复用的约定插件，统一构建配置

---

## 学习路线建议（按阶段）
1. **入门**：1-2 天过基础概念与快速上手，了解 Wrapper 和生命周期
2. **核心**：重点吃透 Task、插件机制与依赖管理（耗时最长）
3. **多项目**：学习多模块、复合构建、`buildSrc` 与约定插件
4. **高级**：编写自定义插件、深入 Provider 与增量构建、性能调优
5. **工程化**：CI/CD 集成、构建扫描、发布管理
6. **扩展**：Android / Kotlin Multiplatform 等垂直领域

建议在每个阶段都配合代码练习，将知识点落到实际项目中。