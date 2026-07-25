     1|# CI/CD 进阶技术文档：从自动化到工程卓越
     2|
     3|## 1. 概述 - CI/CD 进阶的核心理念、与基础 CI/CD 的区别、适用场景
     4|
     5|### 1.1 核心理念
     6|基础 CI/CD 通常关注“能否自动构建和部署”，而进阶 CI/CD 关注“如何高效、安全、可靠地交付价值”。其核心包括：
     7|*   **全链路自动化**：从代码提交到生产环境上线，无人工干预。
     8|*   **可观测性**：不仅知道部署成功与否，还要知道性能影响、错误率变化。
     9|*   **安全性左移 (Shift Left Security)**：在构建阶段即进行漏洞扫描和镜像签名。
    10|*   **基础设施即代码 (IaC)**：CI/CD 流水线本身也是代码，版本可控，可复用。
    11|
    12|### 1.2 与基础 CI/CD 的区别
    13|
    14|| 特性 | 基础 CI/CD | 进阶 CI/CD |
    15|| :--- | :--- | :--- |
    16|| **触发机制** | 简单的 Push 触发 | 事件驱动、定时、依赖触发、人工审批 |
    17|| **并行度** | 串行执行 | DAG 有向无环图并行，矩阵构建 |
    18|| **环境管理** | 静态环境配置 | 动态命名空间， ephemeral environments |
    19|| **部署策略** | 直接覆盖 (Overwrite) | 蓝绿、金丝雀、滚动更新 |
    20|| **安全合规** | 较少或无 | Secret 管理、镜像签名、SBOM、合规检查 |
    21|| **反馈循环** | 构建日志 | 指标监控、告警、用户体验反馈 |
    22|
    23|### 1.3 适用场景
    24|*   **微服务架构**：服务众多，依赖复杂，需要精细化的流水线编排。
    25|*   **高可用要求**：金融、医疗等行业，需要零停机部署和快速回滚。
    26|*   **多环境管理**：Dev, Staging, Prod 多套环境，需保证配置一致性。
    27|*   **大规模并发**：高频次发布（每日多次），需极致优化构建速度和资源利用率。
    28|
    29|```yaml
    30|# 示例：基础 vs 进阶的简单对比概念
    31|# 基础：
    32|steps:
    33|  - build
    34|  - test
    35|  - deploy
    36|
    37|# 进阶：
    38|stages:
    39|  - name: validate
    40|    steps: [lint, security_scan]
    41|  - name: build
    42|    matrix: [node:16, node:18, node:20]
    43|    parallel: true
    44|  - name: test
    45|    depends_on: build
    46|    strategy: canary # 先跑冒烟测试
    47|  - name: deploy
    48|    condition: branch == 'main' && passed(test)
    49|    strategy: blue_green
    50|```
    51|
    52|---
    53|
    54|## 2. 流水线设计模式
    55|
    56|### 2.1 DAG 流水线
    57|DAG（Directed Acyclic Graph）允许定义复杂的依赖关系，而非简单的线性步骤。这是现代 CI/CD 的核心能力。
    58|
    59|### 2.2 并行/串行策略
    60|*   **并行**：适用于独立的任务，如不同语言模块的构建、不同测试套件。
    61|*   **串行**：适用于有依赖关系的任务，如构建完成后才能测试，测试通过才能部署。
    62|
    63|### 2.3 阶段编排与条件执行
    64|使用 `when`、`if` 语句根据分支、变量、前一步结果动态控制流程。
    65|
    66|```yaml
    67|# GitHub Actions 示例：DAG 与条件执行
    68|name: Advanced CI Pipeline
    69|
    70|on:
    71|  push:
    72|    branches: [ main, develop ]
    73|  pull_request:
    74|    branches: [ main ]
    75|
    76|jobs:
    77|  # Job 1: 代码质量检查
    78|  lint:
    79|    runs-on: ubuntu-latest
    80|    steps:
    81|      - uses: actions/checkout@v4
    82|      - run: npm install && npm run lint
    83|
    84|  # Job 2: 单元测试 (并行运行)
    85|  unit-test:
    86|    needs: lint
    87|    runs-on: ubuntu-latest
    88|    strategy:
    89|      matrix:
    90|        node-version: [16, 18, 20]
    91|    steps:
    92|      - uses: actions/checkout@v4
    93|      - name: Setup Node.js ${{ matrix.node-version }}
    94|        uses: actions/setup-node@v4
    95|        with:
    96|          node-version: ${{ matrix.node-version }}
    97|      - run: npm ci
    98|      - run: npm test -- --coverage
    99|
   100|  # Job 3: 集成测试 (仅在 develop 分支触发)
   101|  integration-test:
   102|    needs: unit-test
   103|    runs-on: ubuntu-latest
   104|    if: github.ref == 'refs/heads/develop' || github.event_name == 'pull_request'
   105|    steps:
   106|      - uses: actions/checkout@v4
   107|      - run: docker compose up -d
   108|      - run: npm run test:integration
   109|      - run: docker compose down
   110|
   111|  # Job 4: 构建镜像 (仅在 main 分支且测试通过后)
   112|  build-image:
   113|    needs: [unit-test, integration-test]
   114|    runs-on: ubuntu-latest
   115|    if: github.ref == 'refs/heads/main'
   116|    steps:
   117|      - uses: actions/checkout@v4
   118|      - name: Build Docker Image
   119|        run: docker build -t myapp:${{ github.sha }} .
   120|      - name: Upload Artifact
   121|        uses: actions/upload-artifact@v4
   122|        with:
   123|          name: docker-image
   124|          path: ./myapp.tar
   125|
   126|  # Job 5: 部署到 Staging
   127|  deploy-staging:
   128|    needs: build-image
   129|    runs-on: ubuntu-latest
   130|    environment: staging
   131|    steps:
   132|      - name: Download Image
   133|        uses: actions/download-artifact@v4
   134|        with:
   135|          name: docker-image
   136|      - name: Deploy to Staging
   137|        run: |
   138|          echo "Deploying version ${{ github.sha }} to Staging"
   139|          # kubectl apply -f k8s/staging/
   140|```
   141|
   142|---
   143|
   144|## 3. GitHub Actions 进阶
   145|
   146|### 3.1 自定义 Actions
   147|创建复用的 Action 可以标准化团队工作流。
   148|
   149|**目录结构：**
   150|```text
   151|my-custom-action/
   152|  ├── action.yml
   153|  ├── index.js
   154|  └── package.json
   155|```
   156|
   157|**action.yml:**
   158|```yaml
   159|name: 'Post Processing'
   160|description: 'Run post-processing scripts'
   161|inputs:
   162|  script-path:
   163|    description: 'Path to the script'
   164|    required: true
   165|outputs:
   166|  result:
   167|    description: 'Processing result'
   168|    value: ${{ steps.process.outputs.result }}
   169|runs:
   170|  using: 'node16'
   171|  main: 'index.js'
   172|```
   173|
   174|**index.js:**
   175|```javascript
   176|const core = require('@actions/core');
   177|const fs = require('fs');
   178|
   179|try {
   180|  const scriptPath = core.getInput('script-path');
   181|  // 模拟执行脚本
   182|  console.log(`Executing ${scriptPath}`);
   183|  const result = "success";
   184|  
   185|  // 设置输出
   186|  core.setOutput('result', result);
   187|} catch (error) {
   188|  core.setFailed(error.message);
   189|}
   190|```
   191|
   192|### 3.2 矩阵构建 (Matrix Builds)
   193|用于在不同 OS、Node 版本、数据库版本上并行测试。
   194|
   195|```yaml
   196|jobs:
   197|  test:
   198|    runs-on: ${{ matrix.os }}
   199|    strategy:
   200|      matrix:
   201|        os: [ubuntu-latest, windows-latest, macos-latest]
   202|        node-version: [14, 16, 18]
   203|        # 排除不兼容的组合
   204|        exclude:
   205|          - os: windows-latest
   206|            node-version: 14
   207|```
   208|
   209|### 3.3 Reusable Workflows
   210|将通用流程提取为单独的工作流文件，通过 `workflow_call` 触发。
   211|
   212|**`.github/workflows/build.yml`:**
   213|```yaml
   214|name: Reusable Build Workflow
   215|
   216|on:
   217|  workflow_call:
   218|    inputs:
   219|      project-name:
   220|        required: true
   221|        type: string
   222|    secrets:
   223|      DOCKER_USERNAME:
   224|        required: true
   225|      DOCKER_PASSWORD:
   226|        required: true
   227|
   228|jobs:
   229|  build:
   230|    runs-on: ubuntu-latest
   231|    steps:
   232|      - uses: actions/checkout@v4
   233|      - name: Login to Docker Hub
   234|        uses: docker/login-action@v3
   235|        with:
   236|          username: ${{ secrets.DOCKER_USERNAME }}
   237|          password: ${{ secrets.DOCKER_PASSWORD }}
   238|      - name: Build and Push
   239|        run: |
   240|          docker build -t myregistry/${{ inputs.project-name }}:${{ github.sha }} .
   241|          docker push myregistry/${{ inputs.project-name }}:${{ github.sha }}
   242|```
   243|
   244|**调用方 `.github/workflows/ci.yml`:**
   245|```yaml
   246|name: CI Pipeline
   247|on: push
   248|
   249|jobs:
   250|  call-build:
   251|    uses: ./.github/workflows/build.yml
   252|    with:
   253|      project-name: "my-app"
   254|    secrets:
   255|      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
   256|      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
   257|```
   258|
   259|### 3.4 缓存策略
   260|加速依赖安装是提升 CI 速度的关键。
   261|
   262|```yaml
   263|- name: Cache pip packages
   264|  uses: actions/cache@v3
   265|  with:
   266|    path: ~/.cache/pip
   267|    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
   268|    restore-keys: |
   269|      ${{ runner.os }}-pip-
   270|
   271|- name: Cache Node modules
   272|  uses: actions/cache@v3
   273|  with:
   274|    path: ~/.npm
   275|    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
   276|    restore-keys: |
   277|      ${{ runner.os }}-node-
   278|```
   279|
   280|### 3.5 Self-hosted Runner 管理
   281|对于特殊硬件需求或内部网络访问，使用自托管 Runner。
   282|
   283|**Shell 脚本启动 Runner:**
   284|```bash
   285|#!/bin/bash
   286|# 创建目录并进入
   287|mkdir actions-runner && cd actions-runner
   288|# 下载最新版本的 Runner
   289|curl -o actions-runner-linux-x64-2.312.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.312.0/actions-runner-linux-x64-2.312.0.tar.gz
   290|tar xzf ./actions-runner-linux-x64-2.312.0.tar.gz
   291|
   292|# 配置 Runner
   293|./config.sh --url https://github.com/your-org --token YOUR_PAT_TOKEN
   294|./run.sh
   295|```
   296|
   297|---
   298|
   299|## 4. GitLab CI 进阶
   300|
   301|### 4.1 复杂 DAG
   302|GitLab 14+ 支持原生 DAG 流水线，比传统的 stages 更灵活。
   303|
   304|```yaml
   305|# .gitlab-ci.yml
   306|stages:
   307|  - build
   308|  - test
   309|  - deploy
   310|
   311|variables:
   312|  DOCKER_DRIVER: overlay2
   313|
   314|build_job:
   315|  stage: build
   316|  script:
   317|    - echo "Building..."
   318|  artifacts:
   319|    paths:
   320|      - build/
   321|  rules:
   322|    - if: '$CI_COMMIT_BRANCH == "main"'
   323|
   324|test_job:
   325|  stage: test
   326|  needs: ["build_job"]
   327|  script:
   328|    - echo "Testing..."
   329|  dependencies:
   330|    - build_job
   331|
   332|deploy_job:
   333|  stage: deploy
   334|  needs: ["test_job"]
   335|  script:
   336|    - echo "Deploying..."
   337|  rules:
   338|    - if: '$CI_COMMIT_BRANCH == "main"'
   339|      when: manual
   340|```
   341|
   342|### 4.2 动态子管道 (Dynamic Sub-pipelines)
   343|使用 `include: component` 或脚本生成 YAML 来动态创建子管道，适合模块化项目。
   344|
   345|```yaml
   346|# 主流水线
   347|generate_jobs:
   348|  stage: test
   349|  script:
   350|    - python scripts/generate_ci_yaml.py > .gitlab-ci-generated.yml
   351|  artifacts:
   352|    reports:
   353|      dotenv: .gitlab-ci-generated.env
   354|  rules:
   355|    - if: '$CI_PIPELINE_SOURCE == "pipeline"'
   356|
   357|include:
   358|  - local: '.gitlab-ci-generated.yml'
   359|```
   360|
   361|### 4.3 Runner 分级管理
   362|通过标签 (Tags) 和调度器 (Scheduler) 管理不同类型的 Runner。
   363|
   364|```yaml
   365|# 只在特定 Runner 上运行的任务
   366|deploy-prod:
   367|  stage: deploy
   368|  tags:
   369|    - production-runner
   370|    - linux
   371|  script:
   372|    - ./deploy-to-prod.sh
   373|```
   374|
   375|### 4.4 环境变量与 Secret
   376|利用 GitLab CI/CD Settings 中的 Variables 和 Masked/Protected 选项保护敏感信息。
   377|
   378|```yaml
   379|variables:
   380|  DB_PASSWORD: ${{ secrets.DB_PASSWORD }} # 从 CI/CD 变量读取
   381|
   382|# 在脚本中使用时，GitLab 会自动屏蔽日志输出
   383|script:
   384|  - echo "Connecting to DB with password: $DB_PASSWORD" # 日志中显示为 [MASKED]
   385|```
   386|
   387|### 4.5 Artifact 与 Cache 最佳实践
   388|*   **Artifact**：用于传递文件给后续 Job，注意清理过期数据。
   389|*   **Cache**：用于加速依赖下载，基于 Key 哈希。
   390|
   391|```yaml
   392|cache:
   393|  key: ${CI_COMMIT_REF_SLUG}
   394|  paths:
   395|    - vendor/
   396|    - node_modules/
   397|  policy: pull-push # 默认拉取，如果有新依赖则推送
   398|
   399|artifacts:
   400|  expire_in: 1 week
   401|  paths:
   402|    - coverage/lcov.info
   403|```
   404|
   405|---
   406|
   407|## 5. Jenkins Pipeline as Code
   408|
   409|### 5.1 Jenkinsfile 高级用法
   410|使用 Declarative Pipeline 结合 Scripted Pipeline 处理复杂逻辑。
   411|
   412|```groovy
   413|pipeline {
   414|    agent any
   415|
   416|    environment {
   417|        DOCKER_CREDENTIALS = credentials('docker-hub-creds')
   418|        KUBECONFIG = credentials('kube-config')
   419|    }
   420|
   421|    stages {
   422|        stage('Checkout') {
   423|            steps {
   424|                checkout scm
   425|            }
   426|        }
   427|
   428|        stage('Build & Test') {
   429|            parallel {
   430|                stage('Unit Tests') {
   431|                    steps {
   432|                        sh 'npm test'
   433|                    }
   434|                }
   435|                stage('Linting') {
   436|                    steps {
   437|                        sh 'npm run lint'
   438|                    }
   439|                }
   440|            }
   441|        }
   442|
   443|        stage('Deploy') {
   444|            when {
   445|                branch 'main'
   446|            }
   447|            steps {
   448|                script {
   449|                    def appImage = sh(script: 'echo $(date +%s)', returnStdout: true).trim()
   450|                    sh "docker tag myapp:latest myrepo/myapp:${appImage}"
   451|                    sh "docker push myrepo/myapp:${appImage}"
   452|                    
   453|                    withKubeConfig([credentialsId: 'kube-config']) {
   454|                        sh """
   455|                            kubectl set image deployment/myapp myapp=myrepo/myapp:${appImage}
   456|                        """
   457|                    }
   458|                }
   459|            }
   460|        }
   461|    }
   462|
   463|    post {
   464|        always {
   465|            cleanWs()
   466|        }
   467|        failure {
   468|            slackSend channel: '#ci-failures', message: "Build failed for ${env.JOB_NAME}"
   469|        }
   470|    }
   471|}
   472|```
   473|
   474|### 5.2 Shared Libraries
   475|将通用逻辑提取为共享库，减少代码重复。
   476|
   477|**`vars/deployToStaging.groovy`:**
   478|```groovy
   479|def call(String appName, String version) {
   480|    echo "Deploying ${appName} version ${version} to Staging"
   481|    sh "kubectl set image deployment/${appName} ${appName}=myrepo/${appName}:${version}"
   482|    sh "kubectl rollout status deployment/${appName}"
   483|}
   484|```
   485|
   486|**Jenkinsfile 中使用：**
   487|```groovy
   488|@Library('my-shared-lib') _
   489|
   490|pipeline {
   491|    stages {
   492|        stage('Deploy') {
   493|            steps {
   494|                deployToStaging('my-app', env.BUILD_NUMBER)
   495|            }
   496|        }
   497|    }
   498|}
   499|```
   500|
   501|