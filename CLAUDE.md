# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RuoYi-Vue-Fast is a Spring Boot 2.5.15 + Vue.js enterprise management framework (version 3.9.1). It is a **single-module** Java backend (unlike the multi-module RuoYi variant) providing RBAC, code generation, scheduled tasks, and system monitoring via a RESTful API.

## Build & Run Commands

```bash
# Build (skip tests)
mvn clean package -Dmaven.test.skip=true

# Run via Maven
mvn spring-boot:run

# Run packaged JAR
java -jar target/ruoyi.jar

# Windows scripts
bin/package.bat   # Build
bin/clean.bat     # Clean

# Linux/Mac scripts
ry.sh start|stop|restart|status
```

There are no unit or integration tests in this repository despite Spring Boot test being on the classpath.

API docs available at `http://localhost:8080/swagger-ui.html` when running.

## Architecture

### Package Structure

```
com.ruoyi
├── RuoYiApplication.java        # Entry point (@SpringBootApplication, excludes DataSourceAutoConfiguration)
├── common/                      # Cross-cutting utilities
│   ├── constant/                # Global constants
│   ├── core/                    # Text/domain utilities
│   ├── enums/                   # Enums (HTTP status, user status)
│   ├── exception/               # Custom exception hierarchy
│   ├── filter/                  # Servlet filters (XSS, CORS)
│   └── utils/                   # SecurityUtils, StringUtils, IP utils, etc.
├── framework/                   # Infrastructure layer
│   ├── aspectj/                 # AOP: @Log, @RateLimiter, @DataScope, @DataSource
│   ├── config/                  # 15 Spring @Configuration beans
│   ├── datasource/              # Dynamic multi-datasource routing
│   ├── redis/                   # RedisCache utility wrapper
│   ├── security/                # JWT filter, Spring Security config, UserDetailsService
│   ├── task/                    # Async task manager
│   └── web/                     # BaseController, AjaxResult, GlobalExceptionHandler
└── project/                     # Business domain modules
    ├── system/                  # Users, roles, menus, depts, config, dict, notice, etc.
    ├── monitor/                 # Operation logs, login logs, scheduled job logs
    └── tool/gen/                # Code generation engine (Velocity templates)
```

### Request Lifecycle

1. `AuthenticationTokenFilter` validates JWT from `Authorization` header → populates `SecurityContext`
2. Controller (annotated with `@PreAuthorize("@ss.hasPermi('...')")`) receives request
3. AOP aspects fire: `LogAspect` (audit logging), `RateLimiterAspect`, `DataScopeAspect` (row-level security)
4. Controller → Service interface → MyBatis Mapper → MySQL via Druid pool
5. Response wrapped in `AjaxResult` → `{code, msg, data}`

### Security Model

- **Authentication**: JWT stored in Redis with 30-minute TTL (auto-refreshed if within 20 min of expiry)
- **Authorization**: Spring Security + `@PreAuthorize` with custom `PermissionService` (`@ss`)
- **Data scope**: `@DataScope` AOP limits query results by user department/role
- **Password hashing**: BCrypt

### Key AOP Annotations

| Annotation | Location | Effect |
|---|---|---|
| `@Log(title, businessType)` | Controllers | Records operation to `sys_oper_log` |
| `@RateLimiter` | Controllers | Redis-backed request throttling |
| `@DataScope` | Service methods | Injects SQL fragment for department/user filtering |
| `@DataSource` | Service methods | Routes to master or slave datasource |

### Response Conventions

- **Success**: `AjaxResult.success(data)` → `{code: 200, msg: "操作成功", data: ...}`
- **Error**: `AjaxResult.error(msg)` → `{code: 500, msg: "...", data: null}`
- **Paginated lists**: `getDataTable(list)` → `TableDataInfo {code, msg, rows, total}`
- **Soft deletes**: `del_flag` column (`"0"` = exists, `"2"` = deleted)

### Configuration Files

| File | Purpose |
|---|---|
| `src/main/resources/application.yml` | App name, port (8080), file upload paths, security settings |
| `src/main/resources/application-druid.yml` | Druid connection pool, master/slave DB URLs |
| `src/main/resources/logback.xml` | Log rotation, separate error log file |
| `src/main/resources/mybatis/mybatis-config.xml` | MyBatis global settings |
| `src/main/resources/mybatis/**/*Mapper.xml` | SQL mapper files |

### Code Generation

The `tool/gen` module generates CRUD scaffolding (Java controller/service/mapper, Vue pages, SQL) from database table metadata using Velocity templates in `src/main/resources/vm/`.
