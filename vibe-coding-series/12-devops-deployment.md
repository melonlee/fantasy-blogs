# Vibe Coding 系列 12：DevOps 与部署

代码写完了，总要上线的。这大概是整个开发流程里最让人提心吊胆的环节——本地跑得好好的，一部署就出问题。环境差异、依赖版本、配置文件泄露，这些都是常见的坑。这篇来把部署这件事系统地过一遍：从本地打包到 Docker 容器化，再到 GitHub Actions 自动化流水线，最后聊几个生产环境的注意事项。

## Docker 容器化

Docker 的核心价值是把"在我机器上能跑"变成"在服务器上也能跑"。通过把应用和它的运行环境一起打包，消除了"环境差异"这个老大难问题。

![可爱的卡通程序员在 Docker 鲸鱼上部署代码](/tmp/fantasy-blogs/vibe-coding-series/images/12-devops-deployment.jpg)

### 前端 Dockerfile

先从前端开始。React 应用用 Vite 构建，最终产物是一堆静态文件，用 Nginx 来服务：

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# 先复制依赖清单，利用 Docker 缓存层
COPY package.json package-lock.json ./
RUN npm ci

# 再复制源代码
COPY . .

# 构建
ARG VITE_API_BASE=https://api.production.example.com
ENV VITE_API_BASE=$VITE_API_BASE
RUN npm run build

# 生产镜像
FROM nginx:alpine

# 从 builder 阶段复制构建产物
COPY --from=builder /app/dist /usr/share/nginx/html

# Nginx 配置
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

分层构建是关键——先装依赖，再复制代码，最后构建。这样在代码变动而依赖不变的情况下，Docker 可以复用缓存的依赖层，不需要每次都重新下载 npm 包。

### 后端 Dockerfile

Spring Boot 走的是另一个路数——编译成 JAR 之后直接用 JDK 运行：

```dockerfile
# backend/Dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Maven 包装脚本
COPY mvnw .
RUN chmod +x mvnw

# 复制 pom.xml 和源码
COPY pom.xml .
COPY src ./src

# 构建，跳过测试（测试在 CI 阶段做）
RUN ./mvnw package -DskipTests -Pprod

# 生产镜像用 JRE 而不是 JDK（体积更小）
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# 创建非 root 用户，增强安全性
RUN addgroup -g 1001 -S appgroup && \
    adduser  -u 1001 -S appuser -G appgroup

COPY --from=builder /app/target/myapp.jar app.jar

USER appuser

# JVM 内存和 GC 配置
ENV JAVA_OPTS="-Xmx512m -Xms256m -XX:+UseG1GC"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

几个值得说的点：`eclipse-temurin` 是 OpenJDK 的上游，替代了已经停更的 `openjdk` 官方镜像；生产镜像用 JRE 而非 JDK，体积差出几百兆；`adduser` 创建了非 root 用户，容器内如果存在安全漏洞也不会直接拿到 root 权限；`JAVA_OPTS` 设置了 JVM 内存上限，防止容器 OOM。

### Nginx 配置

前端 Docker 镜像里用到了 Nginx 配置：

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # HTML 不缓存（保证 SPA 路由正确）
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # API 代理到后端
    location /api/ {
        proxy_pass http://backend:8080/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时配置
        proxy_connect_timeout 10s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Vue Router history 模式 fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

`try_files $uri $uri/ /index.html` 这一行是 SPA 应用的关键——所有路由都回退到 index.html，让前端路由接管。如果漏掉这行，刷新页面就会报 404。

### docker-compose.yml

单体部署用 `docker-compose` 管理多个容器最方便：

```yaml
# docker-compose.yml
version: "3.9"

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_API_BASE: https://api.production.example.com
    ports:
      - "80:80"
    depends_on:
      backend:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: ${REDIS_HOST}
      REDIS_PORT: ${REDIS_PORT}
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    restart: unless-stopped
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  redis-data:
```

`depends_on` 配合 `condition: service_healthy` 确保后端完全启动之后前端才开始启动。健康检查用 Spring Boot Actuator 的 `/actuator/health` 端点，比自己写脚本稳定得多。

## Maven 构建脚本

Java 后端用 Maven 管理构建。`pom.xml` 里的生产配置是重点：

```xml
<!-- backend/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <profiles>
        <!-- 生产环境 profile -->
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
            </properties>
            <dependencies>
                <!-- 生产依赖 -->
            </dependencies>
        </profile>
    </profiles>

    <build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <!-- 打包成可执行 JAR -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.springframework.boot</groupId>
                            <artifactId>spring-boot-configuration-processor</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- 代码分析（可选，生产跳过） -->
            <plugin>
                <groupId>com.github.spotbugs</groupId>
                <artifactId>spotbugs-maven-plugin</artifactId>
                <version>4.8.3.1</version>
                <configuration>
                    <effort>Max</effort>
                    <threshold>Medium</threshold>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

构建命令：`mvn clean package -Pprod -DskipTests`。`-Pprod` 激活生产 profile，`-DskipTests` 在 CI 阶段跳过单元测试（CI 里有专门的测试步骤，不需要重复跑）。

## GitHub Actions CI/CD

代码推送到 GitHub 之后，CI/CD 流水线自动跑起来：构建、测试、打包、部署。

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '17'
  NODE_VERSION: '20'

jobs:
  # 阶段一：代码检查和测试
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      # 后端质量检查
      - name: Backend SpotBugs
        working-directory: backend
        run: mvn spotbugs:check

      - name: Backend Tests
        working-directory: backend
        env:
          SPRING_PROFILES_ACTIVE: test
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: testdb
          DB_USER: testuser
          DB_PASSWORD: testpass
        run: mvn test

      # 前端质量检查
      - name: Frontend Install
        working-directory: frontend
        run: npm ci

      - name: Frontend Type Check
        working-directory: frontend
        run: npm run typecheck

      - name: Frontend Tests
        working-directory: frontend
        run: npm test -- --coverage --ci

  # 阶段二：构建 Docker 镜像
  build:
    name: Build Docker Images
    runs-on: ubuntu-latest
    needs: quality
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: |
            myexample/backend:latest
            myexample/backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and Push Frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: |
            myexample/frontend:latest
            myexample/frontend:${{ github.sha }}
          build-args: |
            VITE_API_BASE=https://api.production.example.com
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # 阶段三：部署到服务器
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to Server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/myapp
            docker-compose pull
            docker-compose up -d
            docker image prune -f
```

这套流水线分三个阶段：质量检查（并行跑前后端的测试和静态分析）、构建镜像（只有在 main 分支 push 时才触发）、部署到生产服务器。关键是 `needs: quality` 确保了质量检查不通过就不会继续后面的流程。

## 环境配置管理

生产环境的配置管理是个敏感话题。数据库密码、API 密钥这些敏感信息不能硬编码在代码里，更不能提交到 GitHub。

Spring Boot 的做法是通过环境变量注入：

```yaml
# backend/src/main/resources/application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000

  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境不用 auto，用 Flyway 管理
    show-sql: false
    properties:
      hibernate:
        format_sql: false

  redis:
    host: ${REDIS_HOST}
    port: ${REDIS_PORT}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 2

server:
  port: 8080
  error:
    include-message: never  # 生产环境不暴露错误详情
    include-stacktrace: never

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when_authorized
```

敏感配置完全通过环境变量注入，`application-prod.yml` 里只有占位符。`ddl-auto: validate` 表示不自动创建表结构，生产环境的数据库变更由 Flyway 迁移脚本管理，保证每次部署的 schema 版本一致。

## 生产环境注意事项

说几个实际部署时容易踩的坑：

**健康检查不要只看进程。** 很多新手只配置了进程存活检查，但应用可能卡在启动一半、数据库连接失败、JVM OOM 的状态。Spring Boot Actuator 的 `/actuator/health` 会检查数据库、Redis 等关键依赖，比单纯的进程检查可靠得多。

**JVM 内存设置要给容器留余量。** Java 进程的 `-Xmx` 不要设成容器内存上限，要留出 10-20% 给操作系统和容器开销。否则在系统内存紧张时，OOM Killer 会直接杀掉容器，而不是触发 JVM 的垃圾回收。

**Nginx 的 worker 进程数。** 默认的 `worker_processes auto` 会自动匹配 CPU 核数，这个一般不需要改。但 `worker_connections` 默认是 1024，高并发场景下可能不够。

**日志收集。** Docker 容器的日志默认写到 `json-file`，时间长了会占用大量磁盘。用 `docker logs --tail` 临时看可以，长期还是要接 ELK 或 Loki 之类的日志收集系统。

**定期清理无用镜像。** CI 每次构建都会产生新镜像，时间长了服务器磁盘会被占满。`docker image prune -f` 可以清理 dangling 镜像，加入 CI 的清理步骤里。

## 结语

部署这件事，技术上不难，坑都在细节里。Docker 把环境差异消掉了，但网络、内存、日志、配置这些现实问题还是得一个一个面对。有了 GitHub Actions 自动化之后，部署从"手动操作"变成"代码控制"，出问题的概率小了很多，回滚也只需要一个 commit。

Vibe Coding 系列到这篇就告一段落了。从 Cursor 基础到 Composer、Rules、Memory，再到自定义 Agent、全栈开发、DevOps 部署，这十二篇文章覆盖了一个项目从想法到上线的完整链路。工具终究只是工具，真正让项目跑起来的是你投入的时间和思考。Cursor 能帮你省下不少重复劳动，但架构决策、代码质量、上线后的运维，这些还是得人来扛。
