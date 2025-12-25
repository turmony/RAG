# Java企业级智能知识库平台 - 开发文档

## 项目整体架构说明（MVP 视角）

本项目是一个基于 RAG（Retrieval-Augmented Generation）技术的企业级智能知识库平台，采用最小可用产品（MVP）设计理念：

### 核心组件
- **知识库管理**: 文档上传、存储、向量化
- **向量检索**: 基于 Milvus 的语义相似度搜索
- **智能问答**: 结合检索结果与 Ollama 大模型生成回答
- **基础权限**: 用户管理和文档权限控制

### 技术架构
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Backend       │    │   Middleware    │
│   (Vue.js)      │◄──►│   (Spring Boot) │◄──►│   (Docker)      │
│   Windows       │    │   Windows       │    │   WSL Ubuntu    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   Ollama        │
                       │   (Qwen2.5-14B) │
                       │   Windows       │
                       └─────────────────┘
```

### 数据流
1. 用户上传文档 → 存储到 MySQL + 向量化存储到 Milvus
2. 用户提问 → 向量检索相关文档片段
3. 检索结果 + 问题 → 发送给 Ollama 生成回答

## 完整项目目录结构

```
rag-knowledge-platform/
├── backend/                          # Spring Boot 后端
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/rag/platform/
│   │   │   │   ├── config/           # 配置类
│   │   │   │   │   ├── DatabaseConfig.java
│   │   │   │   │   ├── MilvusConfig.java
│   │   │   │   │   ├── OllamaConfig.java
│   │   │   │   │   └── RedisConfig.java
│   │   │   │   ├── controller/       # REST API 控制器
│   │   │   │   │   ├── DocumentController.java
│   │   │   │   │   ├── QueryController.java
│   │   │   │   │   └── UserController.java
│   │   │   │   ├── service/          # 业务逻辑服务
│   │   │   │   │   ├── DocumentService.java
│   │   │   │   │   ├── RagService.java
│   │   │   │   │   ├── VectorService.java
│   │   │   │   │   └── EmbeddingService.java
│   │   │   │   ├── entity/           # JPA 实体
│   │   │   │   │   ├── Document.java
│   │   │   │   │   ├── User.java
│   │   │   │   │   └── QueryLog.java
│   │   │   │   ├── repository/       # 数据访问层
│   │   │   │   │   ├── DocumentRepository.java
│   │   │   │   │   └── UserRepository.java
│   │   │   │   └── dto/              # 数据传输对象
│   │   │   │       ├── DocumentDTO.java
│   │   │   │       └── QueryRequest.java
│   │   │   └── resources/
│   │   │       ├── application.yml
│   │   │       └── db/migration/     # Flyway 迁移脚本
│   │   └── test/                     # 测试代码
│   └── pom.xml
├── frontend/                         # Vue.js 前端
│   ├── src/
│   │   ├── components/
│   │   │   ├── DocumentUpload.vue
│   │   │   ├── QueryInterface.vue
│   │   │   └── DocumentList.vue
│   │   ├── views/
│   │   │   ├── Home.vue
│   │   │   └── Admin.vue
│   │   ├── router/
│   │   └── store/
│   ├── public/
│   └── package.json
├── docker/                           # Docker 相关配置
│   ├── docker-compose.yml
│   └── middleware/
│       ├── mysql/
│       ├── redis/
│       ├── milvus/
│       └── rabbitmq/
├── docs/                            # 文档
│   ├── api.md
│   └── deployment.md
└── README.md
```

## 运行环境与职责边界说明

### Windows 11（主要开发环境）
- **职责**: 代码编写、Spring Boot 服务运行、Ollama 模型部署
- **工具**: IDEA/VS Code、Java 17、Maven、Ollama
- **网络**: 可访问 WSL Ubuntu 的 Docker 服务

### WSL2 Ubuntu（中间件运行环境）
- **职责**: Docker 引擎运行、所有中间件容器部署
- **中间件**: MySQL、Redis、Milvus、RabbitMQ
- **网络**: 与 Windows 共享网络，可被 Windows 访问

### 关键约束
- Ollama 必须运行在 Windows（不支持 Docker 部署）
- 所有中间件必须通过 Docker 在 WSL Ubuntu 运行
- 禁止使用 Docker Desktop
- Windows 代码通过网络访问 WSL 的中间件服务

## 后端开发阶段

### 0️⃣ 阶段 0：WSL Ubuntu 中间件部署（开发第一步）

#### 步骤 0.1：安装 Docker Engine
在 WSL Ubuntu 中执行以下命令：

```bash
# 更新包索引
sudo apt update

# 安装必要的包
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# 添加 Docker 的官方 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 设置稳定仓库
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动 Docker 服务
sudo systemctl start docker
sudo systemctl enable docker

# 将当前用户添加到 docker 组（避免每次使用 sudo）
sudo usermod -aG docker $USER
```

#### 步骤 0.2：创建项目网络
```bash
# 创建 Docker 网络（注意：Docker Compose 会自动创建 rag_rag-network）
# docker network create rag-network  # 不需要手动创建
```

#### 步骤 0.3：部署 MySQL
```bash
docker run -d \
  --name rag-mysql \
  --network rag_rag-network \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=rag_db \
  -e MYSQL_USER=rag_user \
  -e MYSQL_PASSWORD=rag_pass \
  -p 3307:3306 \
  mysql:8.0
```

#### 步骤 0.4：部署 Redis
```bash
docker run -d \
  --name rag-redis \
  --network rag_rag-network \
  -p 6380:6379 \
  redis:7-alpine
```

#### 步骤 0.5：部署 Milvus（向量数据库）
```bash
# 创建数据目录
mkdir -p ~/milvus-data

# 运行 Milvus standalone（推荐使用 docker-compose.yml 中的完整配置）
docker run -d \
  --name rag-milvus \
  --network rag_rag-network \
  -p 19530:19530 \
  -p 9091:9091 \
  -v ~/milvus-data:/var/lib/milvus \
  milvusdb/milvus:v2.6.7 \
  milvus run standalone
```

#### 步骤 0.6：使用 Docker Compose 部署所有服务

项目根目录已包含 `docker-compose.yml` 文件，包含所有中间件服务的完整配置：

```yaml
# docker-compose.yml 配置包含：
# - MySQL (端口 3307:3306)
# - Redis (端口 6380:6379)
# - RabbitMQ (端口 5673:5672, 15673:15672)
# - Milvus (端口 19530:19530, 9091:9091)
```

**启动所有服务：**
```bash
# 在项目根目录执行
docker compose up -d
```

**停止所有服务：**
```bash
docker compose down
```

**查看服务状态：**
```bash
docker compose ps
```

👉「请完成测试文档中 0.1-0.6 测试」

### 1️⃣ 阶段 1：项目架构与目录设计

#### 步骤 1.1：创建 Spring Boot 项目
在 Windows 中使用 Spring Initializr 创建项目：

```bash
# 使用 curl 创建项目（或者使用 IDEA 的 Spring Initializr）
curl -G https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,mysql,redis,data-redis,validation \
  -d javaVersion=17 \
  -d type=maven-project \
  -d groupId=com.rag \
  -d artifactId=platform \
  -d name=rag-platform \
  -d packageName=com.rag.platform \
  --output rag-platform.zip

# 解压项目
unzip rag-platform.zip
cd rag-platform
```

#### 步骤 1.2：配置 application.yml
创建 `src/main/resources/application.yml`：

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/rag_db?useSSL=false&serverTimezone=UTC
    username: rag_user
    password: rag_pass
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

  redis:
    host: localhost
    port: 6379

milvus:
  host: localhost
  port: 19530

ollama:
  base-url: http://localhost:11434
  model: qwen2.5:14b
  embedding-model: bge-m3

logging:
  level:
    com.rag.platform: DEBUG
```

#### 步骤 1.3：创建基础包结构
按照目录结构创建所有必要的包和类文件。

👉「请完成测试文档中 1.1-1.3 测试」

### 2️⃣ 阶段 2：数据库设计（最小必要表）

#### 步骤 2.1：创建 User 实体
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String email;
    private LocalDateTime createTime;

    // getters and setters
}
```

#### 步骤 2.2：创建 Document 实体
```java
@Entity
@Table(name = "documents")
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    private String filePath;
    private Long fileSize;
    private String fileType;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User uploader;

    private LocalDateTime uploadTime;
    private LocalDateTime updateTime;

    // getters and setters
}
```

#### 步骤 2.3：创建 QueryLog 实体
```java
@Entity
@Table(name = "query_logs")
public class QueryLog {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(columnDefinition = "TEXT")
    private String question;

    @Column(columnDefinition = "TEXT")
    private String answer;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    private LocalDateTime queryTime;
    private Long responseTime; // 响应时间（毫秒）

    // getters and setters
}
```

👉「请完成测试文档中 2.1-2.3 测试」

### 3️⃣ 阶段 3：API 接口设计（满足前端与 RAG 即可）

#### 步骤 3.1：创建 UserController
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/register")
    public ResponseEntity<User> register(@RequestBody User user) {
        return ResponseEntity.ok(userService.register(user));
    }

    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {
        String token = userService.login(request.getUsername(), request.getPassword());
        return ResponseEntity.ok(token);
    }
}
```

#### 步骤 3.2：创建 DocumentController
```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    @Autowired
    private DocumentService documentService;

    @PostMapping("/upload")
    public ResponseEntity<Document> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam("title") String title,
            @RequestHeader("Authorization") String token) {

        Long userId = extractUserIdFromToken(token);
        Document document = documentService.uploadDocument(file, title, userId);
        return ResponseEntity.ok(document);
    }

    @GetMapping
    public ResponseEntity<List<DocumentDTO>> getUserDocuments(@RequestHeader("Authorization") String token) {
        Long userId = extractUserIdFromToken(token);
        return ResponseEntity.ok(documentService.getUserDocuments(userId));
    }
}
```

#### 步骤 3.3：创建 QueryController
```java
@RestController
@RequestMapping("/api/query")
public class QueryController {

    @Autowired
    private RagService ragService;

    @PostMapping
    public ResponseEntity<QueryResponse> query(
            @RequestBody QueryRequest request,
            @RequestHeader("Authorization") String token) {

        Long userId = extractUserIdFromToken(token);
        QueryResponse response = ragService.processQuery(request.getQuestion(), userId);
        return ResponseEntity.ok(response);
    }
}
```

👉「请完成测试文档中 3.1-3.3 测试」

### 4️⃣ 阶段 4：核心业务功能实现

#### 步骤 4.1：实现 UserService
```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private JwtUtil jwtUtil;

    public User register(User user) {
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        user.setCreateTime(LocalDateTime.now());
        return userRepository.save(user);
    }

    public String login(String username, String password) {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new RuntimeException("User not found"));

        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new RuntimeException("Invalid password");
        }

        return jwtUtil.generateToken(user.getId());
    }
}
```

#### 步骤 4.2：实现 DocumentService
```java
@Service
public class DocumentService {

    @Autowired
    private DocumentRepository documentRepository;

    @Autowired
    private VectorService vectorService;

    @Autowired
    private FileStorageService fileStorageService;

    @Transactional
    public Document uploadDocument(MultipartFile file, String title, Long userId) {
        // 1. 保存文件到本地
        String filePath = fileStorageService.store(file);

        // 2. 读取文件内容
        String content = extractContent(file);

        // 3. 创建文档实体
        Document document = new Document();
        document.setTitle(title);
        document.setContent(content);
        document.setFilePath(filePath);
        document.setFileSize(file.getSize());
        document.setFileType(file.getContentType());
        document.setUploader(userRepository.findById(userId).orElseThrow());
        document.setUploadTime(LocalDateTime.now());

        Document savedDoc = documentRepository.save(document);

        // 4. 向量化存储
        vectorService.storeDocumentVectors(savedDoc);

        return savedDoc;
    }

    public List<DocumentDTO> getUserDocuments(Long userId) {
        return documentRepository.findByUploaderId(userId)
                .stream()
                .map(this::convertToDTO)
                .collect(Collectors.toList());
    }
}
```

👉「请完成测试文档中 4.1-4.2 测试」

### 5️⃣ 阶段 5：RAG 最小可用能力实现

#### 步骤 5.1：实现 EmbeddingService（调用 Ollama）
```java
@Service
public class EmbeddingService {

    @Autowired
    private OllamaConfig ollamaConfig;

    public List<Double> generateEmbedding(String text) {
        try {
            String url = ollamaConfig.getBaseUrl() + "/api/embeddings";

            Map<String, Object> request = Map.of(
                "model", ollamaConfig.getEmbeddingModel(),
                "prompt", text
            );

            RestTemplate restTemplate = new RestTemplate();
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            HttpEntity<Map<String, Object>> entity = new HttpEntity<>(request, headers);
            ResponseEntity<Map> response = restTemplate.postForEntity(url, entity, Map.class);

            @SuppressWarnings("unchecked")
            List<Double> embedding = (List<Double>) response.getBody().get("embedding");
            return embedding;

        } catch (Exception e) {
            throw new RuntimeException("Failed to generate embedding", e);
        }
    }
}
```

#### 步骤 5.2：实现 VectorService（Milvus 操作）
```java
@Service
public class VectorService {

    @Autowired
    private MilvusConfig milvusConfig;

    @Autowired
    private EmbeddingService embeddingService;

    private MilvusServiceClient client;

    @PostConstruct
    public void init() {
        client = new MilvusServiceClient(ConnectParam.newBuilder()
                .withHost(milvusConfig.getHost())
                .withPort(milvusConfig.getPort())
                .build());
    }

    public void storeDocumentVectors(Document document) {
        // 分割文档为块
        List<String> chunks = splitDocument(document.getContent());

        // 为每个块生成向量并存储
        for (int i = 0; i < chunks.size(); i++) {
            List<Double> embedding = embeddingService.generateEmbedding(chunks.get(i));

            // 插入向量到 Milvus
            insertVector(document.getId(), i, embedding, chunks.get(i));
        }
    }

    public List<String> searchRelevantChunks(String query, int topK) {
        List<Double> queryEmbedding = embeddingService.generateEmbedding(query);

        // 在 Milvus 中搜索相似向量
        SearchParam searchParam = SearchParam.newBuilder()
                .withCollectionName("documents")
                .withVectorFieldName("embedding")
                .withVectors(List.of(queryEmbedding))
                .withTopK(topK)
                .build();

        R<SearchResults> response = client.search(searchParam);

        // 提取相关文档块
        return extractRelevantChunks(response);
    }
}
```

#### 步骤 5.3：实现 RagService（核心 RAG 逻辑）
```java
@Service
public class RagService {

    @Autowired
    private VectorService vectorService;

    @Autowired
    private EmbeddingService embeddingService;

    @Autowired
    private QueryLogRepository queryLogRepository;

    @Autowired
    private OllamaConfig ollamaConfig;

    public QueryResponse processQuery(String question, Long userId) {
        long startTime = System.currentTimeMillis();

        try {
            // 1. 检索相关文档片段
            List<String> relevantChunks = vectorService.searchRelevantChunks(question, 3);

            // 2. 构建上下文
            String context = String.join("\n\n", relevantChunks);

            // 3. 构建提示词
            String prompt = buildPrompt(question, context);

            // 4. 调用 Ollama 生成回答
            String answer = callOllamaGenerate(prompt);

            // 5. 记录查询日志
            saveQueryLog(question, answer, userId, System.currentTimeMillis() - startTime);

            return new QueryResponse(answer, relevantChunks.size());

        } catch (Exception e) {
            throw new RuntimeException("Failed to process query", e);
        }
    }

    private String buildPrompt(String question, String context) {
        return String.format(
            "基于以下上下文信息回答问题。如果上下文中没有相关信息，请说'根据提供的知识库内容，无法回答此问题'。\n\n" +
            "上下文：\n%s\n\n" +
            "问题：%s\n\n" +
            "回答：",
            context, question
        );
    }

    private String callOllamaGenerate(String prompt) {
        try {
            String url = ollamaConfig.getBaseUrl() + "/api/generate";

            Map<String, Object> request = Map.of(
                "model", ollamaConfig.getModel(),
                "prompt", prompt,
                "stream", false
            );

            RestTemplate restTemplate = new RestTemplate();
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);

            HttpEntity<Map<String, Object>> entity = new HttpEntity<>(request, headers);
            ResponseEntity<Map> response = restTemplate.postForEntity(url, entity, Map.class);

            return (String) response.getBody().get("response");

        } catch (Exception e) {
            throw new RuntimeException("Failed to generate answer", e);
        }
    }
}
```

👉「请完成测试文档中 5.1-5.3 测试」

## 前端基础实现

### 步骤 F.1：创建 Vue.js 项目
```bash
# 安装 Vue CLI（如果没有）
npm install -g @vue/cli

# 创建项目
vue create rag-frontend
cd rag-frontend

# 安装依赖
npm install axios vue-router vuex element-ui
```

### 步骤 F.2：创建基础组件
创建 DocumentUpload.vue、QueryInterface.vue 等组件，实现基本的文件上传和问答界面。

👉「请完成测试文档中 F.1-F.2 测试」

---

**注意**: 所有开发完成后，请按照测试文档进行完整验证，确保每个组件都能正常工作。
