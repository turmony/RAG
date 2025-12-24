版本: 1.0
语言: 中文
定位: 企业级 ToB 智能知识库平台（RAG 架构）——后端优先，前端随后实现。目标用于 Java 后端校招/面试展示，强调架构、工程化与可测试性。

一、项目整体技术架构说明（概览）
- 应用类型：单体应用（模块化）——便于校招演示与面试讲解，同时各模块可拆分成独立服务。
- 核心职责：
  - Auth 模块：用户认证（JWT）、权限（RBAC）、会话管理。
  - User 模块：用户与组织信息。
  - Document 模块：文档导入/解析/元数据管理/版本管理。
  - Ingest 模块：数据清洗、分段、Embedding 生成（bge-m3）、向量写入（Milvus）。
  - VectorStore 层：Milvus 封装 + Redis 缓存层（常用向量/ID 缓存）。
  - Retrieval 层：基于向量检索 + 关键词检索的混合检索封装。
  - RAG 服务层：Prompt 构造、LLM 调用（Ollama）、答案合成、Re-rank（Phase3）。
  - API 层：RESTful API（OpenAPI/Swagger），契约优先。
  - Infra 层：Ollama 客户端、Milvus 客户端、Redis 客户端、RabbitMQ 任务队列。
  - Admin/Operator：管理界面与运维接口（仅用于演示/内部管理）。
- 中间件：MySQL 8（业务数据与元数据）、Redis（缓存/短期会话）、Milvus（向量索引）、RabbitMQ（异步任务/批量 ingestion）。
- 部署方式：开发阶段使用 Docker Compose（所有中间件必须 Docker 化），生产可选 k8s。

二、完整项目目录结构设计（后端 + 前端 — 详细到 package/module 级别）
- 根目录（monorepo / 单体）
  - backend/ (Maven multi-module)
    - pom.xml (父 POM)
    - backend-app/ (Spring Boot 可执行模块)
      - src/main/java/
        - com.example.rag
          - config                  // 全局配置（安全、DB、Client 配置）
            - SecurityConfig
            - OllamaClientConfig
            - MilvusConfig
          - controller              // REST 控制器（仅处理参数/响应与权限）
            - AuthController
            - DocumentController
            - SearchController
            - AdminController
          - dto                     // API DTO/VO（请求/响应契约）
            - request
            - response
          - service                 // 业务逻辑（无外部依赖细节）
            - AuthService
            - UserService
            - DocumentService
            - IngestService
            - RetrievalService
            - RagService
          - repository              // 持久层（MyBatis/JPA/手写 JDBC）
            - mysql
            - milvus (仅封装，不直接做业务逻辑)
          - domain/model            // 领域模型（实体）
            - User
            - Document
            - EmbeddingRecord
          - infra                   // 外部系统封装（Ollama, Milvus, Redis, RabbitMQ）
            - client
              - OllamaHttpClient
              - MilvusClient
              - RedisClient
              - RabbitMqClient
            - persistence           // DB helpers, mappers
          - util                    // 工具与通用函数（分页、时间、序列化）
          - exception               // 统一异常与错误码
          - security                // JWT、RBAC 实现细节（仅接口与策略）
          - docs                    // OpenAPI / Swagger 配置与 API 文档
      - src/test/java/            // 单元与集成测试
    - backend-domain/ (公共模块)
      - src/main/java/com.example.rag.domain
        - 公共 DTO、CommonError、Constants
    - backend-infra/ (可选，外部客户端与实现)
      - src/main/java/com.example.rag.infra
        - OllamaHttpClientImpl (封装、重试、超时)
        - MilvusClientImpl
    - db/
      - migrations/              // DDL 与 Flyway/Liquibase 脚本
      - schema.sql
  - frontend/
    - web/                      // 前端（可选技术栈：React + TypeScript）
      - src/
        - pages/
          - Login
          - Search
          - DocumentDetail
          - Admin
        - services/             // 与后端契约一致的 API 调用封装
        - components/
        - store/
  - infra/                      // Docker Compose 与中间件配置
    - docker-compose.yml
    - olama/                    // Ollama local config notes
    - milvus/
    - mysql/
  - docs/
    - design/
    - decisions/
  - ci/
    - pipeline.yaml

三、每一层 / 每一个包的职责说明（简述）
- `config`: 应该只包含框架配置与客户端 Bean 定义，不放业务逻辑。
- `controller`: 参数校验、权限注解、错误映射与响应规范；委托给 `service`。
- `service`: 纯业务逻辑，依赖 repository 与 infra client；保证幂等与事务边界清晰。
- `repository`: 封装所有数据库交互（SQL 与 ORM）；只返回领域实体或基础 DTO。
- `infra/client`: 所有外部系统调用在此层封装，提供统一的超时、重试、熔断策略。
- `dto`: API 层的契约，任何字段变更必须对应 OpenAPI 文档更新。
- `exception`: 统一异常处理与错误码映射，供全局异常处理器使用。
- `security`: JWT 解析、Token 刷新策略、RBAC 权限决策器。
- `docs`: 自动生成的 API 文档与手写设计说明。

四、后端开发必须严格遵循的顺序（每步均以相应测试锚点结尾）
Phase 0（初始化）
Step 0.1: 创建项目基础结构与父 POM（Maven），建立模块划分与 CI skeleton。
  - 产出：父 POM、模块目录、基础配置文件、Docker Compose skeleton。
  - 验收：👉「请完成测试文档中 1.1 测试」

Step 0.2: 配置基础中间件 Docker Compose（MySQL/Redis/Milvus/RabbitMQ/Ollama stub）。
  - 产出：`infra/docker-compose.yml`、各中间件示例 env 与端口映射。
  - 验收：👉「请完成测试文档中 6.1 测试」

Phase 1：项目架构与目录设计（契约优先）
Step 1.1: 明确模块与包（目录结构），在 `docs/design` 中写明每个模块职责与边界。
  - 产出：目录清单、模块职责文档。
  - 验收：👉「请完成测试文档中 1.2 测试」

Phase 2：数据库设计（表结构先行）
Step 2.1: 设计并提交 DDL（MySQL），包括用户表、文档元数据表、embedding 记录表、权限表、审计表。
  - 产出：`db/migrations/xxx_create_schema.sql`（可在项目中使用 Flyway/Liquibase）。
  - 验收：👉「请完成测试文档中 2.1 测试」

Step 2.2: 设计索引、外键、分区/分表策略（如需）与数据迁移/回滚脚本。
  - 产出：索引与迁移脚本、性能预估说明。
  - 验收：👉「请完成测试文档中 2.2 测试」

Phase 3：API 接口设计（契约先行）
Step 3.1: 以 OpenAPI/Swagger 定义所有对外 REST API（Auth, User, Document, Search, Admin）。
  - 产出：`docs/api/openapi.yaml`，包含示例请求/响应与错误码表。
  - 验收：👉「请完成测试文档中 3.1 测试」

Step 3.2: 定义统一响应结构（成功/错误），并在全局异常处理器中实现。
  - 产出：统一响应 DTO、错误码表。
  - 验收：👉「请完成测试文档中 3.2 测试」

Phase 4：核心业务功能实现（先关键路径）
Step 4.1: 实现认证（JWT）与用户登录/注册接口，确保 Token 签发/验证与刷新机制。
  - 产出：AuthController、AuthService、JWT 策略。
  - 验收：👉「请完成测试文档中 4.1 测试」

Step 4.2: 实现 RBAC 基础（角色、权限关联、注解/拦截器实现）。
  - 产出：权限模型、注解或拦截器。
  - 验收：👉「请完成测试文档中 4.2 测试」

Step 4.3: 实现文档导入（Ingest）流程：文件解析 → 分段 → 生成 Embedding（bge-m3）→ 写入 Milvus。
  - 产出：IngestService、异步任务（RabbitMQ）、Embedding 写入逻辑。
  - 验收：👉「请完成测试文档中 4.3 测试」

Step 4.4: 实现基础检索与回答（Phase1 基础 RAG）：向量检索 + prompt 发送给 LLM（Ollama）并返回答案。
  - 产出：SearchController、RetrievalService、Ollama 调用封装。
  - 验收：👉「请完成测试文档中 5.1 测试」

Phase 5：RAG 能力分阶段演进（按版本推进）
Step 5.1 (Phase2): 混合检索（向量 + 关键词），实现再排序器（简单权重融合）。
  - 产出：混合检索策略实现、配置化权重项。
  - 验收：👉「请完成测试文档中 5.2 测试」

Step 5.2 (Phase3): 引入 Re-rank（利用 LLM 对候选答案/片段排序），并保证可回滚到 Phase2。
  - 产出：Re-rank 模块、性能评估与成本估算。
  - 验收：👉「请完成测试文档中 5.3 测试」

Step 5.3 (Phase4): 多轮对话能力（会话管理、上下文窗口管理、检索+检索重写）。
  - 产出：ConversationManager、上下文保全策略（截断/摘要）。
  - 验收：👉「请完成测试文档中 5.4 测试」

Phase 6：运维与演示准备
Step 6.1: 测试并确认中间件 Docker Compose 能被复现（包含 Ollama 运行说明）。
  - 产出：`infra/docker-compose.yml` 可在开发机一键启动所有必要服务。
  - 验收：👉「请完成测试文档中 6.1 测试」

Step 6.2: 编写部署与本地运行文档（包含环境变量、端口、资源建议）。
  - 产出：`docs/deploy.md`
  - 验收：👉「请完成测试文档中 6.2 测试」

五、前端开发（后端完成必要接口后进行） — 阶段与步骤
原则：后端先做好 API 契约（OpenAPI），前端以契约为准实现。
Phase F1：Login & 基础页面
- Step F1.1: 使用契约实现 `Login` 与鉴权流程（本地存储 Token）。
  - 验收：👉「请完成测试文档中 7.1 测试」

Phase F2：搜索与结果呈现
- Step F2.1: 实现搜索输入、结果列表、详情页，并匹配后端返回的数据结构。
  - 验收：👉「请完成测试文档中 7.2 测试」

六、每一步的交付物（示例）
- DDL SQL 文件（db/migrations）
- OpenAPI 文件（docs/api/openapi.yaml）
- 模块 README（模块职责、使用示例、测试锚点）
- CI 配置（可运行单元/集成测试）
- Docker Compose（可复现中间件）

七、测试与验收流程（与测试文档严格对应）
- 每个开发步骤末尾必须引用测试编号作为唯一验收锚点（参见上文步骤末尾“👉「请完成测试文档中 X.Y 测试」”）。
- 所有测试通过才可合并对应功能分支到主分支。

八、面试讲解要点（给提交者）
- 能清晰讲述为何模块化（可替换 Embedding/VectorStore/Ollama）、为何先 DB 再接口再实现、关键设计权衡（性能 vs 成本 vs 简单性）。
- 对每个核心模块能展示：接口契约、错误处理策略、扩展点。


