# 10_主流测试工具栈大全

## 模块简介

本模块涵盖了软件测试领域最主流的工具栈，从接口测试、自动化、性能、安全、项目管理、DevOps、数据库到日志监控，为测试工程师提供全面的工具使用指南。

共 9 个文件，总计约 9300 行内容，包含详细的安装教程、配置示例、代码片段、对比表格和最佳实践。

---

## 目录结构

```
10_主流测试工具栈大全/
├── README.md                 # 本文件
├── 01_接口测试工具.md         # Postman, curl, Swagger, SoapUI, Apifox
├── 02_自动化工具.md          # Selenium, Playwright, Cypress, Appium, Airtest, Robot Framework
├── 03_性能测试工具.md         # JMeter, Gatling, Locust, k6, wrk/ab
├── 04_安全测试工具.md         # Burp Suite, OWASP ZAP, sqlmap, Nmap, Metasploit
├── 05_项目管理工具.md         # Jira, 禅道, TAPD, TestLink/TestRail, Confluence/语雀/飞书
├── 06_DevOps工具.md          # Jenkins, GitLab CI, GitHub Actions, Docker, Kubernetes, Terraform
├── 07_数据库工具.md           # MySQL Workbench, DBeaver, Navicat, Redis, MongoDB, Flyway/Liquibase
├── 08_日志监控工具.md         # ELK, Prometheus+Grafana, SkyWalking, Sentry, Zabbix
└── 09_工具问题汇总.md         # 各工具FAQ/Troubleshooting (60+问题)
```

## 各文件概览

| 文件 | 行数 | 核心工具数 | 代码示例 | 对比表 | 最佳实践 |
|------|------|-----------|---------|--------|---------|
| 01_接口测试工具 | 809 | 7 | YAML/JSON/Bash/JavaScript | 2 | 5 |
| 02_自动化工具 | 975 | 7 | Python/JavaScript/Robot/Groovy | 3 | 5 |
| 03_性能测试工具 | 976 | 7 | Bash/Java/Scala/Python/JavaScript/Lua | 2 | 5 |
| 04_安全测试工具 | 867 | 10+ | Bash/Python/Groovy/YAML | 3 | 5 |
| 05_项目管理工具 | 856 | 8 | Bash/Python/XML/Groovy/SQL | 2 | 5 |
| 06_DevOps工具 | 1343 | 7 | Groovy/YAML/Dockerfile/HCL | 3 | 5 |
| 07_数据库工具 | 901 | 10+ | SQL/Bash/Python/YAML/XML | 2 | 5 |
| 08_日志监控工具 | 1079 | 10+ | YAML/JSON/Bash/Python | 2 | 5 |
| 09_工具问题汇总 | 1426 | 6大类 | Bash/Groovy/JavaScript/Python/YAML | 6+ | 60+问题 |

---

## 学习路线建议

### 初学者路线
1. **01_接口测试工具** - 掌握 Postman 基础，学会接口调试
2. **05_项目管理工具** - 了解 Jira/禅道，学会 Bug 管理和用例管理
3. **07_数据库工具** - 掌握基本SQL和数据库管理工具
4. **02_自动化工具** - 学习 Selenium/Playwright 自动化基础
5. **09_工具问题汇总** - 遇到问题时快速查阅

### 进阶路线
1. **03_性能测试工具** - JMeter 基础和进阶
2. **06_DevOps工具** - CI/CD流水线和容器化
3. **08_日志监控工具** - 可观测性三大支柱
4. **04_安全测试工具** - Web安全测试基础

### 全面掌握路线
1. 按文件编号顺序系统学习 01-08
2. 每个工具动手实操
3. 结合实战项目练习
4. 参考 09_工具问题汇总 解决问题

---

## 工具矩阵速查

### 按测试类型选择

| 测试类型 | 首选工具 | 备选/进阶 |
|---------|---------|----------|
| API接口测试 | Postman / Apifox | curl + jq / SoapUI |
| Web UI自动化 | Playwright / Selenium | Cypress |
| 移动端自动化 | Appium | Airtest(游戏) |
| 性能测试 | JMeter / k6 | Gatling / Locust |
| 安全测试 | Burp Suite / OWASP ZAP | sqlmap / Metasploit |
| 项目管理 | Jira / TAPD | 禅道 |
| 测试用例管理 | TestRail(付费) / TestLink(免费) | Xray(Jira插件) |
| CI/CD | Jenkins / GitLab CI | GitHub Actions |
| 容器化 | Docker + Compose | Kubernetes |
| 数据库管理 | DBeaver | Navicat(付费) |
| 日志分析 | ELK Stack | Loki(轻量) |
| 指标监控 | Prometheus + Grafana | Zabbix |
| 链路追踪 | SkyWalking(JVM) | Jaeger(多语言) |
| 错误监控 | Sentry | - |

### 按团队规模选择

| 团队规模 | 推荐方案 |
|---------|---------|
| 个人/1-5人 | Postman + Playwright + JMeter + 禅道 + Docker Compose |
| 5-20人 | Apifox + Playwright + k6 + TAPD + GitLab CI + ELK |
| 20-50人 | Postman Enterprise + Selenium Grid + JMeter集群 + Jira + Jenkins |
| 50+人 | 企业私有化部署 + 全工具链 + 自定义平台 |

---

## 内容特点

每个文件均包含：
1. **概念解析** - 清晰的技术概念和架构说明
2. **安装指南** - 多平台(Docker/Linux/macOS/Windows)安装步骤
3. **代码示例** - 可直接运行的代码片段
4. **表格对比** - 方便横向比较选型
5. **最佳实践** - 5条核心实战经验
6. **总结建议** - 针对不同场景的选型建议

---

*最后更新: 2024年6月*
