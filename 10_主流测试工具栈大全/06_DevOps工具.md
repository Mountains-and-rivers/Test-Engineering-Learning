# 06 DevOps工具

## 目录

1. [DevOps与测试概述](#devops与测试概述)
2. [Jenkins CI/CD核心工具](#jenkins-cicd核心工具)
3. [GitLab CI 一体化DevOps](#gitlab-ci-一体化devops)
4. [GitHub Actions 云端CI/CD](#github-actions-云端cicd)
5. [Docker 容器化测试环境](#docker-容器化测试环境)
6. [Kubernetes 容器编排](#kubernetes-容器编排)
7. [Ansible/Terraform 基础设施即代码](#ansibleterraform-基础设施即代码)
8. [5个DevOps测试集成实践](#5个devops测试集成实践)

---

## DevOps与测试概述

### DevOps生命周期

```
┌──────────────────────────────────────────────────────────┐
│                       DevOps 循环                        │
│                                                          │
│    Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → ...
│                                                          │
│    测试介入的每个阶段:                                     │
│    Plan:  测试策略、测试计划                               │
│    Code:  单元测试、代码审查、SAST                        │
│    Build: 编译检查、单元测试、依赖扫描                     │
│    Test:  集成测试、API测试、UI测试、性能测试、安全测试    │
│    Release: 冒烟测试、UAT验收                            │
│    Deploy: 金丝雀发布验证、蓝绿部署验证                    │
│    Operate/Monitor: 线上监控、告警、混沌工程              │
└──────────────────────────────────────────────────────────┘
```

### 测试在CI/CD流水线中的位置

```yaml
# 标准CI/CD流水线中的测试阶段
pipeline:
  stages:
    - compile          # 编译
    - unit-test        # 单元测试 (快, <5分钟)
    - code-analysis    # 静态代码分析
    - build            # 构建镜像
    - integration-test # 集成/API测试 (<15分钟)
    - security-scan    # 安全扫描
    - ui-test          # UI自动化测试
    - performance-test # 性能测试 (可选, 夜间执行)
    - deploy-staging   # 部署到预发布环境
    - smoke-test       # 冒烟测试
    - deploy-prod      # 部署到生产
    - post-deploy-test # 生产验证
```

---

## Jenkins CI/CD核心工具

### 2.1 Jenkins 概述

Jenkins 是最流行的开源CI/CD工具，拥有庞大的插件生态(1800+)和最大的社区支持。

**安装：**
```bash
# Docker 安装（推荐）
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# 获取初始密码
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# War包方式
java -jar jenkins.war --httpPort=9090

# Ubuntu
sudo apt install jenkins
```

### 2.2 Pipeline as Code

Jenkins Pipeline 使用 Groovy DSL 定义CI/CD流程，管道即代码，纳入版本管理。

**Declarative Pipeline 完整示例：**
```groovy
pipeline {
    agent any

    // 环境变量
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        APP_NAME = 'my-application'
        SONAR_HOST = 'http://sonarqube:9000'
    }

    // 构建参数
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: '部署环境')
        booleanParam(name: 'RUN_PERF_TEST', defaultValue: false, description: '是否执行性能测试')
        string(name: 'VERSION', defaultValue: '1.0.0', description: '版本号')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Compile & Unit Test') {
            steps {
                sh '''
                    mvn clean compile
                    mvn test
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${APP_NAME}:${params.VERSION}")
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_REGISTRY}/${APP_NAME}:${params.VERSION}").push()
                    }
                }
            }
        }

        stage('Deploy to Test') {
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                      ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${params.VERSION} \
                      -n testing
                    kubectl rollout status deployment/${APP_NAME} -n testing
                """
            }
        }

        stage('API Test') {
            steps {
                sh '''
                    newman run collections/api_tests.json \
                      -e environments/test.json \
                      --reporters cli,junit,htmlextra \
                      --reporter-junit-export results/api-junit.xml
                '''
            }
            post {
                always {
                    junit 'results/api-junit.xml'
                }
            }
        }

        stage('UI Test') {
            steps {
                sh '''
                    cd e2e-tests
                    npm install
                    npx playwright test --reporter=junit
                '''
            }
            post {
                always {
                    junit 'e2e-tests/results/junit.xml'
                }
            }
        }

        stage('Performance Test') {
            when {
                expression { params.RUN_PERF_TEST }
            }
            steps {
                sh '''
                    jmeter -n -t performance/test_plan.jmx \
                      -l results/perf-results.jtl \
                      -e -o results/perf-report/
                '''
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "确认发布到生产环境？"
                ok "发布"
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                      ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${params.VERSION} \
                      -n production
                """
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    # 生产环境冒烟测试
                    curl -f https://api.example.com/health || exit 1
                    curl -f https://api.example.com/api/v1/version || exit 1
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "[Jenkins] ${env.JOB_NAME} #${env.BUILD_NUMBER} 构建成功",
                body: "构建成功！版本: ${params.VERSION}",
                to: 'team@example.com'
            )
        }
        failure {
            emailext(
                subject: "[Jenkins] ${env.JOB_NAME} #${env.BUILD_NUMBER} 构建失败",
                body: "构建失败！请查看日志: ${env.BUILD_URL}",
                to: 'team@example.com'
            )
        }
        always {
            cleanWs()
        }
    }
}
```

### 2.3 Jenkins 共享库

将通用逻辑抽取为共享库，在多个Pipeline中复用：

```groovy
// vars/testRunner.groovy - 共享库中的自定义步骤

def runUnitTest(String projectDir) {
    sh """
        cd ${projectDir}
        mvn test -Dmaven.test.failure.ignore=false
    """
    junit "${projectDir}/target/surefire-reports/*.xml"
}

def runApiTest(String collectionFile, String environmentFile) {
    sh """
        newman run ${collectionFile} \
          -e ${environmentFile} \
          --reporters cli,junit \
          --reporter-junit-export api-results.xml
    """
    junit 'api-results.xml'
}

def runSeleniumTest(String testDir) {
    sh """
        cd ${testDir}
        mvn test -Dtest=**/*Test -Dbrowser=chrome-headless
    """
    junit "${testDir}/target/failsafe-reports/*.xml"
}

def runSecurityScan(String targetUrl) {
    sh """
        docker run --rm -v \$(pwd):/zap/wrk/:rw owasp/zap2docker-stable \
          zap-full-scan.py -t ${targetUrl} -x zap_report.xml
    """
}

// Pipeline中使用
@Library('my-shared-library') _

pipeline {
    stages {
        stage('Test') {
            steps {
                testRunner.runUnitTest('.')
                testRunner.runApiTest('collections/api.json', 'environments/test.json')
                testRunner.runSeleniumTest('selenium-tests')
            }
        }
    }
}
```

### 2.4 Jenkins Agent (分布式构建)

```bash
# 启动Agent (Java Web Start方式)
java -jar agent.jar -jnlpUrl http://jenkins-master:8080/computer/agent1/jenkins-agent.jnlp \
  -secret <secret> -workDir "/home/jenkins/agent"

# Docker Agent (动态Agent)
# Jenkins 自动创建和销毁Docker容器作为Agent
```

**Pipeline中指定Agent：**
```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args '-v /tmp/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Selenium Test') {
            agent {
                docker {
                    image 'selenium/standalone-chrome:latest'
                    args '-v /dev/shm:/dev/shm --shm-size=2g'
                }
            }
            steps {
                sh 'mvn test -Dtest=SeleniumTest'
            }
        }
    }
}
```

---

## GitLab CI 一体化DevOps

### 3.1 概述

GitLab 提供了从代码托管、CI/CD、安全扫描到部署监控的一体化平台。

### 3.2 .gitlab-ci.yml 完整配置

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - deploy-staging
  - e2e-test
  - deploy-prod

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_REGISTRY: registry.gitlab.com
  APP_NAME: my-app

# 缓存Maven依赖，加速构建
cache:
  paths:
    - .m2/repository/
    - node_modules/

# ==================== 构建阶段 ====================
compile:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn compile
  artifacts:
    paths:
      - target/

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
    paths:
      - target/
  coverage: '/Total.*?([0-9]{1,3})%/'  # 提取覆盖率

build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - main
    - develop

# ==================== 测试阶段 ====================
api-test:
  stage: test
  image:
    name: postman/newman:latest
    entrypoint: [""]
  script:
    - newman run collections/api_tests.json
      -e environments/testing.json
      --reporters cli,junit
      --reporter-junit-export report.xml
  artifacts:
    reports:
      junit: report.xml
  needs: ["deploy-staging"]

integration-test:
  stage: test
  services:
    - postgres:15
    - redis:latest
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_pass
    DATABASE_URL: jdbc:postgresql://postgres:5432/test_db
    REDIS_URL: redis://redis:6379
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn verify -P integration-test
  artifacts:
    reports:
      junit:
        - target/failsafe-reports/TEST-*.xml

# ==================== 安全扫描阶段 ====================
sast:
  stage: security
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
      -Dsonar.qualitygate.wait=true

dependency-scan:
  stage: security
  image: owasp/dependency-check:latest
  script:
    - /usr/share/dependency-check/bin/dependency-check.sh
      --scan .
      --format HTML
      --out report.html
  artifacts:
    paths:
      - report.html

container-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL
      $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA

# ==================== 部署阶段 ====================
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME
      $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
      -n staging
  environment:
    name: staging
    url: https://staging.example.com

e2e-test:
  stage: e2e-test
  image: mcr.microsoft.com/playwright:latest
  script:
    - cd e2e
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - e2e/playwright-report/
      - e2e/test-results/
  needs: ["deploy-staging"]

# ==================== 生产部署 ====================
deploy-prod:
  stage: deploy-prod
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME
      $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHORT_SHA
      -n production
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
  needs: ["e2e-test"]
```

---

## GitHub Actions 云端CI/CD

### 4.1 概述

GitHub Actions 是GitHub内置的CI/CD服务，与代码仓库深度集成。

### 4.2 测试工作流完整配置

```yaml
# .github/workflows/test-pipeline.yml
name: Test Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨2点执行完整测试

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==================== 单元测试 ====================
  unit-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: ['17', '21']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
          cache: maven
      - name: Run Unit Tests
        run: mvn test -B
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results-${{ matrix.java-version }}
          path: target/surefire-reports/

  # ==================== API接口测试 ====================
  api-test:
    runs-on: ubuntu-latest
    needs: [unit-test]
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - name: Start Application
        run: |
          docker compose up -d app
          sleep 30
      - name: Install Newman
        run: npm install -g newman newman-reporter-htmlextra
      - name: Run API Tests
        run: |
          newman run collections/api_tests.json \
            -e environments/ci.json \
            --reporters cli,junit,htmlextra \
            --reporter-junit-export api-results.xml \
            --reporter-htmlextra-export api-report.html
      - name: Upload API Test Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-report
          path: |
            api-results.xml
            api-report.html

  # ==================== E2E测试 (Playwright) ====================
  e2e-test:
    runs-on: ubuntu-latest
    needs: [api-test]
    strategy:
      matrix:
        browser: [chromium, firefox]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: e2e/package-lock.json
      - name: Install dependencies
        run: |
          cd e2e
          npm ci
      - name: Install Playwright Browsers
        run: |
          cd e2e
          npx playwright install --with-deps ${{ matrix.browser }}
      - name: Run E2E Tests
        run: |
          cd e2e
          npx playwright test --project=${{ matrix.browser }}
      - name: Upload Playwright Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.browser }}
          path: |
            e2e/playwright-report/
            e2e/test-results/

  # ==================== 安全扫描 ====================
  security-scan:
    runs-on: ubuntu-latest
    needs: [unit-test]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: SonarQube Scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'HIGH,CRITICAL'

  # ==================== 性能测试 (定时执行) ====================
  performance-test:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    needs: [api-test]
    steps:
      - uses: actions/checkout@v4
      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: performance/k6-script.js
          flags: --out json=results.json
      - name: Upload Performance Results
        uses: actions/upload-artifact@v4
        with:
          name: performance-results
          path: results.json

  # ==================== 汇总通知 ====================
  notify:
    if: always()
    runs-on: ubuntu-latest
    needs: [unit-test, api-test, e2e-test, security-scan]
    steps:
      - name: Notify Test Results
        uses: actions/github-script@v7
        with:
          script: |
            const { data: artifacts } = await github.rest.actions.listWorkflowRunArtifacts({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: context.runId,
            });
            console.log('Test artifacts:', artifacts.artifacts.map(a => a.name));
```

---

## Docker 容器化测试环境

### 5.1 Docker 基础

```bash
# 常用Docker命令
docker pull <image>           # 拉取镜像
docker build -t name:tag .    # 构建镜像
docker run -d --name container image  # 后台运行容器
docker ps                     # 查看运行中容器
docker ps -a                  # 查看所有容器
docker logs -f container      # 查看容器日志
docker exec -it container bash # 进入容器
docker stop container         # 停止容器
docker rm container           # 删除容器
docker rmi image              # 删除镜像
docker compose up -d          # 启动Compose服务
docker compose down           # 停止并删除Compose服务
```

### 5.2 测试环境 Dockerfile

```dockerfile
# Dockerfile - 测试环境
FROM maven:3.9-eclipse-temurin-17

# 安装测试依赖
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    jq \
    chromium \
    chromium-driver \
    firefox-esr \
    && rm -rf /var/lib/apt/lists/*

# 安装 Node.js (用于Playwright/Newman)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# 安装 Newman
RUN npm install -g newman newman-reporter-htmlextra

# 设置Chrome环境变量 (Selenium)
ENV CHROME_BIN=/usr/bin/chromium
ENV CHROMEDRIVER_BIN=/usr/bin/chromedriver

# 设置工作目录
WORKDIR /tests

# 复制测试代码
COPY pom.xml .
COPY src/ src/

# 运行测试
CMD ["mvn", "test"]
```

### 5.3 Docker Compose 集成测试环境

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  # 被测应用
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/testdb
      - SPRING_REDIS_HOST=redis
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      retries: 5

  # 数据库
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      retries: 10

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 10

  # 消息队列
  kafka:
    image: bitnami/kafka:3.6
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
    ports:
      - "9092:9092"

  # 接口测试
  api-tests:
    image: postman/newman:latest
    volumes:
      - ./collections:/etc/newman
      - ./environments:/etc/newman/envs
    command:
      - run
      - /etc/newman/api_tests.json
      - -e
      - /etc/newman/envs/test_env.json
      - --reporters
      - cli,htmlextra
    depends_on:
      app:
        condition: service_healthy

  # UI测试
  ui-tests:
    build:
      context: ./e2e-tests
      dockerfile: Dockerfile
    volumes:
      - ./e2e-tests/results:/app/results
    environment:
      - APP_URL=http://app:8080
      - BROWSER=headless
    depends_on:
      app:
        condition: service_healthy

  # 性能测试
  perf-tests:
    image: grafana/k6:latest
    volumes:
      - ./performance:/scripts
    command: run /scripts/k6-test.js
    depends_on:
      app:
        condition: service_healthy
```

**运行集成测试环境：**
```bash
# 启动环境并运行测试
docker compose -f docker-compose.test.yml up \
  --abort-on-container-exit \
  --exit-code-from api-tests

# 仅启动环境（不运行测试）
docker compose -f docker-compose.test.yml up -d app mysql redis

# 清理环境
docker compose -f docker-compose.test.yml down -v
```

---

## Kubernetes 容器编排

### 6.1 核心概念

```
Kubernetes 核心资源：
├── Pod: 最小部署单元（一个或多个容器）
├── Deployment: 无状态应用部署
├── StatefulSet: 有状态应用（数据库）
├── Service: 服务发现与负载均衡
├── Ingress: HTTP/HTTPS路由
├── ConfigMap: 配置管理
├── Secret: 密钥管理
├── PersistentVolume: 持久化存储
├── Namespace: 资源隔离
└── Job/CronJob: 定时/一次性任务
```

### 6.2 测试环境的Kubernetes部署

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-test
  namespace: testing
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: test
    spec:
      containers:
        - name: app
          image: registry.example.com/my-app:${VERSION}
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "test"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: app-test-svc
  namespace: testing
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

### 6.3 测试Job

```yaml
# k8s/test-job.yaml - 每次部署后自动执行测试
apiVersion: batch/v1
kind: Job
metadata:
  name: api-test-job-${BUILD_ID}
  namespace: testing
spec:
  ttlSecondsAfterFinished: 3600  # 1小时后自动清理
  backoffLimit: 2  # 失败重试次数
  template:
    spec:
      containers:
        - name: newman
          image: postman/newman:latest
          command:
            - newman
            - run
            - /collections/api_tests.json
            - -e
            - /environments/test.json
            - --reporters
            - cli,junit
          env:
            - name: BASE_URL
              value: "http://app-test-svc.testing.svc.cluster.local"
          volumeMounts:
            - name: test-collections
              mountPath: /collections
            - name: test-environments
              mountPath: /environments
      volumes:
        - name: test-collections
          configMap:
            name: test-collections
        - name: test-environments
          configMap:
            name: test-environments
      restartPolicy: Never
```

### 6.4 使用Helm管理测试Chart

```yaml
# helm/charts/my-app/values-test.yaml
replicaCount: 2

image:
  repository: registry.example.com/my-app
  tag: "latest"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hosts:
    - host: test.example.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

test:
  enabled: true
  image: postman/newman:latest
  collection: api_tests.json
  environment: test.json
```

```bash
# 部署测试环境
helm upgrade --install my-app-test ./helm/charts/my-app \
  --namespace testing \
  --values ./helm/charts/my-app/values-test.yaml \
  --set image.tag=$BUILD_TAG

# 运行测试
helm test my-app-test -n testing
```

---

## Ansible/Terraform 基础设施即代码

### 7.1 Ansible 测试环境配置

```yaml
# ansible/test-environment.yml
---
- name: Configure Test Environment
  hosts: test_servers
  become: yes
  vars:
    java_version: "17"
    maven_version: "3.9.6"

  tasks:
    - name: Install Java
      apt:
        name: "openjdk-{{ java_version }}-jdk"
        state: present

    - name: Install Maven
      unarchive:
        src: "https://dlcdn.apache.org/maven/maven-3/{{ maven_version }}/binaries/apache-maven-{{ maven_version }}-bin.tar.gz"
        dest: /opt
        remote_src: yes

    - name: Configure Maven PATH
      lineinfile:
        path: /etc/profile.d/maven.sh
        line: 'export PATH=/opt/apache-maven-{{ maven_version }}/bin:$PATH'
        create: yes

    - name: Install Docker
      apt:
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Install Testing Tools
      apt:
        name:
          - jq
          - curl
          - chromium-browser
        state: present

    - name: Install Newman
      npm:
        name: newman
        global: yes
```

### 7.2 Terraform 测试基础设施

```hcl
# terraform/test-infra/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 创建测试环境VPC
resource "aws_vpc" "test" {
  cidr_block = "10.100.0.0/16"
  tags = {
    Name        = "test-environment"
    Environment = "test"
  }
}

# 创建测试EC2实例
resource "aws_instance" "test_runner" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04
  instance_type = "t3.medium"

  vpc_security_group_ids = [aws_security_group.test_sg.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y docker.io openjdk-17-jdk maven

    # 安装测试工具
    npm install -g newman
    npm install -g playwright

    # 拉取测试镜像
    docker pull postman/newman:latest
    docker pull grafana/k6:latest
  EOF

  tags = {
    Name        = "test-runner"
    Environment = "test"
  }
}
```

---

## 5个DevOps测试集成实践

### 实践1: 测试左移 (Shift-Left Testing)

```groovy
// 在pre-commit阶段运行快速检查
// .git/hooks/pre-commit (或使用pre-commit框架)

#!/bin/bash
echo "Running pre-commit tests..."

# 1. 代码风格检查
mvn checkstyle:check || exit 1

# 2. 快速单元测试（仅运行变更相关的）
mvn test -pl $(git diff --name-only HEAD | grep 'src/main' | cut -d'/' -f1 | sort -u | tr '\n' ',') 2>/dev/null

# 3. 密钥泄露检查
gitleaks detect --source=. --verbose || exit 1

echo "Pre-commit checks passed!"
```

### 实践2: 渐进式测试策略

```yaml
# 分层执行策略
# 基于变更影响范围决定运行哪些测试

test_strategy:
  # 文档变更 → 跳过所有测试
  - if: "**/*.md"
    run: []

  # 配置文件变更 → 仅运行冒烟测试
  - if: "**/application*.yml"
    run: [smoke-test]

  # 代码变更 → 运行单元+API测试
  - if: "src/main/**"
    run: [unit-test, api-test]

  # 前端变更 → 运行UI测试
  - if: "src/frontend/**"
    run: [unit-test, ui-test]

  # 安全相关代码 → 额外运行安全扫描
  - if: "src/main/**/security/**"
    run: [unit-test, api-test, security-scan]

  # 数据库变更 → 运行集成测试
  - if: "**/db/migration/**"
    run: [unit-test, integration-test, api-test]
```

### 实践3: 测试结果聚合与可视化

```yaml
# Allure Framework 测试报告聚合
# .gitlab-ci.yml 示例
generate-report:
  stage: report
  needs:
    - unit-test
    - api-test
    - e2e-test
  image: frankescobar/allure-docker-service:latest
  script:
    - mkdir -p allure-results
    - |
      for dir in unit api e2e; do
        cp $dir/allure-results/*.json allure-results/ 2>/dev/null || true
      done
    - allure generate allure-results -o allure-report
  artifacts:
    paths:
      - allure-report/
```

### 实践4: 环境复制 (Ephemeral Environment)

为每个PR/分支创建独立的临时测试环境：

```yaml
# 动态创建测试环境
create-env:
  stage: deploy
  script:
    - export NAMESPACE="pr-${CI_MERGE_REQUEST_IID}"
    # 创建K8s Namespace
    - kubectl create namespace ${NAMESPACE} || true
    # 部署应用
    - helm upgrade --install app ./charts/app
        --namespace ${NAMESPACE}
        --set image.tag=${CI_COMMIT_SHORT_SHA}
        --set ingress.host=${NAMESPACE}.test.example.com
    - echo "测试环境: https://${NAMESPACE}.test.example.com"
  environment:
    name: review/${CI_MERGE_REQUEST_IID}
    url: https://pr-${CI_MERGE_REQUEST_IID}.test.example.com
    on_stop: destroy-env

destroy-env:
  stage: deploy
  when: manual
  script:
    - export NAMESPACE="pr-${CI_MERGE_REQUEST_IID}"
    - helm uninstall app -n ${NAMESPACE}
    - kubectl delete namespace ${NAMESPACE}
  environment:
    name: review/${CI_MERGE_REQUEST_IID}
    action: stop
```

### 实践5: 混沌工程入门

```yaml
# 使用Litmus Chaos在测试环境演练
# 注入网络延迟
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-delay-test
  namespace: testing
spec:
  appinfo:
    appns: 'testing'
    applabel: 'app=my-app'
    appkind: 'deployment'
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_LATENCY
              value: '2000'  # 2秒网络延迟
            - name: TARGET_CONTAINER
              value: 'app'
            - name: TOTAL_CHAOS_DURATION
              value: '60'  # 持续60秒
```

---

## 总结

DevOps工具的选择与团队规模和基础设施相关：

| 团队规模 | 推荐CI/CD | 推荐容器编排 | 推荐IaC |
|---------|----------|------------|---------|
| 个人/小团队 | GitHub Actions | Docker Compose | Terraform (简单) |
| 中型团队 | GitLab CI | Docker Compose / K8s | Terraform + Ansible |
| 大型企业 | Jenkins (灵活) | Kubernetes | Terraform Cloud |

**关键原则：**
1. **流水线即代码** - 所有的CI/CD配置应该版本化管理
2. **环境一致性** - 开发/测试/生产环境使用相同的容器化方式
3. **自动化测试门禁** - CI/CD中必须有自动化的质量门禁
4. **失败快速反馈** - 优化流水线速度，让问题尽早暴露
5. **监控与告警** - 将测试结果和产品质量指标可视化，及时告警
