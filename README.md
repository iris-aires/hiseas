# Hiseas Cloud

> A microservices reference project for an online course platform, built with **Spring Boot 3, Spring Cloud 2023, and Spring Cloud Alibaba**.
> It demonstrates service discovery, centralized configuration, API routing, authentication and authorization, interservice communication, distributed transactions, traffic control, and request tracing, with a **Docker, Harbor, Kubernetes, and GitLab CI/CD** delivery pipeline.

## Contents

- [1. Overview](#1-overview)
- [2. System Topology](#2-system-topology)
- [3. Technology Stack](#3-technology-stack)
- [4. Modules](#4-modules)
- [5. Architecture and Design](#5-architecture-and-design)
  - [5.1 Authentication and Authorization](#51-authentication-and-authorization)
  - [5.2 Gateway and Routing](#52-gateway-and-routing)
  - [5.3 Interservice Communication](#53-interservice-communication)
  - [5.4 Aggregation and Distributed Transactions](#54-aggregation-and-distributed-transactions)
  - [5.5 Request Tracing](#55-request-tracing)
  - [5.6 Traffic Control and Circuit Breaking](#56-traffic-control-and-circuit-breaking)
  - [5.7 Dynamic Configuration](#57-dynamic-configuration)
- [6. Core Request Flows](#6-core-request-flows)
- [7. Databases and Caching](#7-databases-and-caching)
- [8. Running Locally](#8-running-locally)
- [9. Configuration](#9-configuration)
- [10. CI/CD and Kubernetes Deployment](#10-cicd-and-kubernetes-deployment)
- [11. Project Structure](#11-project-structure)
- [12. Screenshots](#12-screenshots)
- [13. References and License](#13-references-and-license)

## 1. Overview

`hiseas-cloud` models an online course platform with **users, instructors, roles, permissions, user points, courses, and course reviews**. It demonstrates how application services and supporting infrastructure work together:

- **Domain services:** The user and course services handle CRUD operations within their own domains and use separate databases.
- **Aggregation service:** `mgmt-application` does not access a database directly. It orchestrates domain services through OpenFeign and exposes combined responses, such as course details with instructor information.
- **Identity service:** `iam` handles login, creates sessions, and initializes or updates user-role and role-permission caches.
- **API gateway:** `gateway` provides a single entry point for routing, login checks, and access logging.
- **Shared libraries:** `common` supplies enums, constants, utilities, and a trace ID filter. `sa-token` supplies shared authorization support through Spring Boot auto-configuration.

Each domain service separates its **API contract** from its **implementation**. A `*-api` module contains Feign interfaces and DTOs; a `*-service` module implements the business logic. Calling services depend on the lightweight API module without importing implementation details.

## 2. System Topology

![System topology](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208001433721.png)

A typical request follows this path:

```text
Client
  | HTTP request with Access-Token
  v
Istio Gateway API
  |
  v
hiseas-center-gateway (:8431)
  | Login checks and routing
  +-------------------+-------------------+-------------------+
  |                   |                   |                   |
  v                   v                   v                   v
iam-service      mgmt-application    course-service      user-service
  (:15000)           (:10000)           (:9000)             (:8000)
  |                   |                                       ^
  +------ OpenFeign --+-------- Calls to domain services ------+

Supporting infrastructure: Nacos / Redis / MySQL / Seata / Sentinel
```

## 3. Technology Stack

| Category | Technology | Version / Details |
| :--- | :--- | :--- |
| Language / runtime | JDK | `21.0.5 LTS` |
| Application framework | Spring Boot | `3.2.1` |
| Microservices framework | Spring Cloud | `2023.0.1` |
| Microservices integrations | Spring Cloud Alibaba | `2023.0.1.0` |
| Service discovery / configuration | Nacos | Discovery and configuration services |
| API gateway | Spring Cloud Gateway | Reactive, based on WebFlux |
| Authentication / authorization | Sa-Token | `1.38.0`, with JWT, Redis, and alone-redis |
| Interservice calls | OpenFeign + LoadBalancer | Token propagation through a Feign interceptor |
| Distributed transactions | Seata | `@GlobalTransactional` |
| Traffic control / circuit breaking | Sentinel | Dashboard integration |
| Persistence | MyBatis-Plus | `3.5.6` |
| Database | MySQL | `8.0.33` |
| Object mapping | MapStruct | `1.5.5.Final` |
| Cache / session storage | Redis | Separate logical databases for application data and Sa-Token |
| Utilities | Lombok / Hutool / Fastjson2 / Gson | Shared development utilities |
| Containers / image registry | Docker / Harbor | Docker `27.4.0` |
| Orchestration | Kubernetes + Istio Gateway API | Kubernetes `v1.31.4` |
| Artifacts / code quality | Nexus / SonarQube | Private Maven repositories and quality checks |
| CI/CD | GitLab CE | `v16.9.9`, with five-stage pipelines |

## 4. Modules

| Module | Port | Type | Responsibility |
| :--- | :---: | :--- | :--- |
| `hiseas-cloud` | — | Parent POM | Dependency versions, Nexus repositories, Sonar configuration, and compiler plugins, including Lombok and MapStruct annotation processors. |
| `hiseas-common` | — | Shared library | Enums, constants, `RsaUtil`, and the `MdcFilter` trace ID filter. |
| `hiseas-sa-token` | — | Auto-configuration library | Shared Sa-Token configuration, Redis-backed role and permission lookup through `StpInterfaceImpl`, exception handling, and interceptors. |
| `hiseas-center-gateway` | `8431` | API gateway | Routing, login checks, and access logging. |
| `hiseas-center-iam` | `15000` | Identity service | Login, session creation, and role/permission cache initialization. |
| `hiseas-mgmt-application` | `10000` | Aggregation service | Feign orchestration for course details and review/points workflows, without direct database access. |
| `hiseas-center-course` | `9000` | Course domain service | Course and review CRUD operations using `course_db`, split into API and service modules. |
| `hiseas-center-user` | `8000` | User domain service | User, instructor, role, permission, and points CRUD operations using a separate database, split into API and service modules. |

## 5. Architecture and Design

### 5.1 Authentication and Authorization

Authorization follows **RBAC: user → roles → permissions**. Redis stores authorization data so services can check access without querying the identity service for every request.

| Cache | Redis Structure | Populated When | Writer |
| :--- | :--- | :--- | :--- |
| Role permissions | Hash: `role-permissions` → `{ roleCode: ["permission1", "permission2"] }` | Identity service startup, through `@PostConstruct` | `RolePermissionInitializer` |
| User roles | String: `user-roles-{userId}` → `["roleCode", ...]`, with a 24-hour TTL | Successful login | `AuthServiceImpl` |

**Login flow in `hiseas-center-iam`:**

1. `AuthController#login` receives a `LoginDTO` containing `username` and `password`.
2. `AuthServiceImpl` calls `/api/users/login` on `user-service` through Feign. The user service checks required fields, compares the password, and verifies account activation.
3. After validation, `StpUtil.login` creates a session for the user ID and attaches `fullName` as additional information. The token is named `Access-Token`.
4. The identity service retrieves the user's roles through Feign and writes them to `user-roles-{userId}`.

**Authorization in business services:**

`StpInterfaceImpl` in `hiseas-sa-token` implements Sa-Token's `StpInterface`:

- `getRoleList(loginId)` reads roles from `user-roles-{loginId}`.
- `getPermissionList(loginId)` looks up each role in the `role-permissions` hash, combines the permissions, and removes duplicates.

Declare required permissions with annotations:

```java
@SaCheckPermission("user:query")
@GetMapping("/{userId}")
public ResponseEntity<?> getUserById(@PathVariable Long userId) { ... }
```

Use `@SaIgnore` on endpoints that do not require authorization, such as the aggregated course details endpoint.

Sa-Token uses `alone-redis` with **logical database 2**, while application caching uses **logical database 1**. These are separate keyspaces within Redis, not separate Redis instances.

### 5.2 Gateway and Routing

`hiseas-center-gateway` uses reactive Spring Cloud Gateway with two routing mechanisms.

**Explicit routes**, configured in `bootstrap-dev.yml`, remove the first path segment with `StripPrefix=1`:

| Incoming Path | Destination |
| :--- | :--- |
| `/mgmt-application/**` | `lb://mgmt-application` |
| `/course/**` | `lb://course-service` |
| `/user/**` | `lb://user-service` |
| `/auth/**` | `lb://iam-service` |

**Discovery-based routes** are enabled with `gateway.discovery.locator.enabled=true`, allowing routes to be generated from service names registered in Nacos.

Two global filters handle cross-cutting concerns:

- `AuthenticationGlobalFilter` binds the reactive exchange with `SaReactorSyncHolder.setContext(exchange)`, allows the login endpoint, and checks other requests with `StpUtil.isLogin()`. Unauthenticated requests receive a consistent JSON error response.
- `LoggingGlobalFilter`, ordered with `@Order(0)`, records the HTTP method, URI, response status, and duration.

### 5.3 Interservice Communication

Each domain service exposes Feign interfaces and DTOs through its `*-api` module. Service names and direct URLs are configurable:

```java
@FeignClient(
    contextId = "cn-com-hiseas-user-api-IUserApi",
    name = "${cn.com.hiseas.user.api.name:user-service}",
    path = "/api/users",
    url = "${cn.com.hiseas.user.api:}")
```

By default, calls resolve the Nacos service name `user-service`. A configured URL can override discovery for integration testing.

The aggregation service's `TokenPassingInterceptor`, a Feign `RequestInterceptor`, copies `Access-Token` from the incoming request to downstream requests to preserve the caller's authentication context.

| Feign Interface | Path | Method | Service |
| :--- | :--- | :--- | :--- |
| `IUserApi` | `/api/users/login` | POST | user-service |
| `IRoleApi` | `/api/role`, `/api/role/{userId}/roles` | GET | user-service |
| `IPermissionApi` | `/api/permissions` | GET | user-service |
| `IInstructorApi` | `/api/instructors/{instructorId}` | GET | user-service |
| `IUserPointsApi` | `/api/userScore` | PUT | user-service |
| `ICourseApi` | `/api/course/{courseId}` | GET | course-service |
| `ICourseReviewApi` | `/api/courseReview` | POST | course-service |

### 5.4 Aggregation and Distributed Transactions

`hiseas-mgmt-application` acts as a **backend-for-frontend (BFF) and orchestration layer**:

- **Course details:** `CourseAggregationController` retrieves a course, uses its `instructorId` to retrieve instructor information, and merges the results into `CourseDetailVO` through MapStruct.
- **Course reviews and user points:** `CourseReviewServiceImpl` coordinates a review insert in the course database and a five-point increment in the user database. Seata's `@GlobalTransactional` coordinates the two branches so failures can roll back the workflow.

```java
@GlobalTransactional
public ResponseEntity<?> createCourseReview(CourseReviewRespDto dto) {
    courseReviewApi.createCourseReview(dto);    // Branch 1: course database
    userPointsApi.accumulateUserPoints(points); // Branch 2: user database
    // Propagate failures so the transaction manager can trigger rollback.
}
```

### 5.5 Request Tracing

`MdcFilter` in `hiseas-common` generates or reuses a `traceId` for incoming requests and adds it to SLF4J's MDC. The logging pattern includes that context:

```yaml
logging.pattern.level: '%5p [${spring.application.name},%mdc{traceId:-},%mdc{ts:-}]'
```

Trace IDs allow related log entries to be correlated. For correlation across service boundaries, the same ID must also be propagated to downstream requests. Token propagation preserves the authentication context alongside this tracing context.

### 5.6 Traffic Control and Circuit Breaking

The course and aggregation services integrate `spring-cloud-starter-alibaba-sentinel`. The `spring.cloud.sentinel.transport.dashboard` setting connects them to the Sentinel dashboard, where traffic control, circuit-breaking, and hotspot parameter rules can be configured.

### 5.7 Dynamic Configuration

`DynamicRefreshController` demonstrates Nacos configuration refresh using `@RefreshScope` and `@Value("${url:123}")`. After updating `url` in Nacos, `/api/Refresh/getUrl` exposes the refreshed value without restarting the service.

Services also load shared settings from `common.yaml` through `shared-configs`.

## 6. Core Request Flows

**Login and token issuance:**

```text
POST /auth/api/auth/login   { "username": "...", "password": "..." }
  → Gateway allows the login request
  → iam-service calls user-service through Feign to validate credentials
  → StpUtil.login creates a session; user roles are cached
  → Access-Token is returned
```

**Authenticated business request:**

```text
GET /user/api/users/123      Header: Access-Token: xxx
  → AuthenticationGlobalFilter checks login status
  → user-service checks @SaCheckPermission("user:query")
  → StpInterfaceImpl reads user roles and role permissions
  → Access is allowed if the required permission is present
```

**Aggregated course details:**

```text
GET /mgmt-application/api/courses/10
  → mgmt-application
      → ICourseApi calls course-service for course information
      → IInstructorApi calls user-service for instructor information
      → CourseDetailConverter combines the results into CourseDetailVO
```

**Create a review and award points:**

```text
POST /mgmt-application/api/courseReview
  → @GlobalTransactional
      → course-service inserts the review (branch 1)
      → user-service adds 5 points (branch 2)
      → A propagated failure triggers rollback
```

## 7. Databases and Caching

- **Separate databases:** The course service uses `course_db`; the user service has its own database. The aggregation service does not connect directly to either database.
- **Persistence:** MyBatis-Plus maps entities such as `Course` and `User` with `@TableName`, using auto-increment primary keys through `@TableId(type = IdType.AUTO)`.
- **Redis logical databases:** Database 1 holds application cache data; database 2 is used by Sa-Token's `alone-redis` integration.
- **Cache key conventions**, defined in `RedisConstant`:
  - `role-permissions`: a hash mapping roles to permission lists.
  - `user-roles-{userId}`: a string containing the user's role list, with a 24-hour TTL.

> The sample `UserServiceImpl` compares passwords in plaintext. This is demonstration code; use salted password hashing before deploying a real authentication system.

## 8. Running Locally

### Prerequisites

Use JDK 21 and Maven, and prepare the following infrastructure. Docker can be used to run these dependencies.

| Component | Purpose | Default Port |
| :--- | :--- | :---: |
| Nacos | Service discovery and centralized configuration | 8848 |
| MySQL 8 | Business databases | 3306 |
| Redis | Caching and session storage | 6379 |
| Seata Server | Distributed transaction coordinator | 8091 |
| Sentinel Dashboard | Traffic control dashboard | 8080 |

### Environment Conventions

- `SERVER_URL` identifies the infrastructure host.
- `PROFILE` selects the environment in deployment scripts, such as `dev` or `prod`.
- Nacos settings use an environment-specific namespace, `group=BIZ_GROUP`, per-service data IDs such as `course-service-dev.yaml`, and a shared `common.yaml`. Match the namespace IDs to your own Nacos setup.

### Build and Start

Clone the repository and enter its directory:

```bash
git clone <repo-url>
cd hiseas
```

The repository root has no `pom.xml`. Install the parent POM and shared libraries first, then build the services using their own POMs:

```bash
mvn -f hiseas-cloud/pom.xml clean install -Dmaven.test.skip=true
mvn -f hiseas-common/pom.xml clean install -Dmaven.test.skip=true
mvn -f hiseas-sa-token/pom.xml clean install -Dmaven.test.skip=true

# Each domain parent builds its API and service modules.
mvn -f hiseas-center-user/pom.xml clean install -Dmaven.test.skip=true
mvn -f hiseas-center-course/pom.xml clean install -Dmaven.test.skip=true

mvn -f hiseas-center-iam/pom.xml clean install -Dmaven.test.skip=true
mvn -f hiseas-center-gateway/pom.xml clean install -Dmaven.test.skip=true
mvn -f hiseas-mgmt-application/pom.xml clean install -Dmaven.test.skip=true
```

Configure access to the Nexus repositories declared in the POMs, or adapt those repository settings for your environment. The commands above skip tests, matching the sample packaging workflow.

Create the required Nacos namespace and group, populate the service configurations and `common.yaml`, and prepare the databases before starting the applications.

```bash
# Replace this with your infrastructure host.
export SERVER_URL=10.211.55.10
export PROFILE=dev

# Start the user service. Repeat for the other executable service JARs.
java -DSERVER_URL="$SERVER_URL" \
     -jar hiseas-center-user/hiseas-center-user-service/target/*.jar \
     --spring.profiles.active="$PROFILE"
```

Recommended startup order: **user → course → iam → gateway → mgmt**. Run each application in its own terminal, or run its main class from your IDE:

- `HiseasCenterUserServiceApplication`
- `HiseasCenterCourseServiceApplication`
- `HiseasCenterIamApplication`
- `HiseasCenterGatewayApplication`
- `HiseasMgmtApplication`

## 9. Configuration

Configuration is layered across `bootstrap.yml`, environment-specific `bootstrap-dev.yml` or `bootstrap-prod.yml`, and remote Nacos configuration. The following is an illustrative excerpt; replace placeholders with environment-specific values:

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: ${SERVER_URL}:8848
        namespace: <your-namespace-id>
        group: BIZ_GROUP
      config:
        file-extension: yaml
        prefix: ${spring.application.name}-${spring.profiles.active}
        shared-configs:
          - data-id: common.yaml
            refresh: true
            group: BIZ_GROUP
    sentinel:
      transport:
        dashboard: ${SERVER_URL}:8080

sa-token:
  token-name: Access-Token          # Token / cookie name
  timeout: 2592000                  # 30 days, in seconds
  token-style: uuid
  jwt-secret-key: <your-jwt-secret> # Supply a strong secret externally
  alone-redis:
    database: 2                    # Dedicated logical database for session data
    host: ${SERVER_URL}
    port: 6379
```

> Sample configuration includes embedded Redis passwords, JWT secrets, a Sonar token, and environment-specific Nexus addresses. Replace demonstration credentials and use external secret management, Kubernetes Secrets, or encrypted configuration for real deployments.

## 10. CI/CD and Kubernetes Deployment

Service directories provide `.gitlab-ci.yml` and `Dockerfile` files, with Kubernetes manifests under `deployments/`.

The GitLab delivery pipeline has five stages:

| Stage | Action |
| :--- | :--- |
| Package | Build the service with `mvn clean package -Dmaven.test.skip=true`. |
| Build image | Run `docker build`, supplying `PROFILE` and `SERVER_URL` build arguments. |
| Archive artifacts | Archive `target/*.jar` for tagged builds. |
| Push image | Push images to the private Harbor registry identified by `$DOCKER_REGISTRY`. |
| Deploy | Use `kubectl apply` and `kubectl set image` to roll out services in the `hiseas-dev` namespace. |

**Container runtime:** Dockerfiles use `eclipse-temurin:21-jre-alpine`, run as a non-root user, set the timezone to `Asia/Shanghai`, and configure JVM options. The active environment is selected with `--spring.profiles.active=${PROFILE}`.

**Kubernetes:** Deployment and `ClusterIP` Service manifests pull images from Harbor using `imagePullSecrets: harbor-pass`. The Istio Gateway API configuration in `hiseas-gateway-api.yml` listens on port `32613`; an `HTTPRoute` forwards requests to `hiseas-center-gateway-svc-dev:8431`.

```text
Istio Gateway (:32613)
  → HTTPRoute
  → hiseas-center-gateway-svc-dev (:8431)
  → Individual microservice Services
```

Supporting tools include **Nexus** for shared libraries and API artifacts, **SonarQube** for code quality analysis through the parent POM's Sonar plugin, and **Harbor** for container image storage.

## 11. Project Structure

```text
hiseas/
├── hiseas-cloud/                  # Parent POM: dependencies, repositories, compiler, Sonar
├── hiseas-common/                 # Enums, constants, RsaUtil, MdcFilter
├── hiseas-sa-token/               # Sa-Token auto-configuration and authorization support
├── hiseas-center-gateway/         # API gateway (:8431): routing, login checks, access logs
├── hiseas-center-iam/             # Identity service (:15000): login and authorization caches
├── hiseas-mgmt-application/       # Aggregation (:10000): Feign orchestration, Seata transactions
├── hiseas-center-user/            # User domain service (:8000)
│   ├── hiseas-center-user-api/     # Feign interfaces and DTOs for calling services
│   └── hiseas-center-user-service/ # User, instructor, role, permission, and points CRUD
├── hiseas-center-course/          # Course domain service (:9000)
│   ├── hiseas-center-course-api/   # Feign interfaces and DTOs
│   └── hiseas-center-course-service/ # Course and review CRUD
├── deployments/                   # Kubernetes Deployments/Services, Istio Gateway/HTTPRoute
├── LICENCE
└── README.md
```

A domain service implementation follows these layers, illustrated by the user service:

```text
controller/   REST endpoints implementing the API contracts
service/      Business service interfaces
service/impl/ Business implementations extending MyBatis-Plus ServiceImpl
mapper/       MyBatis-Plus mappers
domain/       Database entities mapped with @TableName
converter/    MapStruct entity / DTO conversions
config/       Feign interceptors and application configuration
```

## 12. Screenshots

![Platform screenshot 1](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207232147376.png)

![Platform screenshot 2](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207233044187.png)

![Platform screenshot 3](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207234320196.png)

![Platform screenshot 4](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208142544639.png)

![Platform screenshot 5](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208160327106.png)

## 13. References and License

- Deployment and infrastructure notes: <https://zhengxiang.cc/blog/?tag=Kubernetes>
- Sa-Token documentation: <https://sa-token.cc/>
- License: see [LICENCE](./LICENCE).
