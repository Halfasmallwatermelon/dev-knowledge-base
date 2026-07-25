# CI/CD 进阶技术文档：从自动化到工程卓越

## 1. 概述 - CI/CD 进阶的核心理念、与基础 CI/CD 的区别、适用场景

### 1.1 核心理念
基础 CI/CD 通常关注“能否自动构建和部署”，而进阶 CI/CD 关注“如何高效、安全、可靠地交付价值”。其核心包括：
*   **全链路自动化**：从代码提交到生产环境上线，无人工干预。
*   **可观测性**：不仅知道部署成功与否，还要知道性能影响、错误率变化。
*   **安全性左移 (Shift Left Security)**：在构建阶段即进行漏洞扫描和镜像签名。
*   **基础设施即代码 (IaC)**：CI/CD 流水线本身也是代码，版本可控，可复用。

### 1.2 与基础 CI/CD 的区别

| 特性 | 基础 CI/CD | 进阶 CI/CD |
| :--- | :--- | :--- |
| **触发机制** | 简单的 Push 触发 | 事件驱动、定时、依赖触发、人工审批 |
| **并行度** | 串行执行 | DAG 有向无环图并行，矩阵构建 |
| **环境管理** | 静态环境配置 | 动态命名空间， ephemeral environments |
| **部署策略** | 直接覆盖 (Overwrite) | 蓝绿、金丝雀、滚动更新 |
| **安全合规** | 较少或无 | Secret 管理、镜像签名、SBOM、合规检查 |
| **反馈循环** | 构建日志 | 指标监控、告警、用户体验反馈 |

### 1.3 适用场景
*   **微服务架构**：服务众多，依赖复杂，需要精细化的流水线编排。
*   **高可用要求**：金融、医疗等行业，需要零停机部署和快速回滚。
*   **多环境管理**：Dev, Staging, Prod 多套环境，需保证配置一致性。
*   **大规模并发**：高频次发布（每日多次），需极致优化构建速度和资源利用率。

```yaml
# 示例：基础 vs 进阶的简单对比概念
# 基础：
steps:
  - build
  - test
  - deploy

# 进阶：
stages:
  - name: validate
    steps: [lint, security_scan]
  - name: build
    matrix: [node:16, node:18, node:20]
    parallel: true
  - name: test
    depends_on: build
    strategy: canary # 先跑冒烟测试
  - name: deploy
    condition: branch == 'main' && passed(test)
    strategy: blue_green
```

---

## 2. 流水线设计模式

### 2.1 DAG 流水线
DAG（Directed Acyclic Graph）允许定义复杂的依赖关系，而非简单的线性步骤。这是现代 CI/CD 的核心能力。

### 2.2 并行/串行策略
*   **并行**：适用于独立的任务，如不同语言模块的构建、不同测试套件。
*   **串行**：适用于有依赖关系的任务，如构建完成后才能测试，测试通过才能部署。

### 2.3 阶段编排与条件执行
使用 `when`、`if` 语句根据分支、变量、前一步结果动态控制流程。

```yaml
# GitHub Actions 示例：DAG 与条件执行
name: Advanced CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Job 1: 代码质量检查
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install && npm run lint

  # Job 2: 单元测试 (并行运行)
  unit-test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test -- --coverage

  # Job 3: 集成测试 (仅在 develop 分支触发)
  integration-test:
    needs: unit-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop' || github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d
      - run: npm run test:integration
      - run: docker compose down

  # Job 4: 构建镜像 (仅在 main 分支且测试通过后)
  build-image:
    needs: [unit-test, integration-test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker Image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: ./myapp.tar

  # Job 5: 部署到 Staging
  deploy-staging:
    needs: build-image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Download Image
        uses: actions/download-artifact@v4
        with:
          name: docker-image
      - name: Deploy to Staging
        run: |
          echo "Deploying version ${{ github.sha }} to Staging"
          # kubectl apply -f k8s/staging/
```

---

## 3. GitHub Actions 进阶

### 3.1 自定义 Actions
创建复用的 Action 可以标准化团队工作流。

**目录结构：**
```text
my-custom-action/
  ├── action.yml
  ├── index.js
  └── package.json
```

**action.yml:**
```yaml
name: 'Post Processing'
description: 'Run post-processing scripts'
inputs:
  script-path:
    description: 'Path to the script'
    required: true
outputs:
  result:
    description: 'Processing result'
    value: ${{ steps.process.outputs.result }}
runs:
  using: 'node16'
  main: 'index.js'
```

**index.js:**
```javascript
const core = require('@actions/core');
const fs = require('fs');

try {
  const scriptPath = core.getInput('script-path');
  // 模拟执行脚本
  console.log(`Executing ${scriptPath}`);
  const result = "success";
  
  // 设置输出
  core.setOutput('result', result);
} catch (error) {
  core.setFailed(error.message);
}
```

### 3.2 矩阵构建 (Matrix Builds)
用于在不同 OS、Node 版本、数据库版本上并行测试。

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [14, 16, 18]
        # 排除不兼容的组合
        exclude:
          - os: windows-latest
            node-version: 14
```

### 3.3 Reusable Workflows
将通用流程提取为单独的工作流文件，通过 `workflow_call` 触发。

**`.github/workflows/build.yml`:**
```yaml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      project-name:
        required: true
        type: string
    secrets:
      DOCKER_USERNAME:
        required: true
      DOCKER_PASSWORD:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build and Push
        run: |
          docker build -t myregistry/${{ inputs.project-name }}:${{ github.sha }} .
          docker push myregistry/${{ inputs.project-name }}:${{ github.sha }}
```

**调用方 `.github/workflows/ci.yml`:**
```yaml
name: CI Pipeline
on: push

jobs:
  call-build:
    uses: ./.github/workflows/build.yml
    with:
      project-name: "my-app"
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 3.4 缓存策略
加速依赖安装是提升 CI 速度的关键。

```yaml
- name: Cache pip packages
  uses: actions/cache@v3
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-

- name: Cache Node modules
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### 3.5 Self-hosted Runner 管理
对于特殊硬件需求或内部网络访问，使用自托管 Runner。

**Shell 脚本启动 Runner:**
```bash
#!/bin/bash
# 创建目录并进入
mkdir actions-runner && cd actions-runner
# 下载最新版本的 Runner
curl -o actions-runner-linux-x64-2.312.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.312.0/actions-runner-linux-x64-2.312.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.312.0.tar.gz

# 配置 Runner
./config.sh --url https://github.com/your-org --token YOUR_PAT_TOKEN
./run.sh
```

---

## 4. GitLab CI 进阶

### 4.1 复杂 DAG
GitLab 14+ 支持原生 DAG 流水线，比传统的 stages 更灵活。

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2

build_job:
  stage: build
  script:
    - echo "Building..."
  artifacts:
    paths:
      - build/
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

test_job:
  stage: test
  needs: ["build_job"]
  script:
    - echo "Testing..."
  dependencies:
    - build_job

deploy_job:
  stage: deploy
  needs: ["test_job"]
  script:
    - echo "Deploying..."
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

### 4.2 动态子管道 (Dynamic Sub-pipelines)
使用 `include: component` 或脚本生成 YAML 来动态创建子管道，适合模块化项目。

```yaml
# 主流水线
generate_jobs:
  stage: test
  script:
    - python scripts/generate_ci_yaml.py > .gitlab-ci-generated.yml
  artifacts:
    reports:
      dotenv: .gitlab-ci-generated.env
  rules:
    - if: '$CI_PIPELINE_SOURCE == "pipeline"'

include:
  - local: '.gitlab-ci-generated.yml'
```

### 4.3 Runner 分级管理
通过标签 (Tags) 和调度器 (Scheduler) 管理不同类型的 Runner。

```yaml
# 只在特定 Runner 上运行的任务
deploy-prod:
  stage: deploy
  tags:
    - production-runner
    - linux
  script:
    - ./deploy-to-prod.sh
```

### 4.4 环境变量与 Secret
利用 GitLab CI/CD Settings 中的 Variables 和 Masked/Protected 选项保护敏感信息。

```yaml
variables:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }} # 从 CI/CD 变量读取

# 在脚本中使用时，GitLab 会自动屏蔽日志输出
script:
  - echo "Connecting to DB with password: $DB_PASSWORD" # 日志中显示为 [MASKED]
```

### 4.5 Artifact 与 Cache 最佳实践
*   **Artifact**：用于传递文件给后续 Job，注意清理过期数据。
*   **Cache**：用于加速依赖下载，基于 Key 哈希。

```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - node_modules/
  policy: pull-push # 默认拉取，如果有新依赖则推送

artifacts:
  expire_in: 1 week
  paths:
    - coverage/lcov.info
```

---

## 5. Jenkins Pipeline as Code

### 5.1 Jenkinsfile 高级用法
使用 Declarative Pipeline 结合 Scripted Pipeline 处理复杂逻辑。

```groovy
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('docker-hub-creds')
        KUBECONFIG = credentials('kube-config')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Linting') {
                    steps {
                        sh 'npm run lint'
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def appImage = sh(script: 'echo $(date +%s)', returnStdout: true).trim()
                    sh "docker tag myapp:latest myrepo/myapp:${appImage}"
                    sh "docker push myrepo/myapp:${appImage}"
                    
                    withKubeConfig([credentialsId: 'kube-config']) {
                        sh """
                            kubectl set image deployment/myapp myapp=myrepo/myapp:${appImage}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            slackSend channel: '#ci-failures', message: "Build failed for ${env.JOB_NAME}"
        }
    }
}
```

### 5.2 Shared Libraries
将通用逻辑提取为共享库，减少代码重复。

**`vars/deployToStaging.groovy`:**
```groovy
def call(String appName, String version) {
    echo "Deploying ${appName} version ${version} to Staging"
    sh "kubectl set image deployment/${appName} ${appName}=myrepo/${appName}:${version}"
    sh "kubectl rollout status deployment/${appName}"
}
```

**Jenkinsfile 中使用：**
```groovy
@Library('my-shared-lib') _

pipeline {
    stages {
        stage('Deploy') {
            steps {
                deployToStaging('my-app', env.BUILD_NUMBER)
            }
        }
    }
}
```

### 5.3 Pipeline 难点与排错
*   **隐式上下文丢失**：确保变量在正确的 Stage/Step 作用域内。
*   **超时问题**：调整 `timeout` 指令。
*   **日志分析**：使用 Jenkins Log Parser 插件分析构建日志中的错误模式。

---

## 6. GitOps 实践

### 6.1 ArgoCD 高级配置
ArgoCD 通过声明式方式同步 Kubernetes 资源。

**Application 资源定义：**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: HEAD
    path: apps/my-web-app
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 6.2 Flux CD
Flux 是另一种流行的 GitOps 工具，基于 Kubernetes Operator 模式。

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
patchesStrategicMerge:
  - patch.yaml
```

### 6.3 多集群管理
使用 ArgoCD App of Apps 模式管理多个集群。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-management
spec:
  project: default
  source:
    repoURL: https://github.com/org/multi-cluster-config.git
    path: clusters
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated: {}
```

### 6.4 漂移检测与自愈
ArgoCD 默认开启 `selfHeal`，当检测到集群状态与 Git 仓库不一致时，自动修正。

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
```

---

## 7. 部署策略详解

### 7.1 蓝绿部署 (Blue-Green)
同时运行两个相同的环境，一个在线，一个离线。切换流量瞬间完成。

```yaml
# Kubernetes Service 切换示例
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
    version: green # 切换此标签以切换环境
  ports:
    - port: 80
      targetPort: 8080
```

### 7.2 金丝雀发布 (Canary)
逐步将少量流量引导至新版本，监控指标后再决定是否全量发布。

```yaml
# Argo Rollouts 示例
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: canary-demo
spec:
  replicas: 5
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: canary-demo
  template:
    metadata:
      labels:
        app: canary-demo
    spec:
      containers:
      - name: web-server
        image: nginx:1.19
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 10s}
      - setWeight: 40
      - pause: {duration: 10s}
      - setWeight: 60
      - pause: {duration: 10s}
      - setWeight: 80
      - pause: {duration: 10s}
```

### 7.3 滚动更新 (Rolling Update)
逐个替换 Pod，保持服务可用性。Kubernetes 默认策略。

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

### 7.4 A/B 测试与 Feature Flag
结合负载均衡器和功能标志，针对不同用户群体展示不同功能。

```python
# Python 示例：Feature Flag 判断
def get_version(request):
    user_id = request.user.id
    if user_id % 10 == 0:  # 10% 的用户使用新版本
        return "v2"
    return "v1"
```

---

## 8. CI/CD 安全实践

### 8.1 Secrets 管理
严禁在代码中硬编码密码。使用 HashiCorp Vault 或 SOPS 加密配置文件。

**SOPS 加密示例：**
```bash
# 加密文件
sops --encrypt --encrypted-regex '^(data|stringData)$' config.yaml > config.enc.yaml

# 解密并使用
sops --decrypt config.enc.yaml > config.dec.yaml
```

### 8.2 镜像签名 (Cosign/Notary)
确保容器镜像未被篡改。

```bash
# 使用 Cosign 签名镜像
cosign sign --key cosign.key myregistry/myapp:latest

# 验证签名
cosign verify --key cosign.pub myregistry/myapp:latest
```

### 8.3 依赖扫描
集成 Dependabot 或 Snyk 在 PR 阶段发现漏洞。

```yaml
# GitHub Dependabot 配置
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "daily"
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "daily"
```

### 8.4 SBOM 生成
软件物料清单 (SBOM) 提供组件透明度。

```bash
# 使用 Syft 生成 SBOM
syft myregistry/myapp:latest -o cyclonedx-json > sbom.json
```

---

## 9. 可观测性与监控

### 9.1 Pipeline 指标收集
使用 Prometheus Exporter 暴露 CI/CD 指标。

```python
from prometheus_client import Counter, Histogram

BUILD_COUNT = Counter('ci_build_total', 'Total builds')
BUILD_DURATION = Histogram('ci_build_duration_seconds', 'Build duration')

def track_build():
    BUILD_COUNT.inc()
    with BUILD_DURATION.time():
        run_pipeline()
```

### 9.2 DORA 指标
监控四个关键指标：
1.  **部署频率** (Deployment Frequency)
2.  **变更前置时间** (Lead Time for Changes)
3.  **服务恢复时间** (Time to Restore Service)
4.  **变更失败率** (Change Failure Rate)

### 9.3 Grafana Dashboard
创建自定义 Dashboard 可视化上述指标。

```json
{
  "dashboard": {
    "title": "CI/CD Performance",
    "panels": [
      {
        "title": "Build Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(ci_build_success_total[5m])) / sum(rate(ci_build_total[5m]))"
          }
        ]
      }
    ]
  }
}
```

### 9.4 告警配置
当构建失败率超过阈值时发送通知。

```yaml
# Prometheus Alertmanager 配置
alerts:
  - alert: HighBuildFailureRate
    expr: rate(ci_build_failure_total[5m]) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High build failure rate detected"
```

---

## 10. 性能优化

### 10.1 缓存策略
除了依赖缓存，还可以缓存构建产物。

```yaml
# GitHub Actions 缓存构建产物
- name: Cache build artifacts
  uses: actions/cache@v3
  with:
    path: |
      build/
      dist/
    key: ${{ runner.os }}-build-${{ hashFiles('**/*.ts') }}
```

### 10.2 增量构建
只重新构建受影响的模块。

```bash
# 使用 Bazel 或 Nx 进行增量构建
nx affected --target=build --base=main~1 --head=HEAD
```

### 10.3 并行测试
将测试套件拆分为多个部分并行执行。

```yaml
# Jest 并行测试
jest --maxWorkers=4
```

### 10.4 构建速度分析
使用 `time` 命令或专用工具分析瓶颈。

```bash
time make build
# 输出: real 0m12.345s, user 0m10.123s, sys 0m1.234s
```

### 10.5 资源节约
使用 Spot Instances 或自动缩放的 Runner 集群降低成本。

---

## 11. 多云与混合云部署

### 11.1 AWS CodePipeline
AWS 原生的 CI/CD 服务，深度集成 EC2, ECS, Lambda。

```yaml
# AWS SAM 模板中的 Pipeline
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt PipelineRole.Arn
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeCommit
                Version: '1'
              OutputArtifacts:
                - Name: SourceOutput
              Configuration:
                BranchName: main
                RepositoryName: my-repo
```

### 11.2 Azure DevOps
强大的多平台支持，集成 Kubernetes 和 Terraform。

```yaml
# Azure Pipelines YAML
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Docker@2
  inputs:
    containerRegistry: 'MyACR'
    repository: 'myapp'
    command: 'buildAndPush'
    Dockerfile: '**/Dockerfile'
```

### 11.3 GCP Cloud Build
极速构建，内置容器化支持。

```yaml
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA', '.']
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA']
images:
- 'gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA'
```

### 11.4 跨云 CI/CD 统一
使用抽象层工具如 Spinnaker 或自定义脚本管理多云部署。

---

## 12. 最佳实践与常见问题

### 12.1 团队协作
*   **代码审查**：所有 Pipeline 变更必须经过 PR 审查。
*   **文档**：维护清晰的 README 和 Wiki，解释 Pipeline 逻辑。

### 12.2 回滚策略
始终保留上一个稳定版本的镜像或配置，以便快速回滚。

```bash
# 快速回滚到上一个版本
kubectl rollout undo deployment/myapp
```

### 12.3 环境一致性
使用相同的 Docker 镜像和 Helm Chart 版本在所有环境中，仅通过 ConfigMap/Secret 区分配置。

### 12.4 常见错误与解决方案

| 错误 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| Build Timeout | 依赖下载慢或测试过多 | 启用缓存，并行化测试，增加超时时间 |
| Out of Memory | 构建资源不足 | 增加 Runner 内存，优化构建脚本 |
| Deployment Fail | 配置错误或镜像不存在 | 检查 K8s 事件日志，验证镜像标签 |
| Secret Leak | 日志中打印了敏感信息 | 使用 `mask` 功能，避免 echo 敏感变量 |

```python
# 安全处理 Secret 的 Python 示例
import os
import boto3

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return response['SecretString']

# 永远不要直接 print(secret)
# secret = get_secret('db_password')
# print(f"Using password: {secret}") # BAD

secret = get_secret('db_password')
# 使用 secret 进行连接，但不打印
connect_to_db(password=secret)
```

---

本文档涵盖了 CI/CD 进阶实践的关键领域。实施这些最佳实践需要结合团队的具体需求和基础设施状况进行微调。持续学习和迭代是保持 CI/CD 系统高效和安全的关键。