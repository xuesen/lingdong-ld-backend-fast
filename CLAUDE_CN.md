# CLAUDE.md（中文版）

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指引。

## 项目概述

RuoYi-Vue-Fast 是基于 Spring Boot 2.5.15 + Vue.js 的企业级后台管理框架（版本 3.9.1）。与多模块版本的若依不同，本项目是**单模块**的 Java 后端，提供 RBAC 权限管理、代码生成、定时任务和系统监控等功能，通过 RESTful API 对外服务。

## 构建与运行命令

```bash
# 构建（跳过测试）
mvn clean package -Dmaven.test.skip=true

# 通过 Maven 运行
mvn spring-boot:run

# 运行打包后的 JAR
java -jar target/ruoyi.jar

# Windows 脚本
bin/package.bat   # 构建
bin/clean.bat     # 清理

# Linux/Mac 脚本
ry.sh start|stop|restart|status
```

本仓库没有单元测试或集成测试（尽管 classpath 中包含了 Spring Boot Test 依赖）。

运行后可通过 `http://localhost:8080/swagger-ui.html` 访问 API 文档。

## 架构说明

### 包结构

```
com.ruoyi
├── RuoYiApplication.java        # 启动入口（@SpringBootApplication，排除 DataSourceAutoConfiguration）
├── common/                      # 公共工具层
│   ├── constant/                # 全局常量
│   ├── core/                    # 文本/领域工具类
│   ├── enums/                   # 枚举（HTTP 状态、用户状态）
│   ├── exception/               # 自定义异常体系
│   ├── filter/                  # Servlet 过滤器（XSS、CORS）
│   └── utils/                   # SecurityUtils、StringUtils、IP 工具等
├── framework/                   # 基础设施层
│   ├── aspectj/                 # AOP 切面：@Log、@RateLimiter、@DataScope、@DataSource
│   ├── config/                  # 15 个 Spring @Configuration 配置类
│   ├── datasource/              # 动态多数据源路由
│   ├── redis/                   # RedisCache 工具封装
│   ├── security/                # JWT 过滤器、Spring Security 配置、UserDetailsService
│   ├── task/                    # 异步任务管理
│   └── web/                     # BaseController、AjaxResult、全局异常处理器
└── project/                     # 业务领域模块
    ├── system/                  # 用户、角色、菜单、部门、参数配置、字典、通知公告等
    ├── monitor/                 # 操作日志、登录日志、定时任务日志
    └── tool/gen/                # 代码生成引擎（Velocity 模板）
```

### 请求生命周期

1. `AuthenticationTokenFilter` 从 `Authorization` 请求头验证 JWT → 填充 `SecurityContext`
2. Controller（通过 `@PreAuthorize("@ss.hasPermi('...')")` 注解鉴权）接收请求
3. AOP 切面触发：`LogAspect`（操作审计）、`RateLimiterAspect`（限流）、`DataScopeAspect`（行级数据权限）
4. Controller → Service 接口 → MyBatis Mapper → MySQL（通过 Druid 连接池）
5. 响应封装为 `AjaxResult` → `{code, msg, data}`

### 安全模型

- **认证**：JWT 令牌存储于 Redis，TTL 30 分钟（距过期 20 分钟内自动刷新）
- **授权**：Spring Security + `@PreAuthorize`，使用自定义 `PermissionService`（Bean 名称 `@ss`）
- **数据权限**：`@DataScope` AOP 切面根据用户部门/角色限制查询结果
- **密码加密**：BCrypt

### 核心 AOP 注解

| 注解                          | 使用位置       | 作用                     |
|-----------------------------|------------|------------------------|
| `@Log(title, businessType)` | Controller | 记录操作到 `sys_oper_log` 表 |
| `@RateLimiter`              | Controller | 基于 Redis 的请求限流         |
| `@DataScope`                | Service 方法 | 注入 SQL 片段实现部门/用户级数据过滤  |
| `@DataSource`               | Service 方法 | 动态切换主/从数据源             |

### 响应规范

- **成功**：`AjaxResult.success(data)` → `{code: 200, msg: "操作成功", data: ...}`
- **失败**：`AjaxResult.error(msg)` → `{code: 500, msg: "...", data: null}`
- **分页列表**：`getDataTable(list)` → `TableDataInfo {code, msg, rows, total}`
- **逻辑删除**：`del_flag` 字段（`"0"` = 存在，`"2"` = 已删除）

### 配置文件

| 文件                                              | 用途                        |
|-------------------------------------------------|---------------------------|
| `src/main/resources/application.yml`            | 应用名称、端口（8080）、文件上传路径、安全配置 |
| `src/main/resources/application-druid.yml`      | Druid 连接池配置、主从数据库 URL     |
| `src/main/resources/logback.xml`                | 日志轮转、独立错误日志文件             |
| `src/main/resources/mybatis/mybatis-config.xml` | MyBatis 全局设置              |
| `src/main/resources/mybatis/**/*Mapper.xml`     | SQL 映射文件                  |

### 代码生成

`tool/gen` 模块通过读取数据库表元数据，使用 `src/main/resources/vm/` 中的 Velocity 模板自动生成 CRUD 代码（Java Controller/Service/Mapper、Vue 页面、SQL）。

