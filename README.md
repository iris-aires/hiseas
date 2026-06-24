# Hiseas Cloud

> 一套基于 **Spring Boot 3 + Spring Cloud 2023 + Spring Cloud Alibaba** 构建的在线课程平台微服务脚手架。
> 涵盖服务注册发现、配置中心、API 网关、统一认证鉴权、服务间调用、分布式事务、限流降级、链路追踪，并配套完整的 **Docker + Harbor + Kubernetes + GitLab CI/CD** 云原生交付流水线。

---

## 目录

- [一、项目简介](#一项目简介)
- [二、系统拓扑](#二系统拓扑)
- [三、技术栈](#三技术栈)
- [四、模块说明](#四模块说明)
- [五、整体架构与设计](#五整体架构与设计)
  - [5.1 认证鉴权设计（Sa-Token）](#51-认证鉴权设计sa-token)
  - [5.2 网关与路由](#52-网关与路由)
  - [5.3 服务间调用（OpenFeign）](#53-服务间调用openfeign)
  - [5.4 聚合服务与分布式事务（Seata）](#54-聚合服务与分布式事务seata)
  - [5.5 链路追踪（MDC traceId）](#55-链路追踪mdc-traceid)
  - [5.6 限流降级（Sentinel）](#56-限流降级sentinel)
  - [5.7 动态配置刷新](#57-动态配置刷新)
- [六、核心调用链路](#六核心调用链路)
- [七、数据库与缓存](#七数据库与缓存)
- [八、本地运行](#八本地运行)
- [九、配置说明](#九配置说明)
- [十、CI/CD 与云原生部署](#十cicd-与云原生部署)
- [十一、项目结构](#十一项目结构)
- [十二、效果截图](#十二效果截图)
- [十三、参考资料与许可证](#十三参考资料与许可证)

---

## 一、项目简介

`hiseas-cloud` 是一个面向「在线课程」业务场景的微服务工程样例。业务上包含 **用户、讲师、角色权限、积分、课程、课程评论** 等模块；技术上完整演示了一套生产可用的微服务基础设施如何协同工作：

- **原子微服务**：用户服务、课程服务，各自只负责本领域的 CRUD，独立数据库（库表隔离）。
- **聚合服务**：`mgmt-application` 不直接持有数据库，而是通过 Feign 编排多个原子服务，对外提供面向页面的聚合接口（如「课程详情 = 课程信息 + 讲师信息」）。
- **认证中心**：`iam` 服务统一处理登录、签发会话、维护「用户-角色」「角色-权限」缓存。
- **API 网关**：`gateway` 服务作为统一入口，负责动态路由、登录态校验、访问日志。
- **公共能力**：`common`（枚举/常量/工具/链路 ID 过滤器）、`sa-token`（以 Spring Boot 自动装配方式下发的鉴权能力）作为基础 Jar 被各服务依赖。

> 工程采用「**API 接口模块 + Service 实现模块**」分离的设计：每个原子服务都拆成 `*-api`（仅含 Feign 接口与 DTO，供其它服务依赖）和 `*-service`（真正的业务实现）。这样上游服务只依赖轻量的 `*-api` 包即可发起远程调用，避免引入实现细节。

---

## 二、系统拓扑

![拓扑图](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208001433721.png)

请求自上而下的大致流转：

```
客户端
  │  HTTP（携带 Access-Token）
  ▼
Istio Gateway API ──► hiseas-center-gateway（API 网关 :8431）
                          │  登录态校验（Sa-Token）+ 动态路由
        ┌─────────────────┼───────────────────────────┐
        ▼                 ▼                ▼            ▼
  iam-service        mgmt-application   course-service  user-service
   (认证中心:15000)   (聚合编排:10000)   (课程:9000)     (用户:8000)
        │                 │  OpenFeign                  ▲
        └──── Feign ───────┴── 调用原子服务 ─────────────┘
                          │
              Nacos / Redis / MySQL / Seata / Sentinel
```

---

## 三、技术栈

| 类别 | 选型 | 版本 / 说明 |
| :--- | :--- | :--- |
| 语言 / 运行时 | JDK | `21.0.5 LTS` |
| 基础框架 | Spring Boot | `3.2.1` |
| 微服务框架 | Spring Cloud | `2023.0.1` |
| 微服务套件 | Spring Cloud Alibaba | `2023.0.1.0` |
| 注册中心 / 配置中心 | Nacos | discovery + config |
| 服务网关 | Spring Cloud Gateway | 响应式（WebFlux） |
| 认证鉴权 | Sa-Token | `1.38.0`（JWT + Redis + alone-redis） |
| 服务调用 | OpenFeign + LoadBalancer | 内置 Feign 拦截器透传令牌 |
| 分布式事务 | Seata | `@GlobalTransactional` |
| 限流熔断 | Sentinel | dashboard 接入 |
| 持久层 | MyBatis-Plus | `3.5.6` |
| 数据库 | MySQL | `8.0.33` |
| 对象映射 | MapStruct | `1.5.5.Final` |
| 缓存 | Redis | 业务库 / Sa-Token 独立库隔离 |
| 工具 | Lombok / Hutool / Fastjson2 / Gson | — |
| 容器 / 镜像仓库 | Docker `27.4.0` / Harbor | — |
| 编排 | Kubernetes `v1.31.4` + Istio Gateway API | — |
| 制品仓库 / 代码质量 | Nexus / SonarQube | 私服 + 质量门禁 |
| CI/CD | GitLab CE `v16.9.9` | `.gitlab-ci.yml` 五阶段流水线 |

---

## 四、模块说明

| 模块 | 端口 | 类型 | 说明 |
| :--- | :---: | :--- | :--- |
| `hiseas-cloud` | - | 父 POM | 统一管理依赖版本、Nexus 私服地址、Sonar 配置、编译插件（Lombok/MapStruct 注解处理器）。 |
| `hiseas-common` | - | 公共库 | 全局枚举、常量、`RsaUtil` 工具、`MdcFilter` 链路 ID 过滤器等。 |
| `hiseas-sa-token` | - | 自动装配 Starter | Sa-Token 配置类，以 Spring Boot `AutoConfiguration` 方式被各服务自动加载；内含 `StpInterfaceImpl`（从 Redis 读取角色/权限）、全局异常处理器、拦截器。 |
| `hiseas-center-gateway` | `8431` | API 网关 | 统一入口，基于 Sa-Token 做登录态校验、动态路由、访问日志。 |
| `hiseas-center-iam` | `15000` | 认证鉴权中心 | 处理登录，登录成功后用 Sa-Token 建立会话，并初始化/维护「角色-权限」「用户-角色」缓存。 |
| `hiseas-mgmt-application` | `10000` | 聚合服务 | 不含数据库，通过 Feign 编排原子服务完成业务组装（课程详情、课程评论 + 积分）。 |
| `hiseas-center-course` | `9000` | 原子微服务 | 课程、课程评论模块的 CRUD，独立 `course_db`。拆分为 `-api` / `-service`。 |
| `hiseas-center-user` | `8000` | 原子微服务 | 用户、讲师、角色、权限、积分模块的 CRUD，独立用户库。拆分为 `-api` / `-service`。 |

---

## 五、整体架构与设计

### 5.1 认证鉴权设计（Sa-Token）

整套权限模型是 **RBAC（用户 → 角色 → 权限）**，并通过 Redis 缓存把鉴权数据「预热」起来，让每个微服务都能本地、快速地完成校验。

**缓存模型（两类数据，两个时机）：**

| 缓存 | Redis 结构 | 写入时机 | 写入方 |
| :--- | :--- | :--- | :--- |
| 角色-权限 | Hash：`role-permissions` → `{ 角色code: ["权限1","权限2"] }` | **认证中心启动时**（`@PostConstruct`） | `RolePermissionInitializer` |
| 用户-角色 | String：`user-roles-{userId}` → `["角色code", ...]`（TTL 24h） | **用户登录成功后** | `AuthServiceImpl` |

**登录流程（`hiseas-center-iam`）：**

1. `AuthController#login` 接收 `LoginDTO（username/password）`。
2. `AuthServiceImpl` 通过 Feign 调用 `user-service` 的 `/api/users/login` 校验账号密码（用户服务里做空值校验、密码比对、激活状态校验）。
3. 校验通过后调用 `StpUtil.login(userId, 附带 fullName)` 建立 Sa-Token 会话（JWT 风格，token 名 `Access-Token`）。
4. 再通过 Feign 拉取该用户的角色列表，写入 `user-roles-{userId}` 缓存。

**鉴权流程（任意业务服务）：**

- `hiseas-sa-token` 的 `StpInterfaceImpl` 实现了 Sa-Token 的 `StpInterface`：
  - `getRoleList(loginId)`：从 `user-roles-{loginId}` 读出角色。
  - `getPermissionList(loginId)`：用角色去 `role-permissions` Hash 里查出权限并去重汇总。
- 业务接口上用注解声明所需权限即可，例如：
  ```java
  @SaCheckPermission("user:query")
  @GetMapping("/{userId}")
  public ResponseEntity<?> getUserById(@PathVariable Long userId) { ... }
  ```
- 不需要鉴权的接口用 `@SaIgnore` 放行（如聚合服务的课程详情接口）。

**Redis 库隔离**：Sa-Token 使用 `alone-redis`（`database: 2`）单独存储会话信息，与业务缓存（`database: 1`）物理隔离，互不污染。

### 5.2 网关与路由

`hiseas-center-gateway` 基于响应式 Spring Cloud Gateway，配置了两层路由能力：

- **静态路由**：在 `bootstrap-dev.yml` 中显式声明，并用 `StripPrefix=1` 去掉前缀后转发：

  | 入口路径 | 目标服务 |
  | :--- | :--- |
  | `/mgmt-application/**` | `lb://mgmt-application` |
  | `/course/**` | `lb://course-service` |
  | `/user/**` | `lb://user-service` |
  | `/auth/**` | `lb://iam-service` |

- **动态路由**：开启 `gateway.discovery.locator.enabled=true`，可按 Nacos 中的服务名自动生成路由。

两个全局过滤器：

- `AuthenticationGlobalFilter`：先 `SaReactorSyncHolder.setContext(exchange)` 解决响应式上下文问题，放行登录接口，其余请求用 `StpUtil.isLogin()` 校验登录态，未登录返回统一 JSON 错误体。
- `LoggingGlobalFilter`（`@Order(0)`）：记录每个请求的方法、URI、响应码与耗时。

### 5.3 服务间调用（OpenFeign）

- 每个原子服务对外暴露 `*-api` 模块（Feign 接口 + DTO），上游只依赖该轻量包即可发起调用。
- Feign 客户端用 **可配置的服务名 / URL**，例如：
  ```java
  @FeignClient(
      contextId = "cn-com-hiseas-user-api-IUserApi",
      name = "${cn.com.hiseas.user.api.name:user-service}",
      path = "/api/users",
      url  = "${cn.com.hiseas.user.api:}")
  ```
  默认走 Nacos 注册名 `user-service`，也可通过配置覆盖为固定 URL（便于联调）。
- **令牌透传**：聚合服务里的 `TokenPassingInterceptor`（`feign.RequestInterceptor`）会从当前请求头取出 `Access-Token`，再塞进下游 Feign 请求头，保证调用链上的登录态不丢失。

主要 Feign 接口一览：

| 接口 | 路径 | 方法 | 归属服务 |
| :--- | :--- | :--- | :--- |
| `IUserApi` | `/api/users/login` | POST | user-service |
| `IRoleApi` | `/api/role`、`/api/role/{userId}/roles` | GET | user-service |
| `IPermissionApi` | `/api/permissions` | GET | user-service |
| `IInstructorApi` | `/api/instructors/{instructorId}` | GET | user-service |
| `IUserPointsApi` | `/api/userScore` | PUT | user-service |
| `ICourseApi` | `/api/course/{courseId}` | GET | course-service |
| `ICourseReviewApi` | `/api/courseReview` | POST | course-service |

### 5.4 聚合服务与分布式事务（Seata）

`hiseas-mgmt-application` 是典型的「BFF / 聚合编排层」：

- **课程详情聚合**（`CourseAggregationController`）：先调课程服务拿课程，再用课程里的 `instructorId` 调用户服务拿讲师，最后用 MapStruct 合并成 `CourseDetailVO` 返回。
- **课程评论 + 积分（跨服务一致性）**（`CourseReviewServiceImpl`）：一次「写评论」业务需要同时操作 **课程库（写评论）** 和 **用户库（加 5 积分）**，分属两个服务两个数据库。这里用 Seata 的 `@GlobalTransactional` 保证两步要么都成功、要么都回滚：
  ```java
  @GlobalTransactional
  public ResponseEntity<?> createCourseReview(CourseReviewRespDto dto) {
      courseReviewApi.createCourseReview(dto);          // 分支事务①：课程库
      userPointsApi.accumulateUserPoints(points);       // 分支事务②：用户库
      // 任一步异常必须抛出，否则 TM 感知不到，会误提交
  }
  ```

### 5.5 链路追踪（MDC traceId）

`hiseas-common` 的 `MdcFilter` 为每个进入服务的请求生成/透传 `traceId`，并放入 SLF4J 的 MDC，配合日志格式：

```
logging.pattern.level: '%5p [${spring.application.name},%mdc{traceId:-},%mdc{ts:-}]'
```

这样一条请求在多个服务里的日志都带同一个 `traceId`，便于全链路排查。`Access-Token` 透传 + traceId 透传共同构成了调用链上下文。

### 5.6 限流降级（Sentinel）

课程服务、聚合服务等都接入了 `spring-cloud-starter-alibaba-sentinel`，通过 `spring.cloud.sentinel.transport.dashboard` 上报到 Sentinel 控制台，可在控制台配置流控、熔断、热点参数规则。

### 5.7 动态配置刷新

`DynamicRefreshController` 使用 `@RefreshScope + @Value("${url:123}")` 演示了 Nacos 配置热更新：在 Nacos 中修改 `url` 配置后，无需重启即可通过 `/api/Refresh/getUrl` 拿到最新值。各服务还通过 `shared-configs: common.yaml` 共享一份公共配置。

---

## 六、核心调用链路

**① 登录拿令牌**

```
POST /auth/api/auth/login   { "username": "...", "password": "..." }
  → 网关放行登录接口
  → iam-service：Feign 调 user-service 校验 → StpUtil.login → 写 user-roles 缓存
  → 返回 Access-Token
```

**② 带令牌访问业务接口（鉴权）**

```
GET /user/api/users/123      Header: Access-Token: xxx
  → 网关 AuthenticationGlobalFilter 校验 isLogin
  → user-service：@SaCheckPermission("user:query")
       → StpInterfaceImpl 读 user-roles → role-permissions → 命中放行
```

**③ 聚合查询课程详情**

```
GET /mgmt-application/api/courses/10
  → mgmt-application
       → Feign ICourseApi   → course-service 取课程
       → Feign IInstructorApi → user-service 取讲师
       → CourseDetailConverter 合并 → CourseDetailVO
```

**④ 写评论 + 加积分（分布式事务）**

```
POST /mgmt-application/api/courseReview
  → @GlobalTransactional
       → course-service 写评论（分支①）
       → user-service  加 5 积分（分支②）
       → 异常则整体回滚
```

---

## 七、数据库与缓存

- **数据库（库级隔离）**：课程服务使用 `course_db`，用户服务使用独立用户库，聚合服务**不直连数据库**。持久层统一用 MyBatis-Plus（实体如 `Course`、`User` 用 `@TableName` 映射，主键 `@TableId(type = IdType.AUTO)`）。
- **Redis 分库**：
  - `database: 1` —— 业务缓存。
  - `database: 2`（Sa-Token `alone-redis`）—— 登录会话 / 鉴权数据，与业务隔离。
- **缓存键约定**（见 `RedisConstant`）：
  - `role-permissions`（Hash）：角色 → 权限列表。
  - `user-roles-{userId}`（String，TTL 24h）：用户 → 角色列表。

> 注意：示例代码里 `UserServiceImpl` 的密码为明文比对，仅用于演示，**生产环境务必改为加盐哈希**。

---

## 八、本地运行

### 前置依赖

启动前需准备好以下基础设施（建议用 Docker 部署）：

| 组件 | 用途 | 默认端口 |
| :--- | :--- | :---: |
| Nacos | 注册中心 + 配置中心 | 8848 |
| MySQL 8 | 业务数据库 | 3306 |
| Redis | 缓存 + 会话 | 6379 |
| Seata Server | 分布式事务 TC | 8091 |
| Sentinel Dashboard | 限流控制台 | 8080 |

### 关键约定

- 各服务通过环境变量 `SERVER_URL` 指向基础设施主机、`PROFILE` 指定激活环境（`dev`/`prod`）。
- 配置全部托管在 Nacos：`namespace=a02b0bc3-...`、`group=BIZ_GROUP`，`data-id` 形如 `course-service-dev.yaml`，并共享 `common.yaml`。

### 启动步骤

```bash
# 1. 克隆并进入工程
git clone <repo-url> && cd hiseas

# 2. 安装公共/父依赖到本地仓库（common、sa-token、cloud、*-api）
mvn clean install -Dmaven.test.skip=true

# 3. 在 Nacos 中创建对应 namespace / group，并导入各服务 *-dev.yaml + common.yaml

# 4. 设置环境变量（示例）
export SERVER_URL=10.211.55.10        # 基础设施主机
export PROFILE=dev

# 5. 依次启动（建议顺序）：user → course → iam → gateway → mgmt
#   方式 A：IDEA 直接运行各服务的 *Application 主类
#   方式 B：命令行
java -jar hiseas-center-user/hiseas-center-user-service/target/*.jar \
     -DSERVER_URL=$SERVER_URL --spring.profiles.active=dev
```

> 各 `*Application` 启动类位于对应 `-service` 模块下，如 `HiseasCenterUserServiceApplication`、`HiseasCenterGatewayApplication`、`HiseasCenterIamApplication`、`HiseasMgmtApplication`、`HiseasCenterCourseServiceApplication`。

---

## 九、配置说明

配置采用 `bootstrap.yml`（基础）+ `bootstrap-{dev|prod}.yml`（环境差异）+ Nacos 远端配置的分层结构。核心片段：

```yaml
spring:
  cloud:
    nacos:
      discovery: { server-addr: ${SERVER_URL}:8848, namespace: ..., group: BIZ_GROUP }
      config:    { file-extension: yaml, prefix: ${spring.application.name}-${spring.profiles.active},
                   shared-configs: [{ data-id: common.yaml, refresh: true, group: BIZ_GROUP }] }
    sentinel:
      transport: { dashboard: ${SERVER_URL}:8080 }

sa-token:
  token-name: Access-Token          # 令牌/Cookie 名称
  timeout: 2592000                   # 30 天
  token-style: uuid
  jwt-secret-key: ******             # JWT 密钥（生产请用强随机并外置）
  alone-redis: { database: 2, host: ${SERVER_URL}, port: 6379 }  # 会话独立库
```

> 🔐 安全提示：仓库内示例配置包含明文的 Redis 密码、JWT 密钥、Sonar Token、Nexus 地址等，**仅供演示**。实际部署请改用密文 / K8s Secret / 配置中心加密，切勿直接沿用。

---

## 十、CI/CD 与云原生部署

每个服务都带有 `.gitlab-ci.yml` + `Dockerfile` + `deployments/*.yaml`，构成完整交付链路。

**GitLab 流水线（五阶段）：**

| 阶段 | 动作 |
| :--- | :--- |
| 项目打包 | `mvn clean package -Dmaven.test.skip=true` |
| 镜像构建 | `docker build`，注入 `PROFILE` / `SERVER_URL` 构建参数 |
| 归档产物 | 打 Tag 时归档 `target/*.jar` |
| 镜像推送 | 推送到私有镜像仓库（Harbor，`$DOCKER_REGISTRY`） |
| 集群部署 | `kubectl apply` + `kubectl set image` 滚动更新到 `hiseas-dev` 命名空间 |

**Dockerfile 要点：** 基于 `eclipse-temurin:21-jre-alpine`，创建非 root 用户运行，设定时区 `Asia/Shanghai` 与 JVM 参数，通过 `--spring.profiles.active=${PROFILE}` 激活环境。

**Kubernetes：** `deployments/` 下为各服务的 `Deployment + Service`（`ClusterIP`），镜像从 Harbor 拉取（`imagePullSecrets: harbor-pass`）。对外暴露通过 **Istio Gateway API**（`hiseas-gateway-api.yml`）：在 `:32613` 监听，`HTTPRoute` 把流量导向 `hiseas-center-gateway-svc-dev:8431`。

```
Istio Gateway(:32613) ──HTTPRoute──► hiseas-center-gateway-svc-dev(:8431) ──► 各微服务 Service
```

配套基础设施：**Nexus**（Maven 私服，托管 `*-api`/公共包）、**SonarQube**（代码质量门禁，父 POM 已集成 sonar 插件）、**Harbor**（镜像仓库）。

---

## 十一、项目结构

```
hiseas/
├── hiseas-cloud/                 # 父 POM：依赖/版本/私服/Sonar 统一管理
├── hiseas-common/                # 公共库：枚举、常量、RsaUtil、MdcFilter(traceId)
├── hiseas-sa-token/              # Sa-Token 自动装配 Starter：StpInterfaceImpl/异常处理/拦截器
├── hiseas-center-gateway/        # API 网关（:8431）：路由 + 登录态校验 + 访问日志
├── hiseas-center-iam/            # 认证鉴权中心（:15000）：登录 + 角色/权限缓存初始化
├── hiseas-mgmt-application/      # 聚合服务（:10000）：Feign 编排 + Seata 分布式事务
├── hiseas-center-user/           # 用户原子服务（:8000）
│   ├── hiseas-center-user-api/       # Feign 接口 + DTO（供他人依赖）
│   └── hiseas-center-user-service/   # 用户/讲师/角色/权限/积分 CRUD 实现
├── hiseas-center-course/         # 课程原子服务（:9000）
│   ├── hiseas-center-course-api/     # Feign 接口 + DTO
│   └── hiseas-center-course-service/ # 课程/课程评论 CRUD 实现
├── deployments/                  # 各服务 K8s Deployment/Service + Istio Gateway/HTTPRoute
├── LICENCE
└── README.md
```

每个原子服务 `-service` 内部分层（以 user 为例）：

```
controller/  REST 接口（实现对应 *-api）
service/     业务接口
service/impl 业务实现（继承 MyBatis-Plus ServiceImpl）
mapper/      MyBatis-Plus Mapper
domain/      数据库实体（@TableName）
converter/   MapStruct 实体 ⇄ DTO 转换
config/      Feign 拦截器 / AppConfig
```

---

## 十二、效果截图

![截图1](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207232147376.png)

![截图2](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207233044187.png)

![截图3](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207234320196.png)

![截图4](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208142544639.png)

![截图5](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208160327106.png)

---

## 十三、参考资料与许可证

- 部署与基础设施搭建参考：<https://zhengxiang.cc/blog/?tag=Kubernetes>
- Sa-Token 官方文档：<https://sa-token.cc/>
- 许可证：见 [LICENCE](./LICENCE)
