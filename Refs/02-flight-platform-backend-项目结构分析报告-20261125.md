# 飞参判读平台 · 后端项目结构分析报告（20261125 版）

> 本文件是对 2026-11-18 版 `03-flight-platform-backend-项目结构分析报告` 的重写，已修正其中已过时的描述（如“`backend/` 即 git 根”、`flight-data-batch-read-validation.md` 存在等）。所有路径、类名、端口均已从代码核实；文档与代码冲突时**以代码为准**。

---

## 1. 项目概览

### 1.1 是什么

一个「**飞参判读平台**」的后端服务，基于 **RuoYi-Vue 3.9.2（Spring Boot 3）** 脚手架二次开发。

它把飞行参数遥测时序数据（ClickHouse）、机型信号定义（ICD）、判读模板三样东西合在一起，自动产出判读结论、曲线图与报告：

```
飞参时序 + ICD 位级定义 + 判读模板
        → 阶段识别（哪一段飞行）
        → 规则判读（这段飞行是否正常）
        → 曲线渲染 + 报告归档（HTML/PDF/DOCX）
```

### 1.2 关键定位

`flight-platform-app/pom.xml` 明确声明 —— **"飞参判读平台唯一启动与部署模块"**。5 个 `ruoyi-*` 模块是脚手架基座，全部自研业务与启动入口都收敛在 `flight-platform-app` 内。

本次分析按要求**只深挖 `flight-platform-app`**，ruoyi 基座仅在集成点处引用。

### 1.3 三处最容易误解的命名（观察）

| 名字 | 实际含义 | **不是**什么 |
|---|---|---|
| **Keyword（关键字）** | **24 个内置的信号断言/统计原语**（`AssertGreaterThan`、`LogMaxValue`、`LogValueJumpTime` …），对飞参时序做数值判定 | 不是文本关键词，不是正则匹配（全仓无任何正则/AC 自动机） |
| **Template（模板）** | **判读逻辑定义** —— 阶段/规则的 GROUP-CONDITION 逻辑树 + Keyword 绑定 + ICD 字段引用 + 版本 | 不是 Word/HTML 报告模板。报告版式是固定的 Thymeleaf 模板 `report/default-report.html` |
| **ICD** | **Interface Control Document**，飞参报文的**位级结构定义**（字节偏移、位区间、量纲、上下限、冗余通道） | 不是「初始条件与数据」 |

另外，`business/graph` 指的是**飞参曲线/图表**，不是「计算图」。该包内没有节点/边模型。

### 1.4 规模统计（观察）

| 业务域 | 文件数 | 职责 |
|---|---|---|
| `keywords` | 144 | 24 个信号断言原语、执行契约、三值结论代数 |
| `template` | 141 | 判读模板：版本状态机、逻辑树、DataDiagnostics 导入、运行时快照 |
| `task` | 98 | 四阶段任务编排、租约状态机、执行尝试、结果提交 |
| `icd` | 91 | 报文位级定义、冗余识别、版本与发布、外部同步 |
| `messaging` | 89 | 事务性 Outbox/Inbox、Kafka 契约、租约令牌、DLT |
| `ingestion` | 78 | 飞参发现与对账、ClickHouse 适配、Redis 采样缓存 |
| `judgement` | 77 | 判读执行引擎、快照装载、逻辑树运行、结果装配 |
| `graph` | 74 | 公式库、绘图变量、曲线对齐采样、图表制品管线 |
| `report` | 53 | 报告生命周期、HTML 渲染、图片内联、PDF/DOCX 转换 |
| `authorization` | 19 | 资源级授权（注解 + AOP + 授权表） |
| `render` | 7 | Java ↔ Node 渲染服务桥接（HTTP 客户端 + 降采样） |
| **business 合计** | **871** | |
| `common` | 29 | 存储、配置、异常、公式引擎、时间窗、JSON 规范化 |
| 启动类 | 1 | `FlightPlatformApplication` |
| **主源码合计** | **901** | |
| 测试 | 295 | 与 main 包结构一一镜像 |

---

## 2. 技术栈

| 维度 | 选型（证据：父 `pom.xml` + `flight-platform-app/pom.xml`） |
|---|---|
| 语言 / 构建 | Java 17，Maven 多模块聚合（仓库根自带 `mvnw`） |
| 框架 | Spring Boot 3.5.14，Spring MVC，Spring Security |
| Web / 文档 | knife4j + springdoc-openapi（Swagger UI / v3/api-docs），8 个分组 |
| 认证 | JWT（jjwt 0.9.1）+ kaptcha 验证码 |
| ORM / 分页 | **纯 MyBatis 3.0.5**（无 MyBatis-Plus）+ PageHelper 2.1.1（`helperDialect: oracle`） |
| 连接池 | Druid 1.2.28（dynamicDataSource master/slave，slave 未启用） |
| 主数据库 | **达梦 DM8**（`DmJdbcDriver18` 8.1.3.140），schema `FLIGHT` |
| 时序数据库 | **ClickHouse**（housepower native 驱动 + clickhouse-jdbc HTTP 驱动，**只读**） |
| 消息中间件 | **Kafka**（spring-kafka，Outbox/Inbox 可靠投递，12 个 topic / 17 种事件） |
| 缓存 / 租约 | **Redis**（Lettuce），承担采样缓存 + 幂等 Inbox 之外的加速层 |
| 对象存储 | AWS SDK v2 S3（可指 MinIO），**无本地文件系统实现** |
| 报告生成 | docx4j 17.0.4（原生 WordprocessingML）、jsoup 1.23.2（HTML 净化）、Playwright 1.61.0（HTML→PDF） |
| 图表渲染 | **独立部署的 Node.js 服务**（Puppeteer-core + 系统 Chrome） |
| 链路追踪 | Micrometer Tracing(Brave) + 自定义 `TraceIdFilter` / `TraceIdProvider` |
| 表达式 | **自研 `FormulaExpression`**（594 行递归下降解析器，仅用于绘图公式）；SpEL 仅用于资源授权切面 |
| 其他 | fastjson2、poi-ooxml、oshi、yauaa、zstd-jni（ClickHouse native 传输必需） |

**架构性选择**：无 `@EnableAsync`、无 `@Async`、无 `CompletableFuture`。异步一律靠 **Kafka Outbox/Inbox + `@Scheduled` 轮询**，不引入 Spring 异步线程池。

---

## 3. 目录与关键文件地图

```
backend/
├── pom.xml                         # 聚合 POM：版本属性 / 依赖管理（仓库根，自带 mvnw）
├── ruoyi-common|system|framework|quartz|admin   # RuoYi 脚手架五模块（本次不深入，默认不动）
├── flight-platform-app/            # 唯一启动与部署模块（业务全在这里）
│   ├── pom.xml                     # 打包插件：浏览器外置 + antrun copy
│   └── src/
│       ├── main/java/com/huike/flight/platform/
│       │   ├── FlightPlatformApplication.java   # 启动入口，扫描 com.huike.flight.platform + com.ruoyi
│       │   ├── business/                        # 11 个业务域（见第 7 节）
│       │   └── common/                          # 自研通用层（29 文件，见 §5.1）
│       ├── main/resources/
│       │   ├── application.yml                  # 主配置 + flight.* 业务开关 + node-renderer.base-url
│       │   ├── application-dev.yml / -prod.yml / -druid.yml
│       │   ├── contracts/                       # JSON Schema（stage-analysis / rule-judgment / report-result / report-result-ref，4 个 schema）
│       │   ├── mapper/**/*Mapper.xml            # MyBatis 映射（DM + clickhouse 子目录）
│       │   ├── redis/ingestion/*.lua            # 采样缓存的 Lua 脚本（原子发布/追加）
│       │   ├── templates/report/default-report.html
│       │   ├── browsers/                        # 浏览器二进制（**已入 git**，坑 #18）
│       │   └── mybatis/mybatis-config.xml, logback-spring.xml
│       └── test/java/...                        # 295 个测试类（镜像 main 包结构）
├── node-renderer/                  # 独立部署的 Node 图表渲染服务
│   ├── src/server.js, services/{chromeDetector,browserPool,chartRenderer}.js
│   ├── src/public/{render.html,chart-renderer.js}
│   ├── package.json（start: node src/server.js，端口默认 19876）
│   └── README.md, GRAPH_GENERATION_CONTRACT.md, package-lock.json
├── sql/                            # flight-dm.sql(58 表) / clickhouse.sql / ruoyi.sql / 3 个迁移脚本
└── error-codes.md                  # 错误码台账（160 条，与代码一致，有契约测试守着）
```

---

## 4. 启动与运行路径

### 4.1 启动

启动入口：`flight-platform-app/src/main/java/com/huike/flight/platform/FlightPlatformApplication.java`，`@SpringBootApplication(scanBasePackages = {"com.huike.flight.platform", "com.ruoyi"})`。

构建与启动命令（cwd = `backend/flight-platform-backend/`，仓库根自带 `mvnw`）：

```bash
mvn -o -pl flight-platform-app -am compile                              # 编译（EXIT=0）
mvn -pl flight-platform-app -am test                                    # 单测
mvn -pl flight-platform-app -am -DskipTests package                     # 打包
mvn -o -pl flight-platform-app -am install -DskipTests                  # 启动前先装依赖
mvn -o -pl flight-platform-app spring-boot:run                          # 启动（**不带 -am**）
```

构建注意事项（都是踩过的坑）：

- **`mvn compile` 只编 main，不编 test** —— 改了公共方法签名后必须至少跑一次 `mvn test`，否则测试模块编译失败是事后才发现。
- `mvn -am` 增量编译会报「找不到符号」**假错误**（上游 `target/classes` 缺类）。**原样重跑一次**或先 `mvn -pl <上游模块> clean` 再跑，别为此改无关文件。
- 带 `-am` 跑 `-Dtest=<单类>` 时**必须**同时加 `-Dsurefire.failIfNoSpecifiedTests=false`，否则无匹配测试的模块直接 BUILD FAILURE。
- `mvn spring-boot:run` 的参数走 `spring-boot.run.arguments`，**不要**写 `mvn spring-boot:run --server.port=8080`（Maven 判为无法解析）。
- `spring-boot:run` **不能跑在聚合根上**（会报「Unable to find a suitable main class」）。
- 测试用 Redis 端口在 `application-test.yml` 里写的是 **7001**，实际是 **6379** ⇒ 跑测试加 `-Dspring.data.redis.port=6379`。
- 本地起测加 `--spring.devtools.restart.enabled=false`：外部 `mvn test` 会改 `target/classes` 触发 devtools 重启，重启后 MyBatis 会把同一份 mapper XML 解析两次，报 `Result Maps collection already contains key` 拖垮整个应用。
- 跑全量测试前**先记基线** —— 仓库存在与本次任务无关的存量红。

后端**没有独立的 typecheck**，`compile` 即静态检查。

### 4.2 运行时强依赖

| 服务 | 地址 | 说明 |
|---|---|---|
| **达梦 DM8** | `192.168.2.201:5237`（容器内 5236） | 主库，`SYSDBA`，schema `FLIGHT`。**必需** |
| **ClickHouse** | `192.168.2.201:9000`（native 传输） | 飞参时序与 ICD 规则源，**只读**。dev profile 库名 `flight_platform`，默认 `HT_DATAS_DB` |
| **Redis** | `192.168.2.201:6379` | 采样缓存、引擎结果缓存 |
| **Kafka** | `192.168.2.201:29092` | Outbox/Inbox 事件，**12 个 topic / 17 种事件** |
| **MinIO**（S3 兼容） | `http://192.168.2.201:9001` | bucket `flight-reports`，`path-style-access=true`。**只新建独立 bucket，不得改删既有资源** |
| **Node 渲染服务** | `http://localhost:19876` | **独立部署**，Java 不负责启动或管理；Java 侧通过 `node-renderer.base-url` 配置（顶层段，默认 `${NODE_RENDERER_BASE_URL:http://localhost:19876}`） |

### 4.3 跨平台说明

后端本身是 JVM 应用，理论跨平台；但生产部署与测试均围绕达梦 DM8 + ClickHouse + Kafka + MinIO 这些外部服务展开，且依赖系统浏览器做 PDF/图表渲染，实际交付以 **Linux 服务器部署 + 本地 Windows 开发调试** 为主。

---

## 5. 架构分层

```
SCAFFOLD[RuoYi 脚手架基座]
  C[ruoyi-common] --> F[ruoyi-framework] --> S[ruoyi-system]
  Q[ruoyi-quartz]  A[ruoyi-admin: 系统 Controller]

APP[flight-platform-app 自研业务]
  CTRL[Controller 层（21 个自研 Controller）]
  SVC[Service / Engine 层]
  PERSIST[Mapper / Repository（MyBatis 手写 XML）]
  MSG[messaging: Outbox / Inbox（Kafka）]
  COMMON[common: 存储 / 公式 / 时间窗 / 异常]

CTRL --> SVC --> PERSIST
SVC <--> MSG
SVC --> COMMON
PERSIST --> DM[(达梦 DM8 主库)]
SVC --> CH[(ClickHouse 飞参数据<br/>只读)]
SVC --> S3[(S3 / MinIO<br/>报告与曲线)]
MSG <--> KAFKA[(Kafka)]
SVC -. HTTP .-> NR[node-renderer<br/>独立部署]
NR --> S3
```

### 5.1 自研通用层（`platform/common`，29 文件）

| 子包 | 作用 | 关键点 |
|---|---|---|
| `config` | `DataSourceConfig`、`ClickHouseMyBatisConfig`、`S3Properties` | **多数据源靠静态分流，不靠运行时路由**（见 §8.3） |
| `storage` (+`s3`) | `ObjectStorage` 抽象 + `S3ObjectStorageAdapter` | **只有 S3 一个实现，无本地文件系统实现** |
| `expression` | `FormulaExpression`（594 行自研递归下降解析器） | 仅用于绘图公式；**判读不用表达式引擎** |
| `json` | `CanonicalJsonHasher` | 键递归排序 + SHA-256 → `content_checksum` |
| `time` | `TimeWindow` | 全应用统一 **epoch 毫秒、左闭右开**，时区无关 |
| `exception` | `JsonConversionException` | 真正的异常体系在 `ruoyi-common`（`PlatformException` → `BusinessException`） |
| `aspect` / `annotation` | `AutoFillAspect` + `@AutoFill`/`@AuditField` | 审计字段自动填充，**仅 template 域使用** |
| `persistence` | `DataAccessExceptionClassifier` | 识别达梦重复键（`-6602` / SQLState `23505`） |
| `utils` | `JsonUtil`、`TimeWindow`、`ContentHash`、`TextLimitUtils`、`HtmlDocumentUtils` … | `ThrowUtils` 引入了 knife4j 依赖（Web 层库进入公共层） |

**约定缺失（观察）**：**没有** BaseEntity / BaseMapper / BaseRepository。持久化约定由各域自行落地，且**审计字段与逻辑删除三套并存**（ICD 用 `created_at`+物理删除；template 用 `create_by`+`is_deleted`；RuoYi 用 `create_by`）。

---

## 6. 主数据/控制流

### 6.1 全景链路

```
发现任务(@Scheduled 5min) → ClickHouse → 达梦 DM8 → Outbox → Kafka → Inbox → 判读执行 → 报告生成 → Node 渲染 → S3 归档
```

### 6.2 飞参接入（ingestion）：**拉模型，不是推送**

`business/ingestion/service/FlightDataDiscoveryService.java` → `DefaultFlightDataAccessService.java`。通过 ClickHouse 发现飞参数据，做对账与采样缓存（Redis Lua 脚本原子发布/追加）。

### 6.3 可靠事件（messaging）：事务性 Outbox + 持久化 Inbox

`business/messaging/outbox/publishing/OutboxEventPublisher.java` → `business/messaging/inbox/admission/InboxAdmissionService.java`。

- 12 个 topic / 17 种事件。
- 三重身份对账：`event_id` + `event_key` + Kafka offset（**禁区 Z6**，改幂等语义会导致重复消费或误判冲突）。
- 事务性 Outbox 保证「落库 + 投递」原子性。

### 6.4 判读执行（judgement + task + keywords）

- `business/judgement/model/JudgementModelBuilder.java` → 快照如何编译成逻辑树
- `business/judgement/execution/LogicTreeRunner.java` → 窗口如何逐级传递
- `business/judgement/execution/LogicResultAggregator.java` → 三值结论与窗口交并
- `business/keywords/execution/executor/ComparisonKeywordEvaluator.java` → 单个 Keyword 怎么求值
- `business/task/service/TaskStateService.java` → 状态机与租约

Keyword 是 **24 个内置信号断言原语**（非文本关键词），Java Bean 与 DB 行（`fp_keyword` 三表）**必须同时到位**，否则启动或模板发布会失败。

### 6.5 任务编排（task）

四阶段任务编排、租约状态机、执行尝试、结果提交。注意：**任务归属未强制**（`selectOwnedDetail` 只按 `task_id` 过滤，`owned(id, owner)` 收了 owner 却不用），存在跨用户读取风险（**风险 R1 / 坑 #9**）。

### 6.6 报告生成（report + render + node-renderer）

- `business/report/generation/ResultReadyInboxHandler.java` → `ReportArchiveService` → `business/render/ChartRenderService.java`
- `ChartRenderService` 用 `RestTemplate` POST `{node-renderer.base-url}/api/render/static-chart`，健康检查打 `/health`。
- 失败抛 `GraphErrorCode.RENDER_UNAVAILABLE` / `RENDER_RESULT_INVALID`（`InfrastructureException`）。
- Node 渲染服务独立部署，端口默认 19876，依赖系统 Chrome（`CHROME_PATH`），**不认后端内置的 `browsers/` 目录**。
- `chart-renderer.js` 是从前端仓库手工同步过去的（前端 `npm run build:chart` 产物），改前端图表引擎后**必须**同步。

---

## 7. 核心业务模块

| 业务域 | 主要职责 | 关键 Controller / 入口 |
|---|---|---|
| `authorization` | 资源级授权（注解 + AOP + 授权表） | 资源授权注解 + AOP 切面 |
| `graph` | 飞参曲线/图表，公式库、绘图变量、曲线对齐采样 | `GraphController`、`GraphFormulaController`、`FormulaLibraryController`、`PlotVariableController`、`StaticGraphController` |
| `icd` | 报文位级定义、冗余识别、版本与发布、外部同步 | `AirPlaneController`、`IcdVersionController`、`IcdBlockController`、`IcdFieldController`、`IcdTreeController`、`IcdRedundancyGroupController`、`IcdRedundancyMemberController`、`IcdBlockRedundancyGroupController` |
| `ingestion` | 飞参发现与对账、ClickHouse 适配、Redis 采样缓存 | `FlightDataDiscoveryService` 等 |
| `judgement` | 判读执行引擎、快照装载、逻辑树运行、结果装配 | `JudgementModelBuilder`、`LogicTreeRunner`、`LogicResultAggregator` 等 |
| `keywords` | 24 个信号断言原语、执行契约、三值结论代数 | `KeywordExecutorRegistry`、`KeywordArgumentsFactory` 等 |
| `messaging` | 事务性 Outbox/Inbox、Kafka 契约、租约令牌、DLT | `OutboxEventPublisher`、`InboxAdmissionService` 等 |
| `render` | Java ↔ Node 渲染服务桥接（HTTP 客户端 + 降采样） | `ChartRenderService`、`NodeRendererProcess` |
| `report` | 报告生命周期、HTML 渲染、图片内联、PDF/DOCX 转换 | `ReportArchiveService`、`ReportDebugController`（`@Anonymous` 写端点，禁区外但高危） |
| `task` | 四阶段任务编排、租约状态机、执行尝试、结果提交 | `TaskManagementController`、`TaskQueryService`、`TaskCreationService` |
| `template` | 判读模板：版本状态机、逻辑树、导入、运行时快照 | `TemplateController` 等 |

权限体系：两套并行（Spring Security `@PreAuthorize` + 业务资源授权注解）。`sys_menu.perms` 与 `sys_role_menu` 绑定，非 admin 账号需重新登录 token 才生效；admin 走 `*:*:*` 特殊通道，权限验收必须用非 admin 账号。

---

## 8. 状态与数据契约

### 8.1 JSON 结果契约

`resources/contracts/` 使用 JSON Schema Draft 2020-12 管理持久化结果格式，现有 4 个 schema：

| 结果类型 | schemaVersion | Schema 文件 |
|---|---|---|
| STAGE_ANALYSIS | stage-analysis/1.0 | `stage-analysis-1.0.schema.json` |
| RULE_JUDGMENT | rule-judgment/1.0 | `rule-judgment-1.0.schema.json` |
| 报告结果快照 | report-result/1.0 | `report-result-1.0.schema.json` |
| 报告结果引用事件 | report-result-ref/1.0 | `report-result-ref-1.0.schema.json` |

核心约定（见 `contracts/README.md`）：

- 数据库 `payload_schema_version` 与 JSON 根 `schemaVersion` 必须一致。
- 时间为飞参记录内毫秒坐标，非 Unix 时间戳；时间窗为左闭右开且只存 `startMillis`/`endMillis`。
- 规则级 `logicType` 为**下发期提升字段，不落库**；它不参与 `content_checksum`，也不写入 Schema（已知差异，不是遗漏）。
- 对象字段递归按名称排序、数组保留业务顺序后，以 UTF-8 JSON 计算 SHA-256，写入 `content_checksum`。
- 修改字段时应同步检查 Assembler、前端解析、演示数据与测试；不兼容变更应新增版本文件（**禁区 Z4**）。

### 8.2 数据存储

- 主数据源：达梦 DM8，schema `FLIGHT`。
- 时序源：ClickHouse，**只读**，平台侧不得写入（禁区）。
- 持久层：纯 MyBatis + 手写 XML，无 MyBatis-Plus，无 BaseEntity / BaseMapper / BaseRepository。
- 审计字段与逻辑删除三套并存（见 §5.1）。
- `sql/flight-dm.sql` 是纯结构脚本（58 表），不含任何 INSERT；Keyword 目录三表无种子脚本，新增 Keyword 需手写 INSERT。

### 8.3 多数据源接线方式

多数据源靠**静态分流，不靠运行时路由**。`ClickHouseMyBatisConfig` 中 `SqlSessionTemplate` / `SqlSessionFactory` 故意不暴露为 Spring Bean（禁区 Z1），否则会命中 MyBatis 自动配置的 `@ConditionalOnMissingBean`，导致达梦默认模板不再创建，全部 `@Mapper` 绑定失败。

### 8.4 消息契约

- Kafka 12 个 topic / 17 种事件。
- Outbox/Inbox 可靠投递，三重身份对账（禁区 Z6）。
- 死信工厂 Bean 定义了 3 处但监听器引用的是共享工厂，专用 DLT 路由与不可重试策略不生效（坑 #16）。

### 8.5 配置开关（`flight.*`）

业务参数全部外置到 `application.yml` 的 `flight.*` 节点，不新增顶层配置段（`node-renderer.base-url` 是现存唯一例外，不得新增第二例）。

密钥必须用环境变量占位符（如 `${FLIGHT_XXX_KEY:}`），不得硬编码明文（现存明文凭据见坑 #12）。

`flight.*` 多数占位符无默认值，缺一个就启动失败。

---

## 9. 外部依赖

| 依赖 | 版本 | 用途 |
|---|---|---|
| Spring Boot | 3.5.14 | 应用框架 |
| MyBatis Spring Boot Starter | 3.0.5 | 持久层 |
| Druid | 1.2.28 | 连接池 |
| DmJdbcDriver18 | 8.1.3.140 | 达梦 DM8 JDBC 驱动 |
| clickhouse-jdbc / clickhouse-native-jdbc-shaded | 0.8.6 / 2.7.1 | ClickHouse 时序查询（只读） |
| spring-kafka | — | Kafka Outbox/Inbox |
| Lettuce (Redis) | — | 采样缓存、幂等加速 |
| AWS SDK v2 S3 | 2.54.7 | 对象存储（MinIO 兼容） |
| jjwt | 0.9.1 | JWT 认证 |
| kaptcha | 2.3.3 | 验证码 |
| docx4j | 17.0.4 | WordprocessingML 报告生成 |
| jsoup | 1.23.2 | HTML 净化 |
| Playwright | 1.61.0 | HTML→PDF（跳过浏览器下载，依赖外部浏览器） |
| Puppeteer-core | ^21.0.0 | Node 渲染服务图表渲染 |
| zstd-jni | 1.5.6-9 | ClickHouse native 传输必需 |
| oshi | 7.3.0 | 系统信息 |
| yauaa | 8.1.1 | 用户代理解析 |

---

## 10. 测试与验证

### 10.1 现状

- **295 个测试类**，包结构与 `src/main` 业务域**一一镜像**（覆盖全部 11 个业务域）。
- 测试依赖：`spring-boot-starter-test`、**Testcontainers**（Kafka 等）、H2（test scope）、JUnit 5。
- 有契约一致性测试（`ErrorCodeCatalogTest`、`BusinessPermissionContractTest`、`KafkaContractTest`、`FormulaGraphContractTest`）。
- **`error-codes.md` 与代码完全一致**（160 条错误码，0 缺失、0 多余、0 字段不匹配）—— 有一条契约测试守着。

### 10.2 缺口

| 缺口 | 影响 |
|---|---|
| **`node-renderer/` 零测试**（无 `*.test.js` / `*.spec.js`，`package.json` 无 test 脚本） | 环境依赖最强的组件（Node HTTP 路径）自动化覆盖最低 |
| **两个基于 fixture 的报告测试默认跳过**（`assumeTrue(fixtureDirectory != null && reportId != null)`） | 复杂报告的端到端验收在 CI 中从不执行 |
| **Playwright PDF 路径依赖本机浏览器**，无容器化测试 | 换机器就失败 |
| **4 个 0 字节文件**（1 主源码 + 3 测试） | 见 §11 坑 #20 |

### 10.3 本次执行了什么

**未执行任何构建、测试、迁移或写操作**（遵循只读分析原则）。建议执行命令：

```bash
# 编译打包（跳过测试）
mvn -pl flight-platform-app -am -DskipTests package

# 运行测试（需 Redis，端口加 -Dspring.data.redis.port=6379）
mvn -pl flight-platform-app -am test -Dspring.data.redis.port=6379
```

---

## 11. 风险与开放问题

共 **25 条**（高危 7 条、中危 11 条、低危/开放 7 条）。选取高危与核心中危如下：

| # | 风险 | 证据 | 影响 | 建议 |
|---|---|---|---|---|
| R1 | **任务归属未强制，可跨用户读取判读结果** | `mapper/task/TaskManagementMapper.xml:120-125`（`selectOwnedDetail` 只有 `WHERE t.task_id`）；Java 侧 `task/service/TaskQueryService.java:51-53` 的 `owned(id, owner)` 收了 owner 却不用 | `GET /judgement-tasks`、`/{id}`、`/{id}/result`、`/{id}/analysis-result`、`/{id}/drawing-context` 均可跨用户读取 | SQL 补 `created_by = #{owner}`；Service 层真正使用 owner |
| R2 | **操作审计是空壳** | `mapper/task/JudgementTaskMapper.xml:136-141`（`auditTrail` 的 SQL 是 `SELECT ... FROM dual WHERE 1 = 0`，恒空）；`:143-147`（`audit` 只 `SET updated_time`，actor/operation/reason 全丢） | 取消 / 强停无任何操作留痕，直接削弱可审计性 | 落库真实审计记录 |
| R3 | **`dataType` 语义冲突，发现的飞参全部判为非法** | SQL 过滤并别名 `r.data_type = 0`（`mapper/clickhouse/ClickHouseFlightDataCatalogMapper.xml:35,47`）→ 适配器原样拷贝（`ClickHouseFlightDataCatalogAdapter.java:51`）→ 解析器要求 `dataType == 1`（`FlightDataRegistrationResolver.java:99-101`） | 每一行发现结果都在解析阶段短路为 `METADATA_INVALID` | 确认外部 `data_type=0` 与内部 FP 类型 `1` 的映射并显式转换 |
| R11 | **两个 `@Anonymous` 写端点** | `StaticGraphController.java:36`（写 S3 + 污染 `fp_graph`）与 `ReportDebugController.java:33`（触发完整报告生成）**无需鉴权** |  unauthorized 写操作 | 加鉴权或移除 `@Anonymous` |
| R12 | **JWT 密钥硬编码 + SYSDBA 口令入库** | `application.yml:131`（`token.secret: abcdefghijklmnopqrstuvwxyz`，26 字母，可离线爆破）；`application-dev.yml` 存达梦 SYSDBA、MinIO、ClickHouse 明文口令 | 可离线爆破 JWT；生产凭据泄露 | 改为环境变量占位符并轮换 |
| R24 | **prod profile 下 S3 必然启动失败** | `application-prod.yml` 只覆写浏览器路径，`flight.s3.bucket` 仍为空 ⇒ `S3BucketConfiguration` 抛 `IllegalStateException` | 生产环境无法启动 | 补全 prod 下的 `flight.s3` 配置 |
| R25 | **Druid 监控控制台在 prod 开放** | `/druid/*` + `flight/123456` + `allow:` 为空（任意来源可访问） | 监控接口泄露 | 生产环境限制访问 |

其余中危与低危/开放问题（跨用户读取、审计空壳、Keyword 三表无种子、Node 渲染 page 泄漏、唯一索引被注释、异步链路不可达、死信工厂未接线、文档漂移、浏览器二进制入 git、demo 样例漂移、0 字节文件、图表输出策略不支持、三套浮点语义、单位不换算）详见 `01-*` 工作流程与标准 §10。

---

## 12. 推荐下一步

### 短期（安全与正确性，1 周内）

1. 修复任务归属过滤（`TaskQueryService` + `TaskManagementMapper.xml`），阻止跨用户读取。
2. 补全操作审计落库，废弃恒空 SQL。
3. 给两个 `@Anonymous` 写端点加鉴权或移除。
4. JWT 密钥与生产凭据改为环境变量占位符并轮换。

### 中期（可运维性与一致性，1 个月内）

1. 补全 `flight.s3` 在 prod 下的配置，避免启动失败。
2. 生产环境限制 Druid 监控控制台访问。
3. 为 `node-renderer/` 补全自动化测试。
4. 将 `chart-renderer.js` 的前后端同步机制纳入构建流程（当前是手工复制）。

### 长期（架构性改进）

1. 评估是否引入统一 BaseEntity / BaseRepository 约定，消除审计字段三套并存。
2. 评估 ClickHouse 只读约束是否需放宽为受控写入。
3. 考虑将异步任务链路从 `@Scheduled` 轮询迁移为可 HTTP 触发的可靠流水线。

### 验证命令（尚未执行）

```bash
mvn -pl flight-platform-app -am -DskipTests package
mvn -pl flight-platform-app -am test -Dspring.data.redis.port=6379
```

---

## 13. 新人阅读路径

1. `pom.xml` + `FlightPlatformApplication.java` → 模块边界与启动入口
2. `application.yml` 的 `flight.*` 段 → 所有业务开关与外部依赖位置
3. `resources/contracts/README.md` → 核心数据契约（注意 4 个 schema 均已存在，与旧版“2 个”描述不符）
4. **判读主线**（推荐主线）：
   - `business/judgement/model/JudgementModelBuilder.java` → 快照如何编译成逻辑树
   - `business/judgement/execution/LogicTreeRunner.java` → 窗口如何逐级传递
   - `business/judgement/execution/LogicResultAggregator.java` → 三值结论与窗口交并
   - `business/keywords/execution/executor/ComparisonKeywordEvaluator.java` → 单个 Keyword 怎么求值
   - `business/task/service/TaskStateService.java` → 状态机与租约
5. **数据接入主线**：`business/ingestion/service/FlightDataDiscoveryService.java` → `DefaultFlightDataAccessService.java`
6. **可靠事件主线**：`business/messaging/outbox/publishing/OutboxEventPublisher.java` → `business/messaging/inbox/admission/InboxAdmissionService.java`
7. **报告主线**：`business/report/generation/ResultReadyInboxHandler.java` → `ReportArchiveService` → `business/render/ChartRenderService.java`
8. **横向辅助读**：`business/icd/`（机型定义）、`business/template/`（模板与导入）、`business/authorization/`（资源授权）
9. `error-codes.md` → 建立错误语义全局观（有契约测试守着，与代码一致）
10. `node-renderer/GRAPH_GENERATION_CONTRACT.md` → Java ↔ Node 渲染契约（**先跳过 README**，它已过期）

**首次上手建议**：先跑 `/judgement-tasks/execute`（同步判读）走通一条链路，它不依赖 Kafka 与 Node 渲染服务。

---

## 附录

### 14.1 关键文件索引

| 类别 | 路径 |
|---|---|
| 聚合 POM | `pom.xml` |
| 启动类 | `flight-platform-app/src/main/java/com/huike/flight/platform/FlightPlatformApplication.java` |
| 主配置 | `flight-platform-app/src/main/resources/application.yml` |
| 环境配置 | `application-dev.yml` / `application-prod.yml` / `application-druid.yml` |
| 结果契约 | `flight-platform-app/src/main/resources/contracts/`（4 个 schema） |
| 报告模板 | `flight-platform-app/src/main/resources/templates/report/default-report.html` |
| MyBatis 映射 | `flight-platform-app/src/main/resources/mapper/**/*Mapper.xml` |
| Redis Lua | `flight-platform-app/src/main/resources/redis/ingestion/*.lua` |
| 建表脚本 | `sql/flight-dm.sql`（58 表）、`sql/clickhouse.sql`、`sql/ruoyi.sql` |
| 迁移脚本 | `sql/migration-air-plane-code-unique.sql`、`migration-block-redundancy-group.sql`（**已过时**）、`migration-business-permission.sql` |
| 错误码目录 | `error-codes.md`（160 条，与代码一致） |
| Node 渲染服务 | `node-renderer/`（契约文档 `GRAPH_GENERATION_CONTRACT.md` 有效，`README.md` 已过期） |
| 样例载荷 | `output/report-complex-demo/`（**已漂移**） |

### 14.2 业务域与控制器对照

| 业务域 | Controller | 权限码 |
|---|---|---|
| graph | `GraphController`、`GraphFormulaController`、`FormulaLibraryController`、`PlotVariableController`、`StaticGraphController` | `graph:view` / `graph:edit` |
| icd | `AirPlaneController`、`IcdVersionController`、`IcdBlockController`、`IcdFieldController`、`IcdTreeController`、`IcdRedundancyGroupController`、`IcdRedundancyMemberController`、`IcdBlockRedundancyGroupController` | `icd:view` / `icd:manage` / `icd:redundancy:manage` |
| ingestion | 飞参发现与对账相关 Controller | 飞参相关权限 |
| judgement | 判读任务执行与结果查询 Controller | 判读相关权限 |
| task | 任务编排与查询 Controller | 任务相关权限 |
| report | 报告生成与下载 Controller（含 `ReportDebugController` `@Anonymous`） | 报告相关权限 |
| template | 模板版本与导入 Controller | 模板相关权限 |
| keywords | 关键字管理 Controller | 关键字相关权限 |
| messaging | 消息链路相关端点 | 消息相关权限 |
| render | 渲染服务桥接（无直接业务 Controller） | 无 |
| authorization | 资源授权管理 Controller | 授权相关权限 |

### 14.3 权限体系（两套并行）

1. **Spring Security 层**：`@PreAuthorize` 注解，基于 `sys_menu.perms` 与 `sys_role_menu`；非 admin 账号需重新登录 token 才生效；admin 走 `*:*:*` 特殊通道，**权限验收必须用非 admin 账号复测**。
2. **业务资源授权层**：基于业务资源授权注解 + AOP 切面，控制具体资源（如 ICD 版本、模板、任务）的读写权限。

### 14.4 分析方法说明

本报告基于对 `backend/flight-platform-backend/` 仓库源码、Maven POM、Spring 配置、MyBatis XML 及现有 `docs/` 分析文档的只读检查完成。未执行任何构建、测试、迁移或写操作。文档与代码冲突时以代码为准。
