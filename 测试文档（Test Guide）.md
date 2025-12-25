# Java企业级智能知识库平台 - 测试文档

## 测试原则
- **执行位置明确**: 每个测试明确标注在 Windows 或 WSL Ubuntu 执行
- **命令可直接复制**: 所有测试命令可以直接复制执行
- **结果可验证**: 提供明确的预期结果和通过标准
- **逐步验证**: 按照开发阶段顺序进行测试

---

## 阶段 0：WSL Ubuntu 中间件部署测试

### 0.1 Docker 安装测试
**执行位置**: WSL Ubuntu  
**测试命令**:
```bash
# 检查 Docker 版本
docker --version

# 检查 Docker 服务状态
sudo systemctl status docker

# 检查当前用户是否在 docker 组中
groups $USER
```
**预期结果**:
```
Docker version 24.x.x, build xxxxxxx
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since ...
docker
```
**通过标准**: Docker 服务正在运行，用户在 docker 组中

### 0.2 Docker 网络创建测试
**执行位置**: WSL Ubuntu  
**测试命令**:
```bash
# 查看创建的网络（docker-compose管理的）
docker network ls | grep rag

# 检查网络详情
docker network inspect rag_rag-network
```
**预期结果**:
```
rag_rag-network    bridge    local
```
**通过标准**: rag_rag-network 网络存在

### 0.3 MySQL 服务测试
**执行位置**: WSL Ubuntu (项目根目录)
**测试命令**:
```bash
# 检查服务状态
docker compose ps mysql

# 连接测试
docker compose exec mysql mysql -u rag_user -prag_pass -e "SELECT 1;" rag_db
```
**预期结果**:
```
NAME       IMAGE      SERVICE   STATUS    PORTS
rag-mysql  mysql:8.0  mysql     running   0.0.0.0:3307->3306/tcp
+---+
| 1 |
+---+
| 1 |
+---+
```
**通过标准**: MySQL 容器运行正常，数据库连接成功

### 0.4 Redis 服务测试
**执行位置**: WSL Ubuntu (项目根目录)
**测试命令**:
```bash
# 检查服务状态
docker compose ps redis

# 连接测试
docker compose exec redis redis-cli ping
```
**预期结果**:
```
NAME       IMAGE          SERVICE   STATUS    PORTS
rag-redis  redis:7-alpine redis    running   0.0.0.0:6380->6379/tcp
PONG
```
**通过标准**: Redis 服务运行正常，PING 响应正常

### 0.5 Milvus 服务测试
**执行位置**: WSL Ubuntu (项目根目录)
**测试命令**:
```bash
# 检查服务状态
docker compose ps milvus

# 健康检查
curl http://localhost:9091/healthz
```
**预期结果**:
```
NAME        IMAGE                      SERVICE   STATUS    PORTS
rag-milvus  milvusdb/milvus:v2.6.7    milvus   running   0.0.0.0:9091->9091/tcp, 0.0.0.0:19530->19530/tcp
OK
```
**通过标准**: Milvus 服务运行正常，健康检查返回 "OK"

### 0.6 Docker Compose 服务启动测试
**执行位置**: WSL Ubuntu (项目根目录)
**测试命令**:
```bash
# 检查所有服务状态
docker compose ps

# 验证各个服务端口是否可访问
# MySQL (使用mysqladmin ping)
docker exec rag-mysql mysqladmin ping -h localhost -u root -proot123 || echo "MySQL port check failed"

# Redis (使用redis-cli ping)
docker exec rag-redis redis-cli ping || echo "Redis port check failed"

# RabbitMQ 管理界面
curl -u admin:admin123 http://localhost:15673/api/overview

# Milvus
curl -f http://localhost:9091/healthz || echo "Milvus port check failed"
```
**预期结果**:
```
NAME           IMAGE                    SERVICE    STATUS    PORTS
rag-milvus     milvusdb/milvus:v2.6.7   milvus     running   0.0.0.0:9091->9091/tcp, 0.0.0.0:19530->19530/tcp
rag-mysql      mysql:8.0                mysql      running   0.0.0.0:3307->3306/tcp
rag-rabbitmq   rabbitmq:3-management    rabbitmq   running   0.0.0.0:5673->5672/tcp, 0.0.0.0:15673->15672/tcp
rag-redis      redis:7-alpine           redis      running   0.0.0.0:6380->6379/tcp

# 服务连接检查结果
mysqld is alive
PONG

# RabbitMQ API 响应
{"management_version":"3.x.x",...}
```
**通过标准**: 所有服务运行正常，端口可访问，服务连接测试成功

---

## 阶段 1：项目架构与目录设计测试

### 1.1 Spring Boot 项目创建测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查项目结构
ls -la rag-platform/

# 检查 Maven 配置
cat rag-platform/pom.xml | grep -A 5 "<dependencies>"

# 编译测试
cd rag-platform && mvn clean compile
```
**预期结果**:
```
drwxr-xr-x  12 user  staff   384 Dec 24 12:00 .
drwxr-xr-x   3 user  staff    96 Dec 24 12:00 ..
-rw-r--r--   1 user  staff  1161 Dec 24 12:00 pom.xml
drwxr-xr-x   4 user  staff   128 Dec 24 12:00 src
...
[INFO] BUILD SUCCESS
```
**通过标准**: 项目结构完整，Maven 编译成功

### 1.2 配置文件测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查配置文件
cat rag-platform/src/main/resources/application.yml

# 验证配置文件语法
cd rag-platform && mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=test" &
sleep 5
curl http://localhost:8080/actuator/health
pkill -f "spring-boot:run"
```
**预期结果**:
```
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/rag_db?...
...
{"status":"UP"}
```
**通过标准**: 配置文件存在且格式正确，Spring Boot 可启动并响应健康检查

### 1.3 基础包结构测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查包结构
find rag-platform/src/main/java -type d -name "*" | sort

# 检查关键类文件是否存在
ls -la rag-platform/src/main/java/com/rag/platform/

# 编译检查
cd rag-platform && mvn compile
```
**预期结果**:
```
rag-platform/src/main/java/com/rag/platform
rag-platform/src/main/java/com/rag/platform/config
rag-platform/src/main/java/com/rag/platform/controller
...
drwxr-xr-x  8 user  staff   256 Dec 24 12:00 config
drwxr-xr-x  8 user  staff   256 Dec 24 12:00 controller
drwxr-xr-x  8 user  staff   256 Dec 24 12:00 service
...
[INFO] BUILD SUCCESS
```
**通过标准**: 所有必需的包和基础类文件存在，编译通过

---

## 阶段 2：数据库设计测试

### 2.1 User 实体测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查实体类代码
cat rag-platform/src/main/java/com/rag/platform/entity/User.java

# 启动应用测试 JPA
cd rag-platform && mvn spring-boot:run &
sleep 10

# 测试数据库连接
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass","email":"test@example.com"}'

pkill -f "spring-boot:run"
```
**预期结果**:
```
@Entity
@Table(name = "users")
public class User {
...
{"id":1,"username":"testuser","email":"test@example.com"...}
```
**通过标准**: User 实体代码正确，数据库表创建成功，可插入数据

### 2.2 Document 实体测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查实体类代码
cat rag-platform/src/main/java/com/rag/platform/entity/Document.java

# 验证表结构
cd rag-platform && mvn spring-boot:run &
sleep 10

# 检查数据库表
docker exec -it rag-mysql mysql -u rag_user -prag_pass rag_db -e "DESCRIBE documents;"

pkill -f "spring-boot:run"
```
**预期结果**:
```
+-------------+---------------+------+-----+---------+----------------+
| Field       | Type          | Null | Key | Default | Extra          |
+-------------+---------------+------+-----+---------+----------------+
| id          | bigint(20)    | NO   | PRI | NULL    | auto_increment |
| title       | varchar(255)  | NO   |     | NULL    |                |
| content     | text          | YES  |     | NULL    |                |
...
```
**通过标准**: Document 实体正确，数据库表结构符合设计

### 2.3 QueryLog 实体测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查实体类代码
cat rag-platform/src/main/java/com/rag/platform/entity/QueryLog.java

# 验证表创建
cd rag-platform && mvn spring-boot:run &
sleep 10

# 检查表存在
docker exec -it rag-mysql mysql -u rag_user -prag_pass rag_db -e "SHOW TABLES LIKE 'query_logs';"

pkill -f "spring-boot:run"
```
**预期结果**:
```
+------------------+
| Tables_in_rag_db |
+------------------+
| query_logs       |
+------------------+
```
**通过标准**: QueryLog 实体正确，query_logs 表成功创建

---

## 阶段 3：API 接口设计测试

### 3.1 UserController 测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动应用
cd rag-platform && mvn spring-boot:run &
sleep 10

# 测试用户注册
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"apiuser","password":"apipass","email":"api@example.com"}'

# 测试用户登录
curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"apiuser","password":"apipass"}'

pkill -f "spring-boot:run"
```
**预期结果**:
```
{"id":2,"username":"apiuser","email":"api@example.com"...}
"eyJhbGciOiJIUzI1NiJ9..."
```
**通过标准**: 注册接口返回用户对象，登录接口返回 JWT token

### 3.2 DocumentController 测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动应用
cd rag-platform && mvn spring-boot:run &
sleep 10

# 先注册用户并获取 token
TOKEN=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"apiuser","password":"apipass"}' | tr -d '"')

# 测试文档上传（使用测试文件）
echo "This is a test document content." > test.txt
curl -X POST http://localhost:8080/api/documents/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.txt" \
  -F "title=Test Document"

# 清理测试文件
rm test.txt
pkill -f "spring-boot:run"
```
**预期结果**:
```
{"id":1,"title":"Test Document","content":"This is a test document content."...}
```
**通过标准**: 文档上传接口正常工作，返回文档对象

### 3.3 QueryController 测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动应用
cd rag-platform && mvn spring-boot:run &
sleep 10

# 获取用户 token
TOKEN=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"apiuser","password":"apipass"}' | tr -d '"')

# 测试查询接口
curl -X POST http://localhost:8080/api/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is this document about?"}'

pkill -f "spring-boot:run"
```
**预期结果**:
```
{"answer":"Based on the uploaded document, this appears to be a test document...", "sources":1}
```
**通过标准**: 查询接口返回包含回答和来源数量的响应

---

## 阶段 4：核心业务功能测试

### 4.1 UserService 测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动应用
cd rag-platform && mvn spring-boot:run &
sleep 10

# 测试用户注册和登录流程
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"servicetest","password":"servicepass"}'

LOGIN_RESPONSE=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"servicetest","password":"servicepass"}')

# 验证 token 有效性
TOKEN=$(echo $LOGIN_RESPONSE | tr -d '"')
curl -X GET http://localhost:8080/api/users/profile \
  -H "Authorization: Bearer $TOKEN"

pkill -f "spring-boot:run"
```
**预期结果**:
```
{"id":3,"username":"servicetest"...}
"eyJhbGciOiJIUzI1NiJ9..."
{"username":"servicetest","email":null}
```
**通过标准**: 用户服务正常工作，注册、登录、token 验证全部通过

### 4.2 DocumentService 测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动应用
cd rag-platform && mvn spring-boot:run &
sleep 10

# 获取用户 token
TOKEN=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"servicetest","password":"servicepass"}' | tr -d '"')

# 创建测试文档
echo "This is a comprehensive test document for RAG system validation." > rag_test.txt

# 上传文档
UPLOAD_RESPONSE=$(curl -X POST http://localhost:8080/api/documents/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@rag_test.txt" \
  -F "title=RAG Test Document")

# 获取用户文档列表
curl -X GET http://localhost:8080/api/documents \
  -H "Authorization: Bearer $TOKEN"

# 清理
rm rag_test.txt
pkill -f "spring-boot:run"
```
**预期结果**:
```
{"id":2,"title":"RAG Test Document"...}
[{"id":2,"title":"RAG Test Document","fileSize":64...}]
```
**通过标准**: 文档服务正常，上传和查询功能都工作正常

---

## 阶段 5：RAG 最小可用能力测试

### 5.1 EmbeddingService 测试（Ollama 连接）
**执行位置**: Windows  
**测试命令**:
```bash
# 首先确保 Ollama 在 Windows 上运行
# 检查 Ollama 服务
curl http://localhost:11434/api/tags

# 启动 Spring Boot 应用
cd rag-platform && mvn spring-boot:run &
sleep 15

# 测试嵌入生成（通过内部 API 调用验证）
curl -X POST http://localhost:8080/api/test/embedding \
  -H "Content-Type: application/json" \
  -d '{"text":"test embedding generation"}'

pkill -f "spring-boot:run"
```
**预期结果**:
```
{"models":[{"name":"qwen2.5:14b",...}]}
{"embedding":[0.123, 0.456, ...], "length":768}
```
**通过标准**: Ollama 服务运行正常，嵌入生成功能工作

### 5.2 VectorService 测试（Milvus 连接）
**执行位置**: Windows  
**测试命令**:
```bash
# 启动 Spring Boot 应用
cd rag-platform && mvn spring-boot:run &
sleep 15

# 获取用户 token 并上传测试文档
TOKEN=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"servicetest","password":"servicepass"}' | tr -d '"')

echo "Machine learning is a subset of artificial intelligence." > ml_doc.txt

curl -X POST http://localhost:8080/api/documents/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@ml_doc.txt" \
  -F "title=Machine Learning Document"

# 等待向量化完成
sleep 10

# 检查 Milvus 中的向量
curl http://localhost:9091/api/v1/collection/describe -X GET \
  -d '{"collection_name": "documents"}'

rm ml_doc.txt
pkill -f "spring-boot:run"
```
**预期结果**:
```
{"status":{},"schema":{"fields":[{"name":"id"...}]}}
```
**通过标准**: 向量服务正常，向量成功存储到 Milvus

### 5.3 RagService 完整闭环测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动 Spring Boot 应用
cd rag-platform && mvn spring-boot:run &
sleep 20

# 获取用户 token
TOKEN=$(curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"servicetest","password":"servicepass"}' | tr -d '"')

# 执行 RAG 查询
curl -X POST http://localhost:8080/api/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is machine learning?"}'

# 检查查询日志是否记录
docker exec -it rag-mysql mysql -u rag_user -prag_pass rag_db \
  -e "SELECT question, LEFT(answer, 100) as answer_preview FROM query_logs ORDER BY id DESC LIMIT 1;"

pkill -f "spring-boot:run"
```
**预期结果**:
```
{
  "answer": "Machine learning is a subset of artificial intelligence...",
  "sources": 1,
  "responseTime": 2500
}
+-----------------------------+--------------------------------+
| question                    | answer_preview                 |
+-----------------------------+--------------------------------+
| What is machine learning?   | Machine learning is a subset...|
+-----------------------------+--------------------------------+
```
**通过标准**: RAG 流程完整，查询成功生成回答，日志正确记录

---

## 前端基础实现测试

### F.1 Vue.js 项目创建测试
**执行位置**: Windows  
**测试命令**:
```bash
# 检查项目结构
ls -la rag-frontend/

# 检查依赖
cat rag-frontend/package.json

# 安装依赖并启动
cd rag-frontend && npm install && npm run serve &
sleep 10

# 测试前端访问
curl http://localhost:8080

pkill -f "npm run serve"
```
**预期结果**:
```
drwxr-xr-x  10 user  staff   320 Dec 24 12:00 .
-rw-r--r--   1 user  staff  1234 Dec 24 12:00 package.json
...
<!DOCTYPE html>
<html>
<head>
  <title>RAG Knowledge Platform</title>
...
```
**通过标准**: Vue.js 项目结构正确，前端服务可启动并访问

### F.2 基础组件测试
**执行位置**: Windows  
**测试命令**:
```bash
# 启动前端服务
cd rag-frontend && npm run serve &
sleep 10

# 启动后端服务
cd ../rag-platform && mvn spring-boot:run &
sleep 10

# 测试前后端集成
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"frontendtest","password":"frontendpass"}'

# 通过前端界面测试（模拟）
echo "前端组件集成测试完成"

pkill -f "spring-boot:run"
pkill -f "npm run serve"
```
**预期结果**:
```
{"id":4,"username":"frontendtest"...}
前端组件集成测试完成
```
**通过标准**: 前后端集成正常，前端组件可正常调用后端 API

---

## 完整系统集成测试

### I.1 端到端 RAG 流程测试
**执行位置**: Windows  
**测试命令**:
```bash
# 1. 启动所有服务
cd rag-platform && mvn spring-boot:run &
sleep 15

cd ../rag-frontend && npm run serve &
sleep 10

# 2. 创建用户
curl -X POST http://localhost:8081/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"e2etest","password":"e2epass"}'

# 3. 登录获取 token
TOKEN=$(curl -X POST http://localhost:8081/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"e2etest","password":"e2epass"}' | tr -d '"')

# 4. 上传文档
echo "Spring Boot is a framework for building Java applications." > spring_doc.txt
curl -X POST http://localhost:8081/api/documents/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@spring_doc.txt" \
  -F "title=Spring Boot Guide"

# 5. 等待向量化
sleep 15

# 6. 执行问答查询
curl -X POST http://localhost:8081/api/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is Spring Boot?"}'

# 7. 清理
rm spring_doc.txt
pkill -f "spring-boot:run"
pkill -f "npm run serve"
```
**预期结果**:
```
{"answer":"Spring Boot is a framework for building Java applications...", "sources":1}
```
**通过标准**: 完整 RAG 流程从文档上传到智能问答全部正常工作

---

**测试完成标准**: 所有测试项都通过，系统可实现文档上传、智能问答的完整闭环。
