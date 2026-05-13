
面向熟悉 Spring 全家桶开发者的快速入门指南，全程对比 Spring 机制，帮你秒懂 Guice 的运行逻辑

## 前言

如果你已经深度理解 Spring 的 IOC、AOP、Web 生态，那么 Guice 对你来说会非常好理解：**Guice 就是一个「轻量版的 Spring Core」，专注于依赖注入，没有 Spring 全家桶的那些周边功能，但核心的 DI、AOP、Web 集成逻辑和 Spring 几乎同源**。

Guice 本身只是一个 DI 框架，没有内置 MVC、ORM 等功能，因此我们会搭配原生 Servlet 实现 Web 层，同时完整覆盖你关心的所有工程化场景：

- Servlet 定义与请求流转
    
- Controller/Service/DAO 分层依赖注入
    
- 模块化配置（Module）
    
- AOP 横切逻辑实现
    
- 完整的 Tomcat 环境可运行案例
    

---

## 1. 项目整体结构

我们采用标准的 Maven Web 工程结构，完全符合 Java 工程化规范，包划分清晰：

```Plain
guice-web-demo/
├── pom.xml                     # Maven依赖配置
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           ├── config                  # 配置模块
        │           │   ├── AppModule.java     # 主模块，组装所有子模块
        │           │   ├── DataModule.java    # 数据层配置
        │           │   ├── ServiceModule.java # 业务层配置
        │           │   ├── WebModule.java     # Web层配置
        │           │   └── AopModule.java     # AOP配置
        │           ├── servlet                # Servlet层
        │           │   └── UserServlet.java   # 用户相关请求的Servlet
        │           ├── controller             # 控制层
        │           │   └── UserController.java
        │           ├── service                # 业务层
        │           │   ├── UserService.java   # 接口
        │           │   └── impl
        │           │       └── UserServiceImpl.java
        │           ├── dao                    # 数据访问层
        │           │   ├── UserDao.java       # 接口
        │           │   └── impl
        │           │       └── UserDaoImpl.java
        │           ├── aop                    # AOP相关
        │           │   ├── Loggable.java      # 自定义日志注解
        │           │   └── LoggingInterceptor.java # 日志拦截器
        │           ├── entity                 # 实体类
        │           │   └── User.java
        │           └── GuiceWebContextListener.java # Guice上下文初始化
        └── webapp
            └── WEB-INF
                └── web.xml                   # Web应用配置
```

---

## 2. 环境依赖配置

### 版本说明

Guice 的版本和 Servlet API 版本强绑定：

- **Tomcat 9 及以下**：使用`javax.servlet`，对应 Guice 6.x 版本
    
- **Tomcat 10 及以上**：使用`jakarta.servlet`，对应 Guice 7.x 版本
    

本教程以 Tomcat 9 + Guice 6.0.0 为例，如果你用高版本 Tomcat，只需要把 Guice 版本换成 7.x 即可。

### Maven pom.xml

完整依赖配置，包含 Guice 核心、Servlet 扩展、JSON 处理、日志框架：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>guice-web-demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <guice.version>6.0.0</guice.version>
        <jackson.version>2.15.2</jackson.version>
        <slf4j.version>2.0.7</slf4j.version>
    </properties>

    <dependencies>
        <!-- Guice核心依赖 -->
        <dependency>
            <groupId>com.google.inject</groupId>
            <artifactId>guice</artifactId>
            <version>${guice.version}</version>
        </dependency>
        <!-- Guice Servlet集成扩展 -->
        <dependency>
            <groupId>com.google.inject.extensions</groupId>
            <artifactId>guice-servlet</artifactId>
            <version>${guice.version}</version>
        </dependency>

        <!-- Servlet API -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>4.0.1</version>
            <scope>provided</scope>
        </dependency>

        <!-- JSON处理 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- 日志依赖 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.4.8</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <port>8080</port>
                    <path>/</path>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 3. Web 核心配置

### web.xml

对应 Spring 项目中的`web.xml`，这里我们配置 Guice 的核心组件：

- `GuiceServletContextListener`：对应 Spring 的`ContextLoaderListener`，负责初始化 Guice 的 IOC 容器（Injector）
    
- `GuiceFilter`：对应 Spring 的`DelegatingFilterProxy`，拦截所有请求，交给 Guice 处理，同时管理请求作用域
    

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!-- Guice上下文初始化监听器 -->
    <listener>
        <listener-class>com.example.GuiceWebContextListener</listener-class>
    </listener>

    <!-- Guice核心过滤器，拦截所有请求 -->
    <filter>
        <filter-name>guiceFilter</filter-name>
        <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>guiceFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

</web-app>
```

### Guice 上下文初始化

`GuiceWebContextListener`继承自 Guice 的`GuiceServletContextListener`，负责创建 Guice 的核心容器`Injector`（对应 Spring 的`ApplicationContext`）：

```java
package com.example;

import com.example.config.AppModule;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.servlet.GuiceServletContextListener;

public class GuiceWebContextListener extends GuiceServletContextListener {
    @Override
    protected Injector getInjector() {
        // 创建Guice注入器，加载所有模块配置
        // 对应Spring中创建ApplicationContext，加载所有@Configuration配置类
        return Guice.createInjector(new AppModule());
    }
}
```

---

## 4. 模块化配置（Module）

Guice 中的`Module`对应 Spring 中的`@Configuration`配置类，用来定义接口与实现的绑定规则。我们按照分层拆分模块，实现工程化的配置隔离。

### 主模块 AppModule

主模块负责组装所有子模块，对应 Spring 中的`@Import`，导入其他配置类：

```java
package com.example.config;

import com.google.inject.AbstractModule;

public class AppModule extends AbstractModule {
    @Override
    protected void configure() {
        // 安装子模块，按层拆分配置
        install(new DataModule());    // 数据层配置
        install(new ServiceModule()); // 业务层配置
        install(new WebModule());     // Web层配置
        install(new AopModule());     // AOP配置
    }
}
```

### 数据层模块 DataModule

绑定 DAO 层的接口与实现：

```java
package com.example.config;

import com.example.dao.UserDao;
import com.example.dao.impl.UserDaoImpl;
import com.google.inject.AbstractModule;

public class DataModule extends AbstractModule {
    @Override
    protected void configure() {
        // 绑定接口到实现，对应Spring中的 @Bean public UserDao userDao() { return new UserDaoImpl(); }
        bind(UserDao.class).to(UserDaoImpl.class);
        //...绑定其他DAO
    }
}
```

### 业务层模块 ServiceModule

绑定 Service 层的接口与实现，同时声明单例作用域：

```java
package com.example.config;

import com.example.service.UserService;
import com.example.service.impl.UserServiceImpl;
import com.google.inject.AbstractModule;
import com.google.inject.Singleton;

public class ServiceModule extends AbstractModule {
    @Override
    protected void configure() {
        // 绑定接口到实现，并且声明为单例（整个应用只有一个实例）
        // 对应Spring中的 @Scope("singleton") 或者 @Singleton
        bind(UserService.class).to(UserServiceImpl.class).in(Singleton.class);
        //...绑定其他 service
    }
}
```

### Web 层模块 WebModule

继承自`ServletModule`，用来配置 Servlet、Filter 的 URL 映射，这是 Guice Servlet 扩展的核心配置类：

```java
package com.example.config;

import com.example.servlet.UserServlet;
import com.google.inject.servlet.ServletModule;

public class WebModule extends ServletModule {
    @Override
    protected void configureServlets() {
        // 配置Servlet映射：将 /users/* 路径的请求交给 UserServlet 处理
        // 对应SpringMVC中配置Servlet的url-pattern
        serve("/users/*").with(UserServlet.class);
    }
}
```

---

## 5. 分层实现与依赖注入

我们按照标准的 DAO -> Service -> Controller -> Servlet 分层，完整展示 Guice 的依赖注入链路，每个部分的逻辑都和 Spring 完全对齐。

### 实体类 User

简单的用户实体：

```java
package com.example.entity;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private String email;
}
```

### DAO 层

#### 接口 UserDao

```java
package com.example.dao;

import com.example.entity.User;

public interface UserDao {
    User findById(Long id);
}
```

#### 实现 UserDaoImpl

这里我们用内存模拟数据，不需要真实数据库，方便你直接跑通：

```java
package com.example.dao.impl;

import com.example.dao.UserDao;
import com.example.entity.User;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class UserDaoImpl implements UserDao {
    // 模拟数据库
    private static final Map<Long, User> USER_DB = new ConcurrentHashMap<>();

    static {
        USER_DB.put(1L, new User(1L, "张三", "zhangsan@example.com"));
        USER_DB.put(2L, new User(2L, "李四", "lisi@example.com"));
    }

    @Override
    public User findById(Long id) {
        return USER_DB.get(id);
    }
}
```

### Service 层

#### 接口 UserService

```java
package com.example.service;

import com.example.entity.User;

public interface UserService {
    User getUserById(Long id);
}
```

#### 实现 UserServiceImpl

这里我们通过**构造器注入**DAO，这是 Guice 推荐的注入方式，和 Spring 现在推荐的构造器注入完全一致：

```java
package com.example.service.impl;

import com.example.aop.Loggable;
import com.example.dao.UserDao;
import com.example.entity.User;
import com.example.service.UserService;
import com.google.inject.Inject;

public class UserServiceImpl implements UserService {
    private final UserDao userDao;

    // @Inject 对应Spring的 @Autowired，标记这是一个注入点
    // Guice会自动创建UserDao的实例，注入到构造函数中
    @Inject
    public UserServiceImpl(UserDao userDao) {
        this.userDao = userDao;
    }

    // @Loggable 自定义注解，标记这个方法需要被AOP日志拦截
    @Override
    @Loggable
    public User getUserById(Long id) {
        // 模拟业务处理
        return userDao.findById(id);
    }
}
```

### Controller 层

UserController 负责处理请求的业务编排，注入 Service：

```java
package com.example.controller;

import com.example.entity.User;
import com.example.service.UserService;
import com.google.inject.Inject;

public class UserController {
    private final UserService userService;

    @Inject
    public UserController(UserService userService) {
        this.userService = userService;
    }

    public User getUserById(Long id) {
        return userService.getUserById(id);
    }
}
```

### Servlet 层

UserServlet 负责接收 HTTP 请求，注入 Controller，完成请求流转：

```java
package com.example.servlet;

import com.example.controller.UserController;
import com.example.entity.User;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.inject.Inject;
import com.google.inject.Singleton;

import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Singleton
public class UserServlet extends HttpServlet {
    private final UserController userController;
    private final ObjectMapper objectMapper;

    // 注入Controller和JSON工具
    @Inject
    public UserServlet(UserController userController, ObjectMapper objectMapper) {
        this.userController = userController;
        this.objectMapper = objectMapper;
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // 1. 获取请求参数
        String idStr = req.getParameter("id");
        Long id = Long.parseLong(idStr);

        // 2. 调用Controller处理请求
        User user = userController.getUserById(id);

        // 3. 返回JSON响应
        resp.setContentType("application/json;charset=UTF-8");
        objectMapper.writeValue(resp.getWriter(), user);
    }
}
```

---

## 6. AOP 面向切面编程

Guice 的 AOP 和 Spring 的 AOP 几乎同源，都是基于 AOP Alliance 的`MethodInterceptor`，只是配置方式略有不同。我们实现一个通用的日志拦截器，用来打印方法的入参、出参、耗时。

### 自定义注解 @Loggable

用来标记需要被拦截的方法，对应 Spring 中的自定义注解切点：

```java
package com.example.aop;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

### 日志拦截器 LoggingInterceptor

实现`MethodInterceptor`，对应 Spring 中的`@Around`通知：

```java
package com.example.aop;

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Arrays;

public class LoggingInterceptor implements MethodInterceptor {
    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 前置通知：打印方法信息、入参
        String methodName = invocation.getMethod().getDeclaringClass().getName() + "." + invocation.getMethod().getName();
        Object[] args = invocation.getArguments();
        logger.info("开始调用方法: {}, 参数: {}", methodName, Arrays.toString(args));

        long start = System.currentTimeMillis();
        try {
            // 执行原方法
            Object result = invocation.proceed();

            // 返回通知：打印返回值、耗时
            long cost = System.currentTimeMillis() - start;
            logger.info("方法调用完成: {}, 耗时: {}ms, 返回值: {}", methodName, cost, result);
            return result;
        } catch (Throwable t) {
            // 异常通知：打印异常信息
            long cost = System.currentTimeMillis() - start;
            logger.error("方法调用异常: {}, 耗时: {}ms", methodName, cost, t);
            throw t;
        }
    }
}
```

### AOP 配置模块 AopModule

在模块中绑定拦截器，定义切点规则，对应 Spring 中的`@Pointcut`：

```java
package com.example.config;

import com.example.aop.Loggable;
import com.example.aop.LoggingInterceptor;
import com.google.inject.AbstractModule;
import com.google.inject.matcher.Matchers;

public class AopModule extends AbstractModule {
    @Override
    protected void configure() {
        // 绑定拦截器：
        // 第一个参数：匹配所有类（对应Spring的 execution(* *..*()) 的类匹配）
        // 第二个参数：匹配带有 @Loggable 注解的方法（对应Spring的 @annotation(Loggable)）
        // 第三个参数：拦截器实例
        bindInterceptor(
                Matchers.any(),
                Matchers.annotatedWith(Loggable.class),
                new LoggingInterceptor()
        );
    }
}
```

> Guice AOP 的限制和 Spring JDK 动态代理完全一致：
> 
> 1. 只能拦截**公共、非 final**的方法
>     
> 2. 只有被 Guice 容器管理的对象才能被拦截，自己 new 的对象无法生效
>     
> 3. 无法拦截类内部的方法调用（比如 this. 方法 ()）
>     

---

## 7. 运行与测试

### 启动项目

你可以直接使用 Maven 插件启动 Tomcat：

```bash
mvn tomcat7:run
```

或者打包成 war 包，放到 Tomcat 的 webapps 目录下启动。

### 测试接口

访问接口：`http://localhost:8080/users?id=1`

你会得到如下 JSON 响应：

```json
{"id":1,"name":"张三","email":"zhangsan@example.com"}
```

同时查看日志，你会看到 AOP 拦截的输出：

```Plain
INFO  com.example.aop.LoggingInterceptor - 开始调用方法: com.example.service.impl.UserServiceImpl.getUserById, 参数: [1]
INFO  com.example.aop.LoggingInterceptor - 方法调用完成: com.example.service.impl.UserServiceImpl.getUserById, 耗时: 1ms, 返回值: User(id=1, name=张三, email=zhangsan@example.com)
```

这说明整个链路完全跑通了： 请求 -> GuiceFilter -> UserServlet -> UserController -> UserService -> UserDao 同时 AOP 也成功拦截了 Service 层的方法。

---

## 8. 核心运行原理（对比 Spring）

对于熟悉 Spring 的你来说，整个运行流程你可以完全对应上：

|   |   |   |
|---|---|---|
|阶段|Guice 的处理流程|对应 Spring 的流程|
|容器初始化|Tomcat 启动时，`GuiceServletContextListener`被触发，调用`Guice.createInjector()`，解析所有 Module 的绑定规则，生成 Injector 容器|Tomcat 启动时，`ContextLoaderListener`被触发，调用`new ClassPathXmlApplicationContext()`，解析所有 @Configuration 配置，生成 ApplicationContext 容器|
|请求进入|请求先经过`GuiceFilter`，初始化请求作用域，把 Request、Response 绑定到当前线程，然后交给 Guice 的 Servlet 处理链|请求先经过`DelegatingFilterProxy`，初始化请求作用域，然后交给 SpringMVC 的 DispatcherServlet|
|依赖注入|Guice 根据 URL 找到对应的 Servlet，然后递归解析 Servlet 的所有依赖：Servlet 依赖 Controller，Controller 依赖 Service，Service 依赖 DAO，Guice 自动创建所有实例并注入|Spring 找到对应的 Controller，然后自动注入 Controller 依赖的 Service、DAO|
|AOP 拦截|当调用被拦截的方法时，Guice 创建的代理对象会先执行拦截器的逻辑，然后再执行原方法|Spring 的代理对象执行拦截器逻辑，然后执行原方法|

整个流程和 Spring Web 的运行逻辑几乎一模一样，只是 Guice 更轻量，没有那么多的自动配置、组件扫描等黑魔法，所有的绑定都是显式声明的，更透明。

---

## 9. Guice 的生态与搭配框架

Guice 本身只是 DI 框架，没有 Spring 全家桶的那些功能，如果你需要更完整的 Web 开发能力，可以搭配这些框架：

|   |   |   |
|---|---|---|
|场景|搭配框架|对应 Spring 的组件|
|RESTful API|Jersey + Guice / RestEasy + Guice|SpringMVC / Spring WebFlux|
|ORM 持久层|MyBatis-Guice / Guice-Persist(JPA)|Spring MyBatis / Spring Data JPA|
|安全框架|Shiro + Guice|Spring Security|
|事务管理|MyBatis-Guice 的 @Transactional|Spring 的 @Transactional|
|配置管理|Owner 框架|Spring Boot Configuration|

这些组合可以让你用 Guice 搭建出和 Spring Boot 一样完整的工程化项目，但是体积更小、启动更快，非常适合轻量服务或者对启动速度有要求的场景。

---

## 10. Guice vs Spring 核心差异

最后给你总结一下核心差异，帮你快速建立认知：

1. **定位不同**：Guice 专注于 DI，是一个轻量的工具；Spring 是全家桶，提供了完整的生态
    
2. **配置方式**：Guice 默认是**显式绑定**，所有的接口实现绑定都要手动在 Module 中声明，没有隐藏逻辑；Spring 默认是**隐式扫描**，自动发现 @Component 注解的类，更方便但有隐藏逻辑
    
3. **循环依赖**：Guice 不支持构造器注入的循环依赖，需要你自己用 Provider 打破；Spring 支持构造器循环依赖
    
4. **启动速度**：Guice 启动速度远快于 Spring，因为没有大量的自动配置、扫描逻辑
    
5. **体积**：Guice 核心只有几百 KB，Spring 核心要大很多
    

如果你需要一个轻量、透明、快速的 DI 框架，Guice 是非常好的选择；如果你需要完整的全家桶生态，那 Spring 还是更合适。