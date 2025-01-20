---
title: 使用 GitLab CICD 实现 Spring Boot 应用的自动化部署
categories: [Java]
tags: [SpringBoot,GitLab]
date: 2025-01-20
media_subpath: '/posts/2025/01/20'
---
## 1. 配置GitLab Runner： 

#### GitLab Runner 介绍

**GitLab Runner** 是一个用于执行 GitLab CI/CD 流程中的任务的开源工具。作为一个工人的角色，来完成GitLab需要执行的任务。

它主要负责执行 `.gitlab-ci.yml` 文件中定义的构建、部署等任务。

**GitLab Runner** 可以运行在多种环境中，如本地机器、虚拟机、物理服务器、以及 **Docker 容器** 中，执行任务并将结果反馈给 GitLab 系统。

#### 2.安装和配置 GitLab Runner

#### 1. **在本地或服务器上安装 GitLab Runner**

**安装 GitLab Runner** 的步骤根据操作系统有所不同。以下是 **Linux** 系统上的安装步骤。

1. **安装依赖工具：**
   ```sh
   sudo apt-get install -y curl
   ```
2. **下载并安装 GitLab Runner：**
   ```sh
   curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
   sudo chmod +x /usr/local/bin/gitlab-runner
   ```
3. **安装并启动 GitLab Runner 服务：**
   ```sh
   sudo gitlab-runner install
   sudo gitlab-runner start
   ```

#### 2. **注册 GitLab Runner**

安装完成后，**GitLab Runner** 需要与 GitLab 项目进行连接。通过以下命令来注册 Runner：

```sh
sudo gitlab-runner register
```

此时，系统会要求输入以下信息：

- **GitLab 实例 URL**：例如 `https://gitlab.com`
- **注册令牌**：在 GitLab 项目的 **Settings > CI / CD** 页面可以找到。

接下来，选择 **GitLab Runner** 的执行环境（例如 `docker` 或 `shell`），并根据需要配置执行参数。

#### 3. **检查 GitLab Runner 状态**

完成安装和注册后，可以通过以下命令检查 GitLab Runner 是否正常运行：

```sh
sudo gitlab-runner status
```

该命令会返回 Runner 的当前状态，确保它已经正常连接到 GitLab 并准备执行任务。

#### 2. **离线安装 GitLab Runner**

在没有外部网络连接的环境中，你可以通过以下步骤进行离线安装。离线安装的关键是下载 GitLab Runner 安装包及其依赖包，然后手动将这些文件传输到目标服务器。

#### **离线安装步骤**

1. **在有网络的机器上下载 GitLab Runner 安装包：**
   
   访问 GitLab 官方下载页面，或者直接使用 `curl` 命令下载相应的 GitLab Runner 二进制文件。你可以在有互联网连接的机器上下载：
   ```sh
   curl -L --output gitlab-runner-linux-amd64 https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
   ```
   
   或者前往 [GitLab Runner下载页面](https://gitlab-runner-downloads.s3.amazonaws.com/) 查找适合你操作系统的安装包。
2. **将安装包传输到目标服务器：**
使用 `scp`、`rsync`、或其他文件传输工具，将下载的 `gitlab-runner-linux-amd64` 文件传输到目标服务器上：
   ```sh
   scp gitlab-runner-linux-amd64 user@target-server:/usr/local/bin/gitlab-runner
   ```
3. **手动安装 GitLab Runner：**
在目标服务器上，给 GitLab Runner 二进制文件赋予执行权限：
   ```sh
   sudo chmod +x /usr/local/bin/gitlab-runner
   ```
4. **安装并启动 GitLab Runner 服务：**
将 GitLab Runner 安装为服务并启动：
   ```sh
   sudo gitlab-runner install
   sudo gitlab-runner start
   ```
5. **注册 GitLab Runner：**
你需要获取 GitLab 项目的注册令牌，在 GitLab 项目中的 **Settings > CI / CD** 页面找到这个令牌，并将 GitLab Runner 注册到该项目：
   ```sh
   sudo gitlab-runner register
   ```
   
   按照提示，输入 GitLab 实例的 URL 和注册令牌。之后，选择 Runner 的执行方式（如 `docker` 或 `shell`）并配置环境。
   
   修改gitlab-runner的配置文件

#### 为什么 GitLab 推荐将 Runner 安装在本机，而不是 Docker 环境中？

尽管 **Docker 环境** 提供了高度的隔离性和可移植性，但 **GitLab** 更推荐将 **GitLab Runner** 安装在本机而非 Docker 环境中的原因包括：

1. **简化配置和管理**：
   - 在本机上安装 **GitLab Runner**，配置过程通常更简单，因为本机直接与外部服务和资源集成。特别是在需要与外部工具、数据库等进行交互时，本机安装提供了更高的灵活性。
2. **避免容器隔离带来的复杂性**：
   - 在 Docker 环境中运行 **GitLab Runner** 时，可能会遇到资源限制或容器之间的网络问题，导致构建和测试任务失败。本地安装可以避免这些可能的配置和隔离问题，提供更高的稳定性。

## 2. **理解 GitLab CI/CD 配置文件 `.gitlab-ci.yml`**

- .gitlab-ci.yml 文件概述
  
  `.gitlab-ci.yml` 文件是 GitLab CI/CD 流程的核心配置文件，用于定义项目的自动化流程。它以 **YAML** 格式编写，位于 GitLab 项目的根目录中，负责描述整个 **持续集成（CI）** 和 **持续部署（CD）** 的过程。这个文件告知 GitLab Runner 如何执行项目中的构建、测试、部署等任务。
  
  ### **基本结构和用途**
  
  `.gitlab-ci.yml` 文件的结构由多个组成部分构成，通常包括 **stages**、**jobs** 和 **scripts** 等。每个部分有其特定的作用，确保 CI/CD 流程的顺利执行。
  
  #### **1. stages（阶段）**
  
  **stages** 定义了整个 CI/CD 流程的各个阶段。每个阶段会包含一个或多个任务（jobs）。在 `.gitlab-ci.yml` 文件中，你可以定义一个或多个阶段，GitLab 会按顺序执行这些阶段。
  
  **示例：**
  ```yaml
  stages:
    - build
    - test
    - deploy
  ```
  
  在这个示例中，CI/CD 流程被分为三个阶段：`build`、`test` 和 `deploy`。
  
  #### **2. jobs（任务）**
  
  **jobs** 是 `.gitlab-ci.yml` 文件中的关键部分。每个 job 都是一个具体的任务，负责执行一个特定的操作，如编译、运行测试或部署。每个 job 必须包含执行的命令和关联的阶段。
  
  **基本格式：**
  ```yaml
  job_name:
    stage: stage_name
    script:
      - command1
      - command2
  ```
  
  **示例：**
  ```yaml
  build_job:
    stage: build
    script:
      - echo "Building the project"
      - make build
  ```
  
  这里的 `build_job` 是一个任务，它属于 `build` 阶段，执行的命令包括打印一条消息和运行 `make build`。
  
  #### **3. script（脚本）**
  
  **script** 定义了执行 job 时需要运行的命令。这些命令可以是任何 Shell 命令（如构建工具、测试命令等），也可以是调用其他程序或脚本的命令。
  
  **示例：**
  ```yaml
  test_job:
    stage: test
    script:
      - echo "Running tests"
      - pytest tests/
  ```
  
  在 `test_job` 中，`script` 指定了执行测试的命令，使用 `pytest` 来运行项目中的测试。
  
  #### **4. 其他常用配置项**
  
  除了 `stages`、`jobs` 和 `script`，`.gitlab-ci.yml` 文件还支持许多其他的配置项：
  - **only/except**：限制 job 执行的条件，如仅在特定分支上运行。
  - **before_script/after_script**：分别在每个 job 执行之前和之后运行的命令。
  - **artifacts**：指定 job 执行后需要保存或上传的文件。
  - **cache**：缓存某些文件以加速后续作业的执行。
  - **variables**：定义环境变量，可以在 job 的脚本中使用。
  
  **示例：**
  ```yaml
  job_with_cache:
    stage: build
    script:
      - make install
    cache:
      paths:
        - node_modules/
  ```
  
  此配置在 `job_with_cache` 中，缓存了 `node_modules/` 目录，以便后续任务可以重用这些文件，从而加快构建过程。
  
  #### **5. 触发器和依赖关系**
  - **dependencies**：定义 job 之间的依赖关系，确保某些 job 在其他 job 完成后才能运行。
  - **trigger**：用于触发外部的 GitLab CI/CD 流程或其他项目的 Pipeline。
  
  **示例：**
  ```yaml
  deploy_job:
    stage: deploy
    script:
      - echo "Deploying application"
    dependencies:
      - build_job
  ```
  
  `deploy_job` 将依赖于 `build_job`，即只有 `build_job` 完成后才会执行 `deploy_job`。
  
  ---
- **创建 `.gitlab-ci.yml` 文件：**  
展示如何为 Spring Boot 应用创建一个简单的 CI 配置文件。可以包括构建、测试、打包的步骤。
- **构建阶段：**  
使用 Maven 或 Gradle 进行 Spring Boot 应用的构建，例如：
  ```yaml
  build:
    stage: build
    script:
      - mvn clean install
  ```
- **测试阶段：**  
配置自动化单元测试或集成测试，确保每次代码变更不会引入错误。
  ```yaml
  test:
    stage: test
    script:
      - mvn test
  ```
- **打包阶段：**  
生成可部署的 JAR 或 WAR 文件，通常是执行 `mvn package` 或 `gradle build`。
- **部署阶段：**  
配置部署到测试环境或生产环境的步骤。比如，使用 Docker 部署、远程服务器部署等。

## 3. GitLab CI 配置 (`.gitlab-ci.yml`)

我们在 `gitlab-ci.yml` 文件中定义了构建、打包、部署等步骤。使用 Maven Docker 镜像来构建 JAR 包，并指定 Maven 仓库路径来避免每次构建时下载依赖。

#### 配置文件示例：

```yaml
image: maven:3.8.8-eclipse-temurin-8-focal  # 使用 Maven Docker 镜像

stages:
  - clean
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=/root/.m2/repository"  # 使用挂载的 Maven 仓库
  DEPLOY_JAR: "project.jar"
  ORIGIN_JAR: "project-web.jar"
  ORIGIN_PATH_JAR: "project-web/target/project-web.jar"

# 仅允许手动触发流水线
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "web"'
      when: always
    - when: never

clean:
  stage: clean
  tags:
    - build  # 使用构建 Runner
  script:
    - mvn clean

package:
  stage: package
  tags:
    - build
  before_script:
    # 配置 Maven settings.xml
    - echo "$MAVEN_SETTINGS" > /root/.m2/settings.xml
    - echo "$GLOBAL_MAVEN_SETTINGS" > /usr/share/maven/conf/settings.xml
  script:
    - pwd
    - mvn package
    - echo "Build completed!"
  artifacts:
    paths:
      - $ORIGIN_PATH_JAR
    expire_in: 2 hours
```

#### 配置说明：

- **image**：使用 `maven:3.8.8-eclipse-temurin-8-focal` 镜像来构建 JAR 包。
- **stages**：定义了三个阶段：`clean`、`package` 和 `deploy`。
- **variables**：配置了 Maven 的本地仓库路径和一些部署相关的变量。
- **workflow**：限制了流水线的触发条件，只允许手动触发。
- **clean** 阶段：执行 `mvn clean` 命令清理构建。
- **package** 阶段：执行 Maven 构建，打包 JAR 文件，并指定构建产物保存路径。

#### 2. GitLab Runner 配置

GitLab CI 使用 GitLab Runner 来执行构建和部署任务。在本项目中，我们配置了两个 GitLab Runner：

- **Maven Runner**（Docker Executor）：用于 Maven 构建任务。
- **Shell Runner**（Shell Executor）：用于部署任务。

#### Maven Runner 配置：

```ini
[[runners]]
name = "docker"
url = "http://<gitlab_runner_ip>:8083"  # 请替换为实际 GitLab Runner IP
id = 10
token = "<maven_runner_token>"  # 请替换为实际 token
token_obtained_at = 2025-01-14T02:53:00Z
token_expires_at = 0001-01-01T00:00:00Z
executor = "docker"
clone_url = "http://<gitlab_runner_ip>:8083"

[runners.docker]
pull_policy = "if-not-present"
tls_verify = false
image = "maven:3.8.8-eclipse-temurin-8-focal"
privileged = false
disable_entrypoint_overwrite = false
oom_kill_disable = false
disable_cache = false
volumes = ["/home/gitlab-runner/.m2/repository:/root/.m2/repository", "/home/gitlab-runner/docker/builds:/builds"]
shm_size = 0
network_mtu = 0

```

#### 配置说明：

- **executor**：使用 Docker Executor 运行构建任务。
- **image**：指定构建时使用的 Docker 镜像 `maven:3.8.8-eclipse-temurin-8-focal`。
- **volumes**：将 Maven 仓库和构建目录挂载到容器内，避免每次构建都重新下载依赖。
- **shm_size**：共享内存大小设置为 0，避免在构建过程中占用过多内存。

#### Shell Runner 配置（部署任务）：

```ini
[[runners]]
name = "maven"
url = "http://<gitlab_runner_ip>:8083"  # 请替换为实际 GitLab Runner IP
id = 9
token = "<shell_runner_token>"  # 请替换为实际 token
token_obtained_at = 2025-01-14T02:40:35Z
token_expires_at = 0001-01-01T00:00:00Z
executor = "shell"
builds_dir = "/home/gitlab-runner/builds"
clone_url = "http://<gitlab_runner_ip>:8083"
```

#### 配置说明：

- **executor**：使用 Shell Executor 来执行部署任务。
- **builds_dir**：指定构建目录的位置。
- **token**：配置 Runner 的认证信息，确保 Runner 可以与 GitLab 进行通信。

#### 3. 部署流程

1. **构建任务**：使用 Maven Docker 镜像，在 GitLab Runner 上执行 `mvn clean package` 命令，构建 JAR 包。
2. **部署任务**：构建完成后，使用 Shell Executor 进行部署操作，可以通过 SSH 或其他方式将 JAR 包部署到服务器。

#### 关键注意事项：

- 构建和部署任务分别使用不同的 Runner 来分离构建与部署逻辑。
- 使用 Docker 镜像来运行 Maven 构建，以保证构建环境的一致性。
- 配置了 Maven 本地仓库路径，避免每次构建时重复下载依赖。
