# AWS 部署完整教程：从Docker构建到生产环境

> **语言版本 | Language Versions:** [English](./aws-docker-deployment-guide-en.md) | **中文** (当前)

这是一个专注于实际操作的AWS部署教程，详细解释每个shell命令和配置步骤。

## 目录
1. [Docker本地构建与测试详解](#docker本地构建与测试详解)
2. [AWS CLI配置与认证](#aws-cli配置与认证)
3. [ECR容器镜像推送详解](#ecr容器镜像推送详解)
4. [App Runner服务部署](#app-runner服务部署)
5. [监控调试与故障排除](#监控调试与故障排除)
6. [更新部署流程](#更新部署流程)
7. [AWS基础概念补充](#aws基础概念补充)

---

## Docker本地构建与测试详解

### 环境变量加载机制

在开始构建之前，我们需要理解环境变量在不同操作系统中的加载方式：

#### Mac/Linux系统
```bash
export $(cat .env | grep -v '^#' | xargs)
```

**命令解析：**
- `cat .env`: 读取.env文件内容
- `grep -v '^#'`: 过滤掉以#开头的注释行（-v表示反向匹配）
- `xargs`: 将输入转换为命令行参数
- `export`: 将变量设置为环境变量，供子进程使用

**实际执行过程：**
1. 读取.env文件：`CLERK_SECRET_KEY=sk_test_xxx`
2. 过滤注释后得到：`CLERK_SECRET_KEY=sk_test_xxx`
3. export命令执行：`export CLERK_SECRET_KEY=sk_test_xxx`

#### Windows PowerShell系统
```powershell
Get-Content .env | ForEach-Object {
    if ($_ -match '^(.+?)=(.+)$') {
        [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2])
    }
}
```

**命令解析：**
- `Get-Content .env`: PowerShell读取文件的命令
- `ForEach-Object`: 对每一行执行操作
- `$_ -match '^(.+?)=(.+)$'`: 正则表达式匹配键值对格式
- `$matches[1]`: 匹配到的变量名
- `$matches[2]`: 匹配到的变量值
- `[System.Environment]::SetEnvironmentVariable()`: .NET方法设置环境变量

### Docker构建命令详解

#### Mac/Linux构建
```bash
docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**参数详解：**
- `docker build`: Docker镜像构建命令
- `\`: 行续行符，用于多行命令的可读性
- `--build-arg`: 向Dockerfile传递构建时参数
  - 这些参数只在构建过程中可用，不会保存在最终镜像中
  - 用于传递在构建时需要的敏感信息
- `"$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY"`:
  - 使用双引号确保变量值中的空格等特殊字符被正确处理
  - $符号表示引用之前设置的环境变量
- `-t consultation-app`:
  - tag标记，给构建的镜像命名
  - 如果不指定版本，默认为`latest`
- `.`: 构建上下文，指向当前目录

**构建过程发生了什么：**
1. Docker读取当前目录的Dockerfile
2. 将`NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`作为ARG传入
3. 执行多阶段构建：
   - Stage 1: Node.js环境构建前端静态文件
   - Stage 2: Python环境配置后端服务
4. 最终生成名为`consultation-app:latest`的镜像

### Docker运行命令详解

#### Mac/Linux运行
```bash
docker run -p 8000:8000 \
  -e CLERK_SECRET_KEY="$CLERK_SECRET_KEY" \
  -e CLERK_JWKS_URL="$CLERK_JWKS_URL" \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  consultation-app
```

**参数详解：**
- `docker run`: 运行容器命令
- `-p 8000:8000`: 端口映射
  - 格式：`主机端口:容器端口`
  - 将本机8000端口映射到容器内8000端口
  - 这样就可以通过`localhost:8000`访问容器内的应用
- `-e`: 设置环境变量
  - 这些变量会在容器运行时可用
  - 与构建时的`--build-arg`不同，这些是运行时环境变量
- `consultation-app`: 要运行的镜像名称

**为什么需要分别设置构建时和运行时变量？**
- **构建时变量（--build-arg）**: 用于在构建过程中配置应用（如前端公钥）
- **运行时变量（-e）**: 用于在容器运行时配置应用（如后端密钥）

### 常见构建问题和解决方案

#### 问题1：环境变量未加载
```bash
# 错误现象
echo $CLERK_SECRET_KEY
# 输出为空

# 解决方案
# 1. 检查.env文件是否存在
ls -la .env

# 2. 检查文件内容格式
cat .env

# 3. 重新加载环境变量
export $(cat .env | grep -v '^#' | xargs)
```

#### 问题2：Docker构建失败
```bash
# 错误现象
Error: Cannot find module 'next'

# 解决方案
# 1. 检查package.json是否存在
ls -la package.json

# 2. 清理Docker缓存
docker system prune -f

# 3. 重新构建
docker build --no-cache -t consultation-app .
```

#### 问题3：端口占用
```bash
# 错误现象
Error: Port 8000 is already in use

# 解决方案
# 1. 查找占用端口的进程
lsof -i :8000

# 2. 杀死进程
kill -9 [进程ID]

# 3. 或使用不同端口
docker run -p 8001:8000 ...
```

---

## AWS CLI配置与认证

### AWS CLI安装验证

```bash
# 检查AWS CLI是否已安装
aws --version

# 期望输出类似：
# aws-cli/2.15.30 Python/3.11.6 Darwin/23.1.0 exe/x86_64 prompt/off
```

### AWS配置命令详解

```bash
aws configure
```

**配置过程详解：**

1. **AWS Access Key ID**:
   ```
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   ```
   - 这是AWS用户的唯一标识符
   - 格式通常以`AKIA`开头，后跟16位随机字符
   - 相当于用户名，不是秘密信息

2. **AWS Secret Access Key**:
   ```
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY
   ```
   - 这是与Access Key配对的密码
   - 长度通常为40个字符
   - **极其重要**：这是敏感信息，泄露会导致账户被盗用

3. **Default region name**:
   ```
   Default region name [None]: us-east-1
   ```
   - AWS服务的地理位置
   - 影响网络延迟和数据存储位置
   - 常用区域：
     - `us-east-1`: 美国东部（弗吉尼亚北部）- 最便宜，服务最全
     - `us-west-2`: 美国西部（俄勒冈）- 西海岸用户最佳
     - `eu-west-1`: 欧洲（爱尔兰）- 欧洲用户最佳
     - `ap-southeast-1`: 亚太（新加坡）- 亚洲用户最佳

4. **Default output format**:
   ```
   Default output format [None]: json
   ```
   - 其他选项：`table`, `text`, `yaml`
   - `json`最适合编程处理

### 认证机制原理

当你运行`aws configure`后，AWS CLI会在以下位置创建配置文件：

```bash
# 查看配置文件位置
ls -la ~/.aws/

# 输出：
# credentials  - 存储访问密钥
# config      - 存储区域和输出格式
```

**credentials文件内容：**
```
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY
```

**config文件内容：**
```
[default]
region = us-east-1
output = json
```

### 验证AWS配置

```bash
# 测试AWS CLI配置
aws sts get-caller-identity

# 期望输出：
{
    "UserId": "AIDACKCEVSQ6C2EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/aiengineer"
}
```

**输出解析：**
- `UserId`: IAM用户的唯一ID
- `Account`: 12位AWS账户ID
- `Arn`: Amazon Resource Name，用户的完整路径

---

## ECR容器镜像推送详解

### ECR认证机制

```bash
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com
```

**命令分解分析：**

1. **`aws ecr get-login-password --region $DEFAULT_AWS_REGION`**
   - 向AWS ECR请求临时登录密码
   - 密码有效期12小时
   - 这不是你的AWS密码，而是专门用于Docker的临时token

2. **管道操作 `|`**
   - 将上一个命令的输出作为下一个命令的输入
   - 实现命令链式调用

3. **`docker login --username AWS --password-stdin`**
   - `--username AWS`: ECR的固定用户名就是"AWS"
   - `--password-stdin`: 从标准输入读取密码（即上一个命令的输出）
   - 避免在命令行中明文显示密码

4. **ECR Registry URL格式**
   ```
   $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com
   ```
   - `$AWS_ACCOUNT_ID`: 你的12位AWS账户ID
   - `dkr.ecr`: ECR服务的标识
   - `$DEFAULT_AWS_REGION`: AWS区域
   - `amazonaws.com`: AWS域名

### 平台特定构建

```bash
docker build \
  --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**为什么需要`--platform linux/amd64`？**

1. **Apple Silicon (M1/M2/M3) Mac用户**：
   - 本地CPU架构：ARM64
   - AWS运行环境：x86_64 (AMD64)
   - 不指定平台会导致架构不匹配错误

2. **Intel Mac和Windows用户**：
   - 本地CPU架构：x86_64
   - AWS运行环境：x86_64
   - 可以不指定，但指定更安全

3. **架构不匹配的错误现象**：
   ```
   standard_init_linux.go:228: exec user process caused: exec format error
   ```

### 镜像标记（Tagging）

```bash
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**标记原理：**
- Docker镜像可以有多个标记（tags）
- 本地标记：`consultation-app:latest`
- 远程标记：`123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:latest`
- 同一个镜像，不同标记指向同一个镜像ID

**标记命名规范：**
```
[registry-url]/[repository-name]:[tag]

示例：
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:v1.0.0
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:latest
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:dev
```

### 镜像推送过程

```bash
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**推送过程详解：**

1. **层级上传**：
   ```
   The push refers to repository [123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app]
   a8c6b3421... Pushing [==================================================>]  5.632kB/5.632kB
   b9d7e4567... Pushing [===================>                               ]  45.23MB/108.5MB
   ```

2. **层级复用**：
   - Docker镜像由多个层组成
   - 相同的层在多个镜像间共享
   - 只有改变的层需要重新上传

3. **推送时间估算**：
   - 首次推送：3-10分钟（取决于网络速度）
   - 后续推送：30秒-2分钟（只推送变更层）

### ECR推送故障排除

#### 问题1：认证失败
```bash
# 错误信息
Error response from daemon: login attempt to https://123456789012.dkr.ecr.us-east-1.amazonaws.com/v2/ failed with status: 401 Unauthorized

# 解决方案
# 1. 重新认证
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com

# 2. 检查IAM权限
aws iam list-attached-user-policies --user-name aiengineer
```

#### 问题2：仓库不存在
```bash
# 错误信息
Error response from daemon: repository consultation-app not found

# 解决方案
# 在ECR控制台创建名为'consultation-app'的仓库
aws ecr create-repository --repository-name consultation-app --region $DEFAULT_AWS_REGION
```

#### 问题3：网络超时
```bash
# 错误信息
Error response from daemon: net/http: TLS handshake timeout

# 解决方案
# 1. 检查网络连接
ping amazonaws.com

# 2. 重试推送
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# 3. 如果持续失败，检查防火墙设置
```

---

## App Runner服务部署

### App Runner配置详解

#### 计算资源配置

**vCPU和内存选择：**
```
vCPU: 0.25 vCPU
Memory: 0.5 GB
```

**配置含义：**
- `0.25 vCPU`: 1/4个虚拟CPU核心
  - 适合低流量应用
  - 成本最低的配置
  - 可处理轻度并发请求

- `0.5 GB Memory`: 512MB内存
  - Python FastAPI应用通常需要200-300MB
  - Next.js静态文件很小
  - 剩余内存用于系统缓存

**其他可选配置：**
```
1 vCPU, 2 GB   - 中等流量 (~100并发)
2 vCPU, 4 GB   - 高流量 (~500并发)
4 vCPU, 12 GB  - 企业级 (~2000并发)
```

#### 环境变量配置机制

在App Runner中设置的环境变量：
```
CLERK_SECRET_KEY=sk_test_...
CLERK_JWKS_URL=https://...
OPENAI_API_KEY=sk-...
```

**为什么不需要设置`NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`？**
- 这个变量在Docker构建时已经编译到静态文件中
- 构建时参数 vs 运行时环境变量的区别：
  - **构建时**：前端代码编译，公钥嵌入JavaScript
  - **运行时**：后端服务启动，密钥用于API验证

#### Auto Scaling配置

```
Minimum size: 1
Maximum size: 1
```

**为什么选择这个配置？**

1. **Minimum size: 1**
   - App Runner要求至少1个实例
   - 保证服务持续可用
   - 避免冷启动延迟

2. **Maximum size: 1**
   - 控制成本
   - 对于学习项目，1个实例足够
   - 生产环境通常设置为2-10

**生产环境推荐配置：**
```
Minimum size: 2  (高可用)
Maximum size: 10 (处理流量峰值)
```

#### Health Check详细配置

```
Protocol: HTTP
Path: /health
Interval: 20 seconds
Timeout: 5 seconds
Healthy threshold: 2
Unhealthy threshold: 5
```

**参数解析：**

1. **Protocol: HTTP**
   - App Runner向你的应用发送HTTP请求
   - 路径为`/health`
   - 期望返回HTTP 200状态码

2. **Interval: 20 seconds**
   - 每20秒检查一次
   - 范围：5-300秒
   - 选择20秒平衡了及时发现问题和避免过度检查

3. **Timeout: 5 seconds**
   - 等待响应的最大时间
   - 超过5秒视为检查失败
   - 应该小于Interval时间

4. **Healthy threshold: 2**
   - 连续2次成功才认为服务健康
   - 避免偶发成功造成误判

5. **Unhealthy threshold: 5**
   - 连续5次失败才认为服务不健康
   - 避免偶发失败造成误判

**Health Check端点代码：**
```python
@app.get("/health")
def health_check():
    """Health check endpoint for AWS App Runner"""
    return {"status": "healthy"}
```

### 部署过程监控

#### 部署状态变化
```
Creating → Building → Deploying → Running
```

**各阶段含义：**

1. **Creating (1-2分钟)**
   - App Runner创建服务实例
   - 分配计算资源
   - 配置网络

2. **Building (2-3分钟)**
   - 从ECR拉取Docker镜像
   - 启动容器
   - 应用初始化

3. **Deploying (1-2分钟)**
   - 健康检查
   - 流量路由配置
   - SSL证书配置

4. **Running**
   - 服务正常运行
   - 接受外部流量

#### 部署日志解读

**正常部署日志：**
```
[INFO] Pulling image from ECR
[INFO] Starting container on port 8000
[INFO] Health check passed: GET /health returned 200
[INFO] Service is ready to receive traffic
```

**异常部署日志：**
```
[ERROR] Health check failed: GET /health returned 500
[ERROR] Container exited with code 1
[WARN] Service deployment failed, rolling back
```

---

## 监控调试与故障排除

### CloudWatch日志导航

#### 访问日志的方式

1. **通过App Runner控制台**：
   - 服务页面 → Logs标签
   - 分为Application logs和Service logs

2. **直接访问CloudWatch**：
   - 搜索"CloudWatch" → Logs → Log groups
   - 找到`/aws/apprunner/consultation-app-service/...`

#### 日志类型解析

**Application Logs (应用日志)**：
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "message": "Starting FastAPI server on port 8000"
}
```

**Service Logs (系统日志)**：
```json
{
  "timestamp": "2024-01-15T10:29:55.000Z",
  "level": "INFO",
  "message": "Container started successfully"
}
```

### 常见错误诊断

#### 错误1：健康检查失败

**错误现象：**
```
Health check failed: timeout after 5 seconds
```

**诊断步骤：**
1. 检查应用是否在8000端口启动
2. 确认`/health`端点是否存在
3. 查看应用启动日志

**解决方案：**
```python
# 确保health端点存在
@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now()}

# 检查应用启动
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### 错误2：环境变量未加载

**错误现象：**
```
KeyError: 'CLERK_SECRET_KEY'
```

**诊断步骤：**
1. 检查App Runner环境变量配置
2. 确认变量名拼写正确
3. 验证变量值格式

**解决方案：**
```python
import os

# 安全获取环境变量
clerk_secret = os.getenv("CLERK_SECRET_KEY")
if not clerk_secret:
    raise ValueError("CLERK_SECRET_KEY environment variable is required")
```

#### 错误3：Clerk认证失败

**错误现象：**
```
Clerk authentication failed: Invalid JWKS URL
```

**诊断步骤：**
1. 验证JWKS URL格式
2. 检查Clerk应用配置
3. 确认公钥是否匹配

**JWKS URL格式检查：**
```
正确格式：https://[clerk-app-id].clerk.accounts.dev/.well-known/jwks.json
错误格式：https://clerk.com/[clerk-app-id]/jwks
```

### 性能监控指标

#### CPU和内存监控

**通过CloudWatch查看：**
1. CloudWatch → Metrics → AWS/AppRunner
2. 选择服务名称
3. 查看关键指标：
   - `CPUUtilization`: CPU使用率
   - `MemoryUtilization`: 内存使用率
   - `RequestCount`: 请求数量
   - `ResponseTime`: 响应时间

**正常范围：**
```
CPU使用率: 10-70% (持续90%+需要扩容)
内存使用率: 40-80% (持续95%+需要增加内存)
响应时间: <2秒 (>5秒需要优化)
```

#### 应用性能优化

**Python应用优化：**
```python
# 1. 使用连接池
from openai import OpenAI
client = OpenAI()  # 复用连接

# 2. 缓存静态内容
from fastapi.staticfiles import StaticFiles
app.mount("/static", StaticFiles(directory="static"), name="static")

# 3. 启用GZIP压缩
from fastapi.middleware.gzip import GZipMiddleware
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

---

## 更新部署流程

### 完整更新流程

当你修改代码后，需要重新部署到AWS：

#### 步骤1：重新构建镜像

```bash
# 确保环境变量已加载
export $(cat .env | grep -v '^#' | xargs)

# 构建新镜像（重要：包含platform参数）
docker build \
  --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**为什么需要重新构建？**
- Docker镜像是不可变的
- 代码更改需要创建新的镜像层
- 确保所有依赖项都是最新的

#### 步骤2：重新推送到ECR

```bash
# 重新认证（如果超过12小时）
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com

# 重新标记镜像
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# 推送更新
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**推送优化：**
- Docker只推送变更的层
- 未变更的层会被跳过
- 通常比首次推送快很多

#### 步骤3：触发App Runner部署

**在App Runner控制台：**
1. 进入你的服务
2. 点击"Deploy"按钮
3. 等待部署完成（3-5分钟）

**部署过程监控：**
```
Status: Deploying
↓
Status: Running (新版本)
```

### 版本管理策略

#### 使用语义化版本标记

**而不是只使用`latest`：**
```bash
# 标记特定版本
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:v1.1.0

# 同时保持latest
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# 推送两个标记
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:v1.1.0
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**版本号规则：**
```
v1.0.0 - 初始版本
v1.0.1 - Bug修复
v1.1.0 - 新功能
v2.0.0 - 重大更改
```

#### 回滚策略

**如果新版本有问题：**

1. **快速回滚到ECR中的旧版本：**
   ```bash
   # 查看ECR中的所有版本
   aws ecr list-images --repository-name consultation-app

   # 重新标记旧版本为latest
   aws ecr batch-get-image --repository-name consultation-app --image-ids imageTag=v1.0.0 --query 'images[0].imageManifest' --output text | aws ecr put-image --repository-name consultation-app --image-tag latest --image-manifest file:///dev/stdin
   ```

2. **在App Runner中重新部署：**
   - 点击"Deploy"触发使用latest标记的部署

### 零停机部署考虑

**App Runner的滚动更新：**
- 新实例启动并通过健康检查后
- 旧实例才会被终止
- 理论上实现零停机

**但可能的中断：**
- 健康检查失败导致回滚
- 新代码启动时间过长
- 数据库迁移需要停机

**最佳实践：**
```python
# 1. 优雅关闭
import signal
import sys

def signal_handler(sig, frame):
    print('Gracefully shutting down...')
    # 完成正在处理的请求
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)

# 2. 健康检查包含依赖项检查
@app.get("/health")
def health_check():
    # 检查数据库连接
    # 检查外部API可用性
    return {"status": "healthy"}
```

---

## AWS基础概念补充

### IAM权限模型深入理解

#### IAM核心组件

**Users（用户）**：
- 代表人或应用程序
- 拥有永久的访问凭证
- 我们创建的`aiengineer`就是一个用户

**Groups（组）**：
- 用户的集合
- 便于批量管理权限
- 我们创建的`BroadAIEngineerAccess`就是一个组

**Policies（策略）**：
- JSON文档，定义权限
- 可以附加到用户、组或角色
- 采用"最小权限原则"

**Roles（角色）**：
- 临时权限的集合
- 可以被用户或服务"扮演"
- App Runner使用的ECR访问角色就是一个例子

#### 权限策略示例

**AWSAppRunnerFullAccess策略（简化版）**：
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "apprunner:*"
            ],
            "Resource": "*"
        }
    ]
}
```

**权限控制粒度：**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "apprunner:CreateService",
                "apprunner:UpdateService"
            ],
            "Resource": "arn:aws:apprunner:us-east-1:123456789012:service/consultation-app-service/*"
        }
    ]
}
```

### AWS服务对比

#### Container部署选项对比

**AWS App Runner**：
```
优点：
- 全托管，零配置
- 自动HTTPS和负载均衡
- 按需扩缩容
- 简单定价

缺点：
- 较少定制选项
- 有限的网络控制
- 成本可能较高

适用场景：快速原型、小到中型应用
```

**Amazon ECS + Fargate**：
```
优点：
- 更多控制选项
- 更好的成本控制
- 支持复杂网络配置
- 与其他AWS服务深度集成

缺点：
- 配置复杂
- 需要更多AWS知识
- 学习曲线陡峭

适用场景：企业级应用、复杂架构
```

**AWS Lambda**：
```
优点：
- 真正的无服务器
- 极致的成本效率（按请求计费）
- 自动扩缩容到零
- 无需管理基础设施

缺点：
- 冷启动延迟
- 执行时间限制（15分钟）
- 复杂的本地调试

适用场景：事件驱动、短时任务、API网关
```

#### 存储服务对比

**Amazon ECR vs Docker Hub**：

**ECR优势：**
```
- 与AWS服务深度集成
- 更好的安全性和权限控制
- 在AWS网络内传输更快
- 支持漏洞扫描
```

**Docker Hub优势：**
```
- 免费公开仓库
- 更大的社区
- 更简单的使用方式
- 更好的文档和教程
```

### 区域（Region）和可用区（AZ）

#### 区域选择策略

**基于用户位置：**
```
中国用户 → ap-southeast-1 (新加坡)
美国用户 → us-east-1 (弗吉尼亚) 或 us-west-2 (俄勒冈)
欧洲用户 → eu-west-1 (爱尔兰)
```

**基于成本：**
```
最便宜：us-east-1 (弗吉尼亚北部)
较便宜：us-west-2 (俄勒冈)
稍贵：其他美国区域
最贵：亚太和某些欧洲区域
```

**基于服务可用性：**
```
新服务首发：us-east-1
服务最全：us-east-1, us-west-2, eu-west-1
部分服务缺失：其他区域
```

### 成本优化深度指南

#### 计费模型理解

**App Runner计费：**
```
基础费用：$0.0066/小时（0.25 vCPU，0.5GB内存）
月度成本：$0.0066 × 24 × 30 = $4.75

实际场景：
- 持续运行：$4.75/月
- 工作时间运行（8小时/天）：$1.58/月
- 暂停服务：$0/月（但需要手动启停）
```

**ECR存储费用：**
```
存储：$0.10/GB/月
示例：500MB镜像 = $0.05/月

优化策略：
- 删除旧版本镜像
- 使用多阶段构建减小镜像大小
- 定期清理未使用的镜像
```

#### 成本监控自动化

**设置更详细的预算告警：**

1. **逐日监控：**
   ```
   每日预算：$0.33 ($10/30天)
   告警阈值：80% ($0.26)
   ```

2. **服务级别监控：**
   ```
   App Runner：$5/月预算
   ECR：$1/月预算
   其他服务：$4/月预算
   ```

3. **自动化响应：**
   ```python
   # 使用AWS Lambda监控成本
   import boto3

   def lambda_handler(event, context):
       if event['cost'] > 8.0:  # $8阈值
           # 发送紧急通知
           # 可选：自动暂停服务
           pass
   ```

### 安全最佳实践

#### 访问密钥管理

**密钥轮换策略：**
```bash
# 每90天轮换一次访问密钥
# 1. 创建新密钥
aws iam create-access-key --user-name aiengineer

# 2. 更新本地配置
aws configure

# 3. 测试新密钥
aws sts get-caller-identity

# 4. 删除旧密钥
aws iam delete-access-key --access-key-id AKIA... --user-name aiengineer
```

**密钥存储安全：**
```bash
# 设置适当的文件权限
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config

# 检查文件权限
ls -la ~/.aws/
# -rw------- (只有所有者可读写)
```

#### 网络安全

**HTTPS强制：**
- App Runner自动提供HTTPS
- 自动重定向HTTP到HTTPS
- 免费SSL/TLS证书

**环境变量安全：**
```python
# 在应用中避免记录敏感信息
import logging
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ❌ 错误：可能泄露密钥
logger.info(f"Using API key: {os.getenv('OPENAI_API_KEY')}")

# ✅ 正确：只记录非敏感信息
api_key = os.getenv('OPENAI_API_KEY')
if api_key:
    logger.info("OpenAI API key loaded successfully")
else:
    logger.error("OpenAI API key not found")
```

---

## 结语

这个教程涵盖了从Docker构建到AWS生产部署的完整流程。重点强调了：

1. **实际操作**：每个命令都有详细解释
2. **故障排除**：常见问题和解决方案
3. **最佳实践**：安全、成本控制、性能优化
4. **深度理解**：不仅知道怎么做，更知道为什么这么做

继续学习AWS的建议：
- 实践其他AWS服务（RDS、S3、CloudFront）
- 学习基础设施即代码（Terraform、CloudFormation）
- 了解CI/CD管道（GitHub Actions + AWS）
- 探索监控和日志分析（CloudWatch、X-Ray）

记住：AWS是一个庞大的平台，掌握基础概念比记住具体操作更重要。通过实际项目驱动学习，逐步深入，是最有效的学习方式。

---

**其他语言版本：** [English Version](./aws-docker-deployment-guide-en.md)