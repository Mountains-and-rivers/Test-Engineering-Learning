# CI/CD集成

> **CI/CD集成**（Continuous Integration / Continuous Delivery Integration）：将自动化测试嵌入到持续集成和持续交付流水线中，实现代码提交即触发测试、测试通过即自动部署、测试失败即阻断发版的完整质量门禁。

---

## 目录

1. [Jenkins/GitLab CI/GitHub Actions集成](#1-jenkinsgitlab-cigithub-actions集成)
2. [代码提交触发自动化测试](#2-代码提交触发自动化测试)
3. [定时构建与测试](#3-定时构建与测试)
4. [测试结果与构建状态联动](#4-测试结果与构建状态联动)
5. [Docker容器化执行测试](#5-docker容器化执行测试)
6. [并行分布式测试执行](#6-并行分布式测试执行)
7. [5个CI/CD集成实战案例](#7-5个cicd集成实战案例)

---

## 1. Jenkins/GitLab CI/GitHub Actions集成

### 通俗释义

CI/CD平台就像是"自动化测试的管家"——你写好测试代码后，它帮你自动拉代码、编译、执行测试、生成报告、通知结果。不同的管家有不同的特长：Jenkins最灵活全能、GitLab CI最简洁、GitHub Actions与代码仓库集成最紧密。

### 详细解释

#### 三大平台对比

| 特性 | Jenkins | GitLab CI | GitHub Actions |
|------|---------|-----------|----------------|
| **部署方式** | 自建服务器 | GitLab自带 / 自建Runner | GitHub自带 |
| **配置格式** | Jenkinsfile (Groovy) | .gitlab-ci.yml (YAML) | .github/workflows/*.yml |
| **插件生态** | 极丰富 (1800+) | 较少 | 丰富 (Marketplace) |
| **并行能力** | 强 (Master-Agent) | 强 (多Runner) | 强 (Matrix) |
| **可视化** | 经典Pipeline视图 | 内置Pipeline图 | 内置可视化 |
| **与代码仓库** | 独立配置 | 代码内配置 | 代码内配置 |
| **适合团队** | 中大团队 | 使用GitLab的团队 | 使用GitHub的团队 |

#### Jenkins Pipeline完整示例

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    // 全局环境变量
    environment {
        TEST_ENV = 'test'
        MAVEN_OPTS = '-Xmx2048m'
    }
    
    // 构建参数（手动触发时可选择）
    parameters {
        choice(name: 'TEST_ENV', choices: ['test', 'staging'], description: '测试环境')
        choice(name: 'TEST_TYPE', choices: ['smoke', 'regression', 'all'], description: '测试类型')
        string(name: 'BRANCH', defaultValue: 'main', description: '测试分支')
        booleanParam(name: 'SEND_NOTIFICATION', defaultValue: true, description: '是否发送通知')
    }
    
    // 定时触发（每天凌晨2点全量回归）
    triggers {
        cron('0 2 * * *')
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH}", 
                    url: 'https://github.com/example/project.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }
        
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Automation Test') {
            // 并行执行不同模块的自动化测试
            parallel {
                stage('用户模块测试') {
                    steps {
                        sh '''
                            mvn test -pl user-module \
                                -Dtest.env=${TEST_ENV} \
                                -Dgroups=${params.TEST_TYPE}
                        '''
                    }
                }
                stage('订单模块测试') {
                    steps {
                        sh '''
                            mvn test -pl order-module \
                                -Dtest.env=${TEST_ENV} \
                                -Dgroups=${params.TEST_TYPE}
                        '''
                    }
                }
                stage('支付模块测试') {
                    steps {
                        sh '''
                            mvn test -pl payment-module \
                                -Dtest.env=${TEST_ENV} \
                                -Dgroups=${params.TEST_TYPE}
                        '''
                    }
                }
            }
        }
        
        stage('Generate Report') {
            steps {
                // 生成Allure报告
                sh 'mvn allure:report'
                // 生成自定义报告
                sh 'java -jar report-generator.jar --input allure-results --output reports/'
            }
        }
        
        stage('Publish Report') {
            steps {
                // 发布Allure报告
                allure includeProperties: false,
                    results: [[path: 'target/allure-results']]
                
                // 归档报告文件
                archiveArtifacts artifacts: 'reports/**/*', 
                    fingerprint: true
            }
        }
    }
    
    post {
        always {
            // 清理工作空间
            cleanWs()
        }
        success {
            // 测试全部通过
            emailext(
                subject: "[${currentBuild.result}] ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """测试全部通过!
                    环境: ${params.TEST_ENV}
                    类型: ${params.TEST_TYPE}
                    报告: ${env.BUILD_URL}allure
                """,
                to: 'team@example.com'
            )
        }
        failure {
            // 测试失败
            emailext(
                subject: "[FAILED] ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """测试存在失败用例!
                    环境: ${params.TEST_ENV}
                    失败数: 请查看报告详情
                    报告: ${env.BUILD_URL}allure
                """,
                to: 'team@example.com, manager@example.com'
            )
            
            // 钉钉通知
            dingtalk(
                robot: 'test-robot',
                type: 'MARKDOWN',
                title: '自动化测试失败通知',
                text: [
                    "## 自动化测试失败",
                    "- 项目: ${env.JOB_NAME}",
                    "- 构建: #${env.BUILD_NUMBER}",
                    "- [查看报告](${env.BUILD_URL}allure)"
                ]
            )
        }
    }
}
```

#### GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
variables:
  MAVEN_OPTS: "-Xmx2048m"
  TEST_ENV: "test"

# 定义阶段顺序
stages:
  - build
  - test
  - report
  - deploy

# 缓存Maven依赖
cache:
  paths:
    - .m2/repository/

# 构建阶段
build:
  stage: build
  script:
    - mvn clean compile -DskipTests
  artifacts:
    paths:
      - target/
    expire_in: 1 day

# 冒烟测试（每次提交触发）
smoke-test:
  stage: test
  script:
    - mvn test -Dgroups=smoke -Dtest.env=${TEST_ENV}
  artifacts:
    when: always
    paths:
      - target/allure-results/
      - target/surefire-reports/
    expire_in: 7 days
  only:
    - merge_requests
    - main

# 全量回归测试（仅main分支定时执行）
regression-test:
  stage: test
  script:
    - mvn test -Dgroups=regression -Dtest.env=${TEST_ENV}
  artifacts:
    when: always
    paths:
      - target/allure-results/
    expire_in: 30 days
  only:
    refs:
      - schedules    # 仅定时触发
    variables:
      - $TEST_TYPE == "regression"

# 生成Allure报告
generate-report:
  stage: report
  script:
    - mvn allure:report
    - cp -r target/allure-report/ public/
  artifacts:
    paths:
      - public/
  needs: ["smoke-test", "regression-test"]

# 部署报告到Pages
pages:
  stage: deploy
  script:
    - mkdir -p public
    - cp -r target/allure-report/* public/
  artifacts:
    paths:
      - public
  only:
    - main
```

#### GitHub Actions Pipeline

```yaml
# .github/workflows/automation-test.yml
name: 自动化测试

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨2点
  workflow_dispatch:       # 手动触发
    inputs:
      test_env:
        description: '测试环境'
        required: true
        default: 'test'
        type: choice
        options: ['test', 'staging']
      test_type:
        description: '测试类型'
        default: 'regression'
        type: choice
        options: ['smoke', 'regression', 'all']

env:
  TEST_ENV: ${{ github.event.inputs.test_env || 'test' }}
  TEST_TYPE: ${{ github.event.inputs.test_type || 'smoke' }}

jobs:
  # 冒烟测试（PR时快速检查）
  smoke-test:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      
      - name: 冒烟测试
        run: mvn test -Dgroups=smoke -Dtest.env=${{ env.TEST_ENV }}
      
      - name: 上传测试结果
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: smoke-test-results
          path: target/surefire-reports/

  # 全量回归测试（定时或手动触发）
  regression-test:
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    strategy:
      fail-fast: false
      matrix:
        module: [user, order, payment, product]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      
      - name: ${{ matrix.module }}模块测试
        run: |
          mvn test -pl ${{ matrix.module }}-module \
            -Dgroups=${{ env.TEST_TYPE }} \
            -Dtest.env=${{ env.TEST_ENV }}
      
      - name: 上传Allure结果
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-${{ matrix.module }}
          path: target/allure-results/

  # 汇总报告
  generate-report:
    needs: [smoke-test, regression-test]
    if: always()
    runs-on: ubuntu-latest
    
    steps:
      - name: 下载所有结果
        uses: actions/download-artifact@v4
        with:
          path: allure-results-all/
      
      - name: 生成Allure报告
        uses: simple-elf/allure-report-action@master
        with:
          allure_results: allure-results-all
          allure_history: allure-history
      
      - name: 发布报告到GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
          keep_files: true
      
      - name: 发送Slack通知
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "自动化测试失败！\n项目: ${{ github.repository }}\n分支: ${{ github.ref_name }}\n报告: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 2. 代码提交触发自动化测试

### 通俗释义

代码提交触发是CI/CD的核心价值——开发人员每次提交代码（push / merge request），系统自动跑一遍冒烟测试。如果测试失败，代码就不能合入。这就像每个进入大楼的人都要过安检，把问题挡在主分支门外。

### 详细解释

#### 触发策略矩阵

| 触发事件 | 执行范围 | 超时限制 | 失败策略 |
|----------|----------|----------|----------|
| **Git Push（feature分支）** | 单元测试 | < 5分钟 | 仅通知 |
| **Merge Request / Pull Request** | 冒烟测试 (smoke) | < 15分钟 | 阻断合入 |
| **Merge到Main分支** | 全量回归测试 | < 60分钟 | 阻断+通知 |
| **Tag Release** | 全量+性能+安全 | < 120分钟 | 阻断发版 |

#### 智能增量测试选择

```java
// SmartTestSelector.java
@Component
public class SmartTestSelector {
    
    /**
     * 根据代码变更智能选择需要执行的测试
     */
    public Set<String> selectAffectedTests(List<String> changedFiles) {
        Set<String> selectedTests = new HashSet<>();
        
        // 1. 静态映射：文件路径 -> 测试类
        Map<String, String> staticMapping = loadMappingFromConfig();
        
        // 2. 历史数据映射：基于历史测试失败数据
        Map<String, Set<String>> historicalMapping = loadHistoricalMapping();
        
        // 3. 代码依赖分析：通过import/依赖关系
        Map<String, Set<String>> dependencyMapping = analyzeDependencies();
        
        for (String file : changedFiles) {
            // 静态映射
            if (staticMapping.containsKey(file)) {
                selectedTests.add(staticMapping.get(file));
            }
            
            // 代码变更对应的模块
            String module = extractModule(file);
            if (module != null) {
                selectedTests.addAll(
                    getAllTestsForModule(module, "smoke"));
            }
            
            // 依赖分析
            if (dependencyMapping.containsKey(file)) {
                selectedTests.addAll(dependencyMapping.get(file));
            }
        }
        
        log.info("代码变更: {} 个文件 -> 选择: {} 个测试用例", 
            changedFiles.size(), selectedTests.size());
        
        return selectedTests;
    }
    
    /**
     * 通过Git API获取变更文件列表
     */
    public List<String> getChangedFiles(String baseBranch, String targetBranch) {
        try {
            Process process = new ProcessBuilder(
                "git", "diff", "--name-only",
                baseBranch + "..." + targetBranch
            ).start();
            
            return new BufferedReader(
                new InputStreamReader(process.getInputStream()))
                .lines()
                .filter(f -> f.endsWith(".java") || f.endsWith(".xml") || 
                            f.endsWith(".properties") || f.endsWith(".yaml"))
                .collect(Collectors.toList());
        } catch (IOException e) {
            log.error("获取Git变更文件失败", e);
            return Collections.emptyList();
        }
    }
}
```

---

## 3. 定时构建与测试

### 通俗释义

定时构建就像"闹钟"——到了设定时间自动叫醒测试系统开始工作。最常见的模式是每天凌晨2点自动跑全量回归测试，这样早上上班时就能看到完整的测试报告。

### 详细解释

#### 定时策略设计

```
定时测试矩阵：

┌─────────────────┬──────────────┬──────────────┬──────────────┐
│    时间节点      │   执行范围    │   执行时长    │   适用场景    │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ 每4小时(工作日)  │ 冒烟测试      │ 10-15分钟    │ 持续质量监控  │
│ 每日凌晨2点      │ 全量回归      │ 30-90分钟    │ 日常质量检查  │
│ 每周一凌晨3点    │ 全量+稳定性   │ 2-4小时      │ 周版本验证    │
│ 每迭代结束前     │ 全量+性能     │ 4-8小时      │ 迭代交付验证  │
│ 发版窗口前       │ 全量+安全扫描  │ 6-12小时     │ 上线质量门禁  │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

#### Jenkins定时配置

```groovy
pipeline {
    agent any
    
    triggers {
        // 工作日每4小时冒烟测试
        cron('0 */4 * * 1-5')
        
        // 每天凌晨2点全量回归
        cron('0 2 * * *')
    }
    
    stages {
        stage('Schedule-Based Test Selection') {
            steps {
                script {
                    // 根据当前时间智能选择测试范围
                    def currentHour = new Date().format('HH').toInteger()
                    def dayOfWeek = new Date().format('u').toInteger()
                    
                    if (currentHour == 2) {
                        env.TEST_GROUPS = 'regression'
                        env.TEST_TIMEOUT = '90'
                    } else if (dayOfWeek == 1 && currentHour == 3) {
                        env.TEST_GROUPS = 'regression,stability'
                        env.TEST_TIMEOUT = '240'
                    } else {
                        env.TEST_GROUPS = 'smoke'
                        env.TEST_TIMEOUT = '15'
                    }
                    
                    echo "定时触发: 执行 ${env.TEST_GROUPS} 测试组"
                }
            }
        }
    }
}
```

---

## 4. 测试结果与构建状态联动

### 通俗释义

测试结果与构建状态联动就是"测试结果说了算"——测试全部通过，构建标记为成功（绿色）；测试有失败，构建标记为失败（红色）；测试有不稳定用例，构建标记为不稳定（黄色）。CI系统根据这个颜色自动决定下一步：合并代码、自动部署、还是打回修复。

### 详细解释

```groovy
// 测试结果与构建状态联动
pipeline {
    stages {
        stage('Test') {
            steps {
                // 执行测试
                sh 'mvn test -Dgroups=${TEST_GROUPS}'
            }
        }
    }
    
    post {
        success {
            // 全部通过 -> 自动部署到测试环境
            build job: 'deploy-to-test', parameters: [
                string(name: 'VERSION', value: "${env.BUILD_NUMBER}")
            ]
        }
        unstable {
            // 不稳定（有flaky用例）-> 通知但不阻断
            script {
                def flakyCount = readFile('reports/flaky-count.txt').trim()
                currentBuild.description = "不稳定: ${flakyCount}个Flaky用例"
                
                if (params.AUTO_MERGE && flakyCount.toInteger() <= 3) {
                    // flaky用例少，允许合并但标记
                    echo "Flaky用例在容忍范围内，允许继续"
                }
            }
        }
        failure {
            // 失败 -> 阻断后续流程
            script {
                currentBuild.description = "测试失败"
                
                // 自动创建JIRA Bug
                def failedTests = readFile('reports/failed-tests.txt').split('\n')
                for (test in failedTests.take(5)) {
                    jiraCreateIssue(
                        site: 'JIRA',
                        projectKey: 'BUG',
                        summary: "[自动化] ${test}",
                        description: "自动化测试发现失败用例: ${test}\n构建: ${env.BUILD_URL}",
                        type: 'Bug',
                        priority: 'High'
                    )
                }
            }
            
            // 回滚自动部署
            if (env.AUTO_DEPLOYED == 'true') {
                build job: 'rollback-deployment'
            }
        }
    }
}

// 质量门禁判断
def checkQualityGate() {
    // 读取测试结果
    def allureResults = readJSON file: 'allure-report/widgets/summary.json'
    def passRate = allureResults.statistic.passed.toFloat() / 
                   allureResults.statistic.total * 100
    
    def qualityGatePassed = true
    def reasons = []
    
    // 门禁条件
    if (passRate < 95) {
        qualityGatePassed = false
        reasons.add("通过率 ${passRate.round(1)}% < 95%")
    }
    
    if (allureResults.statistic.failed > 0) {
        def criticalFailures = readFile('reports/critical-failures.txt')
        if (criticalFailures.trim()) {
            qualityGatePassed = false
            reasons.add("存在P0级用例失败")
        }
    }
    
    if (allureResults.statistic.broken > 0) {
        qualityGatePassed = false
        reasons.add("存在Broken用例（环境/框架问题）")
    }
    
    if (!qualityGatePassed) {
        error "质量门禁未通过:\n${reasons.join('\n')}"
    }
}
```

---

## 5. Docker容器化执行测试

### 通俗释义

Docker容器化执行测试就是"把测试环境装进集装箱"——测试代码、依赖、浏览器、工具全部打包成一个镜像，在任何机器上都能一致地运行测试，彻底消除"在我机器上是好的"这类环境问题。

### 详细解释

#### Docker测试镜像

```dockerfile
# Dockerfile
FROM maven:3.9-eclipse-temurin-17

# 安装测试依赖
RUN apt-get update && apt-get install -y \
    google-chrome-stable \
    firefox-esr \
    && rm -rf /var/lib/apt/lists/*

# 安装ChromeDriver
RUN CHROME_VERSION=$(google-chrome --version | cut -d' ' -f3 | cut -d'.' -f1) && \
    wget -q "https://chromedriver.storage.googleapis.com/LATEST_RELEASE_${CHROME_VERSION}" \
    && CHROMEDRIVER_VERSION=$(cat LATEST_RELEASE_${CHROME_VERSION}) \
    && wget -q "https://chromedriver.storage.googleapis.com/${CHROMEDRIVER_VERSION}/chromedriver_linux64.zip" \
    && unzip chromedriver_linux64.zip -d /usr/local/bin/ \
    && chmod +x /usr/local/bin/chromedriver \
    && rm chromedriver_linux64.zip

WORKDIR /app

# 先复制pom.xml利用Docker缓存
COPY pom.xml .
RUN mvn dependency:resolve -DskipTests

# 复制源码和测试代码
COPY src/ src/
COPY testdata/ testdata/
COPY config/ config/

# 默认命令
CMD ["mvn", "test", "-Dtest.env=${TEST_ENV}", "-Dgroups=${TEST_GROUPS}"]
```

#### Docker Compose编排

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 测试执行器
  test-runner:
    build: .
    environment:
      - TEST_ENV=test
      - TEST_GROUPS=regression
      - DB_HOST=test-db
      - REDIS_HOST=test-redis
      - API_BASE_URL=http://mock-api:8080
    volumes:
      - ./reports:/app/reports
      - ./screenshots:/app/screenshots
      - ./logs:/app/logs
    depends_on:
      - test-db
      - test-redis
      - mock-api
    networks:
      - test-network

  # Selenium Grid
  selenium-hub:
    image: selenium/hub:latest
    ports:
      - "4444:4444"
    networks:
      - test-network

  chrome-node:
    image: selenium/node-chrome:latest
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_VNC_NO_PASSWORD=1
    depends_on:
      - selenium-hub
    networks:
      - test-network

  firefox-node:
    image: selenium/node-firefox:latest
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
    depends_on:
      - selenium-hub
    networks:
      - test-network

  # 测试数据库
  test-db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=testpass
      - MYSQL_DATABASE=test_db
    ports:
      - "3307:3306"
    volumes:
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - test-network

  # 测试Redis
  test-redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"
    networks:
      - test-network

  # Mock API服务
  mock-api:
    image: wiremock/wiremock:latest
    ports:
      - "8081:8080"
    volumes:
      - ./wiremock:/home/wiremock
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
```

#### Jenkins中使用Docker

```groovy
pipeline {
    agent {
        docker {
            image 'test-automation:latest'
            args '-v /var/run/docker.sock:/var/run/docker.sock --network host'
        }
    }
    
    stages {
        stage('Start Services') {
            steps {
                sh 'docker compose -f docker-compose.yml up -d test-db test-redis mock-api'
                sh 'sleep 10' // 等待服务启动
            }
        }
        
        stage('Run Tests') {
            steps {
                sh '''
                    mvn test \
                        -Dtest.env=test \
                        -Ddb.host=localhost \
                        -Ddb.port=3307
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                sh 'docker compose -f docker-compose.yml down -v'
            }
        }
    }
}
```

---

## 6. 并行/分布式测试执行

### 通俗释义

当用例数量从几百涨到几千甚至上万时，单机执行已经不够用了。并行/分布式执行就是把测试用例"分头发送"到多台机器上同时跑——1000个用例分给10台机器，每台只跑100个，总时间缩短到十分之一。

### 详细解释

#### Selenium Grid分布式Web测试

```java
// GridDistributedTest.java
public class GridDistributedTest {
    
    @ParameterizedTest
    @ValueSource(strings = {"chrome", "firefox", "edge"})
    public void testOnMultipleBrowsers(String browser) {
        RemoteWebDriver driver = null;
        try {
            // 连接Selenium Grid
            MutableCapabilities caps;
            switch (browser) {
                case "chrome":
                    caps = new ChromeOptions();
                    break;
                case "firefox":
                    caps = new FirefoxOptions();
                    break;
                default:
                    caps = new EdgeOptions();
            }
            
            driver = new RemoteWebDriver(
                new URL("http://selenium-hub:4444/wd/hub"), caps);
            
            // 执行测试
            driver.get("https://www.example.com");
            // ... 测试步骤
            
        } finally {
            if (driver != null) driver.quit();
        }
    }
}
```

#### Jenkins分布式构建

```groovy
pipeline {
    agent none  // 不在master上执行
    
    stages {
        stage('Distributed Tests') {
            parallel {
                stage('Chrome Tests') {
                    agent { label 'chrome-node' }
                    steps {
                        sh 'mvn test -Dgroups=chrome -pl web-tests'
                    }
                }
                stage('Firefox Tests') {
                    agent { label 'firefox-node' }
                    steps {
                        sh 'mvn test -Dgroups=firefox -pl web-tests'
                    }
                }
                stage('API Tests - Shard 1') {
                    agent { label 'linux' }
                    steps {
                        sh 'mvn test -Dshard=1 -DtotalShards=3 -pl api-tests'
                    }
                }
                stage('API Tests - Shard 2') {
                    agent { label 'linux' }
                    steps {
                        sh 'mvn test -Dshard=2 -DtotalShards=3 -pl api-tests'
                    }
                }
                stage('API Tests - Shard 3') {
                    agent { label 'linux' }
                    steps {
                        sh 'mvn test -Dshard=3 -DtotalShards=3 -pl api-tests'
                    }
                }
            }
        }
    }
}
```

#### 用例分片执行

```java
// TestSharding.java
public class TestSharding {
    
    /**
     * 将测试用例均匀分配到N个分片
     */
    public static <T> List<T> getShard(List<T> allTests, int shard, int totalShards) {
        List<T> shardTests = new ArrayList<>();
        for (int i = 0; i < allTests.size(); i++) {
            if (i % totalShards == shard) {
                shardTests.add(allTests.get(i));
            }
        }
        return shardTests;
    }
    
    /**
     * 按测试执行时间加权分片（更均匀）
     */
    public static List<TestSuite> weightedSharding(List<TestCase> allTests, 
                                                    int totalShards) {
        // 1. 获取各用例历史执行时间
        Map<String, Long> durations = TestDurationPredictor.loadHistory();
        
        // 2. 按执行时间降序排序
        List<TestCase> sorted = new ArrayList<>(allTests);
        sorted.sort((a, b) -> Long.compare(
            durations.getOrDefault(b.getName(), 30000L),
            durations.getOrDefault(a.getName(), 30000L)
        ));
        
        // 3. 贪心分配到各分片（尽量让各分片总时间均衡）
        List<TestSuite> shards = new ArrayList<>();
        for (int i = 0; i < totalShards; i++) {
            shards.add(new TestSuite(i));
        }
        
        for (TestCase test : sorted) {
            // 找到当前总时间最少的分片
            TestSuite minShard = shards.stream()
                .min(Comparator.comparingLong(TestSuite::getEstimatedDuration))
                .get();
            minShard.addTest(test, durations.getOrDefault(test.getName(), 30000L));
        }
        
        // 打印分片统计
        for (TestSuite shard : shards) {
            log.info("分片 {}: {} 个用例, 预估 {}s", 
                shard.getId(), shard.getTestCount(), 
                shard.getEstimatedDuration() / 1000);
        }
        
        return shards;
    }
}
```

---

## 7. 5个CI/CD集成实战案例

### 案例1：多分支测试策略

```groovy
// 根据分支自动选择测试策略
def getTestStrategy(branch) {
    switch (branch) {
        case ~/^feature\/.*/:
            return ['groups': 'unit', 'timeout': 300]
        case ~/^bugfix\/.*/:
            return ['groups': 'unit,smoke', 'timeout': 600]
        case 'develop':
            return ['groups': 'smoke,regression', 'timeout': 1800]
        case 'main':
            return ['groups': 'regression,stability', 'timeout': 3600]
        case ~/^release\/.*/:
            return ['groups': 'all', 'timeout': 7200]
        default:
            return ['groups': 'smoke', 'timeout': 900]
    }
}
```

### 案例2：测试环境自动申请与释放

```groovy
pipeline {
    stages {
        stage('Apply Environment') {
            steps {
                script {
                    // 从环境池中申请测试环境
                    def env = httpRequest(
                        url: 'http://env-pool.example.com/api/acquire',
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        requestBody: """{"team":"测试团队","duration":"2h"}"""
                    )
                    def envInfo = readJSON text: env.content
                    env.ENV_ID = envInfo.id
                    env.ENV_CONFIG = envInfo.config
                    
                    echo "申请到测试环境: ${env.ENV_ID}"
                }
            }
        }
        
        stage('Configure Environment') {
            steps {
                sh '''
                    # 更新hosts
                    echo "${ENV_CONFIG}" > env-config.yaml
                    # 初始化数据库
                    # 部署测试版本
                '''
            }
        }
        
        stage('Release Environment') {
            steps {
                script {
                    httpRequest(
                        url: "http://env-pool.example.com/api/release/${ENV_ID}",
                        httpMode: 'POST'
                    )
                    echo "测试环境已释放: ${ENV_ID}"
                }
            }
        }
    }
}
```

### 案例3：蓝绿部署与自动化测试

```groovy
pipeline {
    stages {
        stage('Deploy to Blue') {
            steps {
                // 部署到蓝色环境
                sh 'kubectl apply -f k8s/blue-deployment.yaml'
                sh 'kubectl rollout status deployment/app-blue'
            }
        }
        
        stage('Smoke Test Blue') {
            steps {
                sh 'mvn test -Dgroups=smoke -Dapi.url=http://app-blue.example.com'
            }
        }
        
        stage('Switch Traffic') {
            when { expression { currentBuild.result == 'SUCCESS' } }
            steps {
                // 蓝绿切换：流量切到蓝色环境
                sh 'kubectl patch service app -p \'{"spec":{"selector":{"version":"blue"}}}\''
            }
        }
        
        stage('Regression Test') {
            steps {
                sh 'mvn test -Dgroups=regression -Dapi.url=http://app.example.com'
            }
        }
        
        stage('Rollback if Failed') {
            when { expression { currentBuild.result == 'FAILURE' } }
            steps {
                // 回滚：流量切回绿色环境
                sh 'kubectl patch service app -p \'{"spec":{"selector":{"version":"green"}}}\''
            }
        }
    }
}
```

### 案例4：测试结果自动重跑

```groovy
pipeline {
    stages {
        stage('First Run') {
            steps {
                sh 'mvn test -Dgroups=regression'
            }
        }
        
        stage('Analyze Failures') {
            steps {
                script {
                    // 读取失败用例
                    def failures = readJSON file: 'reports/failures.json'
                    
                    if (failures.size() > 0 && failures.size() <= 10) {
                        // 少量失败：自动重跑
                        env.RERUN = 'true'
                        env.RERUN_TESTS = failures.collect { it.name }.join(',')
                    } else if (failures.size() > 10) {
                        // 大量失败：可能是环境问题，告警不重跑
                        env.RERUN = 'false'
                        currentBuild.result = 'FAILURE'
                        error "大量用例失败 (${failures.size()}个)，跳过自动重跑"
                    }
                }
            }
        }
        
        stage('Rerun Failures') {
            when { expression { env.RERUN == 'true' } }
            steps {
                sh 'mvn test -Dtest=${RERUN_TESTS}'
            }
        }
        
        stage('Final Assessment') {
            steps {
                script {
                    // 重跑后仍失败的 -> 真正失败
                    def stillFailing = readJSON file: 'reports/failures-rerun.json'
                    if (stillFailing.size() > 0) {
                        currentBuild.result = 'FAILURE'
                        
                        // 区分：第一次就失败 vs 重跑后失败
                        def consistentlyFailing = stillFailing.findAll { 
                            it.failedBothRuns 
                        }
                        def flaky = stillFailing.findAll { 
                            !it.failedBothRuns 
                        }
                        
                        echo "持续失败: ${consistentlyFailing.size()}个"
                        echo "Flaky: ${flaky.size()}个"
                    } else {
                        currentBuild.result = 'UNSTABLE' // 首次失败但重跑通过
                        echo "所有用例重跑后通过，标记为不稳定"
                    }
                }
            }
        }
    }
}
```

### 案例5：测试数据版本锁定

```groovy
pipeline {
    stages {
        stage('Determine Test Data Version') {
            steps {
                script {
                    // 获取被测应用的版本
                    def appVersion = sh(
                        script: 'curl -s http://app.example.com/version',
                        returnStdout: true
                    ).trim()
                    
                    // 查找匹配的测试数据版本
                    def dataVersion = sh(
                        script: """
                            curl -s "http://testdata-service.example.com/api/versions?appVersion=${appVersion}" | 
                            jq -r '.latest'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (!dataVersion) {
                        // 找不到匹配的测试数据版本，使用最新的
                        dataVersion = 'latest'
                        echo "警告: 找不到与 ${appVersion} 匹配的测试数据版本，使用最新版"
                    }
                    
                    env.TEST_DATA_VERSION = dataVersion
                    echo "测试数据版本: ${TEST_DATA_VERSION}"
                }
            }
        }
        
        stage('Download Test Data') {
            steps {
                sh '''
                    curl -o testdata.zip \
                        "http://testdata-service.example.com/api/download/${TEST_DATA_VERSION}"
                    unzip testdata.zip -d testdata/
                '''
            }
        }
    }
}
```

---

## 本章小结

| 集成维度 | 核心要点 | 关键工具/实践 |
|----------|----------|-------------|
| CI平台 | Jenkins灵活、GitLab CI简洁、GitHub Actions集成紧密 | Jenkinsfile / .gitlab-ci.yml / workflow |
| 触发策略 | Push触发冒烟、PR触发增量、定时触发全量 | cron / webhook / workflow_dispatch |
| 构建联动 | 测试结果决定构建状态、状态决定后续动作 | post条件块 / quality gate |
| Docker化 | 环境一致性、一键启动、依赖隔离 | Docker Compose / Selenium Grid |
| 分布式 | 分片执行、多Agent并行、按浏览器分发 | Jenkins分布式 / Grid / Sharding |

> CI/CD集成是自动化测试从"手动执行"到"自动执行"的质变。它让自动化测试真正融入研发流程，成为代码质量的一道"自动门禁"。一个好的CI/CD集成：提交即测试、失败即阻断、通过即部署。
