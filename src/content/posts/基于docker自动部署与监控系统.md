---
title: 基于Docker自动部署与监控系统
published: 2026-07-19
description: FastAPI + Nginx + Docker Compose 搭一套带 /metrics 的服务
image: " "
tags: [FastAPI,Nginx,Docker,CI/CD,Prometheus,Grafana,cAdvisor,Alertmanager]
category: 学习笔记
draft: false
---

## 写个python测试的服务

首先做一个简单的后端服务，用 Python 的 FastAPI。

`app/main.py`：

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI(title="demo-app")

VERSION = "v1"


@app.get("/")
def root():
    return {"name": "demo-app", "version": VERSION}


@app.get("/health")
def health():
    return {"status": "ok"}


# 在最后一行，把 /metrics 自动挂上
Instrumentator().instrument(app).expose(app)
```

最后那行 `Instrumentator` 会自动把每个接口的请求数、耗时这些指标挂到 `/metrics`，后面 Prometheus 直接抓这个端点就行。

## requirements.txt

记录一下项目依赖的第三方库：

```
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
prometheus-fastapi-instrumentator>=7.0.0
```

## Dockerfile

然后用 Dockerfile 把它打包成 Docker 镜像：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

RUN useradd --create-home appuser

USER appuser

EXPOSE 8044

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8044"]
```

构建镜像：

```bash
docker build -t demo:test .
```

## .dockerignore

为了构建镜像时不把那些不需要的文件也复制进去，创建个 `.dockerignore`：

```
.git
.github
__pycache__
.pytest_cache
venv
.env
*.pyc
*.pyo
```

重点把 `venv` 排掉，本机虚拟环境打进镜像会和容器里的 Python 打架。

## Nginx 配置

配一下 Nginx 需要用到的配置文件，让它把 80 端口转发到后端的 8044。

`nginx/nginx.conf`：

```nginx
upstream demo_backend {
    server app:8044;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://demo_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

`upstream` 里的 `app` 是 compose 里的服务名，Compose 自带 DNS 能解析到对应容器。

## docker-compose

然后就是 docker-compose 了，把 app 和 nginx 一起编进去管理。

`docker-compose.yml`：

```yaml
name: test
services:
  app:
    image: demo:test
    expose:
      - "8044"
    container_name: test-app
    networks:
      - app-network

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    container_name: test-nginx
    networks:
      - app-network
    depends_on:
      - app
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro

networks:
  app-network:
```

app 只对内暴露 8044，外部只走 nginx 进来。两个服务要挂在同一个 `app-network` 网络里，nginx 才能解析到 `app`。

## 测试

先测试一下这部分干的事情有没有问题。

```bash
docker build -t demo:test .
docker compose up -d
docker compose ps
```

看到 `test-app` 和 `test-nginx` 都是 running 就可以了。

验证接口：

```bash
curl http://localhost/
# {"name":"demo-app","version":"v1"}

curl http://localhost/health
# {"status":"ok"}

curl http://localhost/metrics
# 输出 Prometheus 格式的指标文本
```

三个接口都正常返回，说明从 nginx 到 app 链路通了，`/metrics` 也能被外部抓取。后面只要在 Prometheus 里把这台机器加进 targets 就能开始监控。

## 排错

实际操作时大概率会撞上这几个问题。

### 镜像不存在

`docker compose up` 时 nginx 起来了，app 一直报：

```
No such image: demo:test
```

compose 里 app 用的是 `image: demo:test`，不会自动构建。要先手动 `docker build -t demo:test .` 把镜像做出来，再 up。

### Nginx 起不来：host not found in upstream

```
host not found in upstream "app"
```

nginx 解析不到服务名 `app`。要么两个服务不在同一个网络，要么 `nginx.conf` 里的 upstream 名字和 compose 里的服务名对不上。对一下两边是不是都写的 `app`，是不是都挂了 `app-network`。

### 80 端口被占

```
bind: address already in use
```

宿主机 80 已经被别的程序占了。`sudo ss -tulnp | grep :80` 查一下是谁，停掉它，或者把 compose 里的 `"80:80"` 改成 `"8080:80"` 换个端口。

## CI/CD

前面的内容完成了本地 FastAPI、Docker、Docker Compose 和 Nginx。接下来使用 GitHub Actions 将自动化流程拆成两个职责：

- **CI（持续集成）**：检查代码、运行测试，并构建和推送 Docker 镜像。
- **CD（持续部署）**：在 CI 成功后连接服务器，部署新镜像，执行健康检查，失败时回滚。

```text
提交代码 → CI 测试 → 构建并推送镜像 → CD 部署 → 健康检查 → 成功保留 / 失败回滚
```

### CI

### 1. CI 做什么

CI 只负责验证代码和生成镜像，不负责修改服务器上的运行版本。它回答两个问题：

1. 代码是否能通过自动化测试？
2. 代码是否能成功构建成 Docker 镜像？

测试失败时，镜像构建和后续部署都必须停止。

### 2. 触发条件

当前项目使用 `.github/workflows/test.yml`：

```yaml
name: test

on: [push]
```

每次 push 都会触发 CI。学习阶段这样最容易观察；后续可以增加 Pull Request 触发，让合并前也必须通过测试。

### 3. CI 工作流

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: python -m pytest

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        run: |
          IMAGE=ghcr.io/$(echo ${{ github.repository }} | tr '[:upper:]' '[:lower:]')
          docker build -t $IMAGE:latest .
          docker push $IMAGE:latest
```

### 4. CI 执行顺序

```text
test job
  拉取代码 → 安装 Python → 安装依赖 → 执行 pytest
                                      ↓ 通过
build job
  登录 GHCR → 构建镜像 → 推送镜像
```

`build` 中的 `needs: test` 是关键：test 失败时，build 会被跳过，坏代码不会被推送。

### 5. 关键配置说明

- `ubuntu-latest`：使用 GitHub 提供的干净虚拟机，避免依赖本地环境。
- `packages: write`：允许 workflow 向 GHCR 推送镜像。
- `secrets.GITHUB_TOKEN`：GitHub 自动提供的临时令牌，不需要把个人密码写进仓库。
- `tr '[:upper:]' '[:lower:]'`：将镜像名转为小写，满足 GHCR 命名要求。
- `needs: test`：把测试作为镜像构建的门禁。

### 6. CI 验证

推送代码后，在 GitHub 的 Actions 页面确认：

```text
test job 通过 → build job 通过 → Packages 中出现镜像
```

本地也应先运行：

```bash
python -m pytest
```

不要为了验证回滚而永久跳过测试。坏版本实验应使用独立分支或明确的实验开关，实验完成后恢复正常测试门禁。

### 7. CI 常见问题

- **pytest 失败**：先在本地运行 `python -m pytest`，修复测试或代码后再推送。
- **GHCR 返回 403**：检查 `packages: write`、仓库的 Actions 权限和登录配置。
- **镜像名返回 400**：检查镜像名是否已经全部转为小写。
- **构建找不到镜像上下文**：确认 workflow 的执行目录是项目根目录，并且 Dockerfile 存在。

### CD

### 1. CD 做什么

CD 负责把 CI 产出的镜像部署到服务器：

```text
CI 成功
   ↓
SSH 连接服务器
   ↓
拉取镜像并启动新容器
   ↓
访问 /health
   ├── 返回 200：更新当前版本
   └── 检查失败：恢复上一版本
```

CI 解决“能不能发布”，CD 解决“发布后能不能正常运行”。

### 2. GitHub Secrets

在 `Settings → Secrets and variables → Actions` 中配置：

```text
SERVER_HOST       服务器公网 IP 或域名
SERVER_USERNAME   SSH 用户名
SERVER_PASSWORD   SSH 密码（学习环境使用）
SERVER_PORT       SSH 端口
```

实际项目更推荐使用 SSH 私钥。密码、私钥和 `.env` 不能提交到仓库。

### 3. CD 工作流

当前 `.github/workflows/deploy.yml` 使用 `workflow_run`：

```yaml
name: Deploy

on:
  workflow_run:
    workflows: ["test"]
    types: [completed]
    branches: [master]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          password: ${{ secrets.SERVER_PASSWORD }}
          port: ${{ secrets.SERVER_PORT }}
          script: |
            cd /home/are/qqq/AutoDeploy-demo
            git pull
            sudo ./deploy/deploy.sh
```

`workflow_run` 表示等待 `test` workflow 完成；`if` 条件保证只有 CI 成功时才部署。

### 4. 服务器准备

服务器需要提前完成：

1. 安装 Docker 和 Git；
2. 准备项目目录 `/home/are/qqq/AutoDeploy-demo`；
3. 确认服务器可以访问镜像仓库；
4. 确认部署用户有执行 Docker 的权限；
5. 如果使用 `sudo ./deploy/deploy.sh`，配置该脚本的免密码 sudo 权限。

```bash
sudo visudo
```

```text
are ALL=(ALL) NOPASSWD: /home/are/qq/AutoDeploy-demo/deploy/deploy.sh
```

### 5. 部署脚本分工

```text
deploy/
├── config.sh        保存镜像、端口和重试次数
├── health_check.sh  检查 /health 是否返回 200
└── deploy.sh        执行部署、记录版本和自动回滚
```

`config.sh` 示例：

```bash
IMAGE=ghcr.nju.edu.cn/uarelittou/autodeploy-demo
PORT=8044
CONTAINER=deploy-app
MAX_RETRIES=3
RETRY_INTERVAL=2
VERSIONS_FILE="AutoDeploy-demo_rollback"
```

`health_check.sh` 连续检查 `/health`。返回 200 时退出码为 0；多次失败时退出码为 1，由 `deploy.sh` 触发回滚。

`deploy.sh` 的核心顺序：

```text
读取 current
   ↓
拉取镜像
   ↓
停止旧容器并启动新容器
   ↓
执行健康检查
   ├── 成功：current 更新为新版本
   └── 失败：删除新容器，恢复 current
```

版本记录中的 `current` 表示当前正在运行且已经验证成功的版本。回滚目标应是 `current`，而不是更早的 `previous`。

### 6. CD 验证

服务器首次部署前，可以先手动验证镜像：

```bash
docker pull ghcr.io/uarelittou/autodeploy-demo:latest
curl http://localhost:8044/health
```

确认脚本权限：

```bash
cd ~/AutoDeploy-demo/deploy
sudo chmod +x *.sh
```

执行部署：

```bash
sudo ./deploy.sh
```

检查结果：

```bash
curl http://localhost:8044/health
docker ps
cat deploy/AutoDeploy-demo_rollback
```

### 7. CD 常见问题

- **脚本没有执行权限**：使用 `chmod +x *.sh`，并提交 executable 权限变化。
- **容器名冲突**：检查是否残留同名容器，再清理后重试。
- **端口被占用**：使用 `sudo ss -tulnp | grep :8044` 检查端口。
- **镜像拉取失败**：确认 CI 推送地址和 `config.sh` 中的 `IMAGE` 指向同一个仓库。
- **健康检查失败**：检查容器日志、端口映射和 `/health` 接口。

### CI/CD 联动

### 1. 正常发布

```text
git push
   ↓
CI：pytest 通过
   ↓
CI：构建并推送镜像
   ↓
CD：SSH 进入服务器
   ↓
CD：启动新容器
   ↓
/health 返回 200
   ↓
新版本上线
```

### 2. 测试失败时阻止发布

```text
pytest 失败
   ↓
test job 失败
   ↓
build job 因 needs: test 被跳过
   ↓
CD 不执行
   ↓
服务器仍运行旧版本
```

### 3. 部署失败时自动回滚

实验时可以让 `/health` 返回错误，观察：

```text
新版本启动
   ↓
健康检查失败
   ↓
删除新容器
   ↓
恢复 current 版本
   ↓
旧版本健康检查通过
```

回滚成功后，业务恢复，但本次部署仍应返回非零状态，因为发布本身是失败的。实验完成后恢复正常代码，并确认测试步骤没有被跳过。

















## 监控与告警

前面的应用、Docker、Compose 和 Nginx 已经完成。接下来加入 Prometheus、cAdvisor、Grafana 和 Alertmanager，形成完整的监控与告警链路：

```text
FastAPI /metrics → Prometheus → Grafana
                         ↓
                    Alertmanager → Gmail

cAdvisor ───────────→ Prometheus
```

### Prometheus：采集应用指标

Prometheus 负责定时抓取应用和其他监控组件的指标。在 `docker-compose.yml` 中添加：

先在docker-compose文件里追加Prometheus的容器

```
  prometheus:
    image: prom/prometheus:v3.5.0
    container_name: test-prometheus
    ports:
      - "9090:9090"
    networks:
      - app-network
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"

```

在最下面也追加挂载：

```
volumes:
  prometheus-data:
```



在项目目录下创建prometheus/prometheus.yml配置文件

```
# ==========================================
# 全局配置 (Global Config)
# ==========================================
global:
  scrape_interval: 15s      # 抓取间隔：每 15 秒拉取一次数据
  evaluation_interval: 15s  # 评估间隔：每 15 秒计算一次告警规则
  scrape_timeout: 10s       # 抓取超时：超过 10 秒未拉取到数据则作废

# ==========================================
# 告警配置 (Alerting Config) - 暂不开启
# ==========================================
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - localhost:9093  # 安装 Alertmanager 后取消注释

# ==========================================
# 告警规则文件路径 (Rule Files) - 暂不开启
# ==========================================
rule_files:
  # - "alert_rules.yml"     # 后续可配置具体的告警条件

# ==========================================
# 抓取任务配置 (Scrape Configs) - 核心部分
# ==========================================
scrape_configs:

  # 任务 1：监控 Prometheus 自身状态
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          role: "monitor-center"    # 标签：监控中心

  # 任务 2：监控 测试用的脚本服务 
  
  - job_name: "demo-app"
    metrics_path: /metrics
    static_configs:
      - targets: ["app:8044"]


      # 后续添加新机器按以下格式：
      # - targets: ["192.168.1.100:9100"]
      #   labels:
      #     env: "prod"
      #     hostname: "web-server-01"
```



然后重启两个容器

```
docker compose up -d --force-recreate app prometheus
```

去Prometheus后台看看，< prometheus，demo-app >两个都是up说明可以了

```
http://localhost:9090/targets
```







### cAdvisor：采集 Docker 容器指标

接下来是监控 Docker 容器的 cAdvisor。

在docerk-compose文件添加服务：

```
cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.52.0
    container_name: test-cadvisor
    ports:
      - "8080:8080"
    networks:
      - app-network
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk:/dev/disk:ro
    privileged: true
```

在prometheus.yml文件里追加：

```
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

启动:

```
docker compose up -d cadvisor prometheus
```

重启prometheus

```
docker compose up -d --force-recreate prometheus
```

然后去Prometheus后台看看，< prometheus，demo-app，cadvisor >三个都是up说明可以了









### Grafana：展示监控数据

Grafana 负责把 Prometheus 的指标显示成图表。

```
  grafana:
    image: grafana/grafana:12.0.0
    container_name: test-grafana
    ports:
      - "3000:3000"
    networks:
      - app-network
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
```



挂载追加

```
volumes:
  prometheus-data:
  grafana-data:
```

启动grafana

```
docker compose up -d grafana
```

进入grafana

```
http://localhost:3000
```

默认账号密码都是 : admin



接下来就是在grafana里添加这些监控数据了

进入：

Connections → Data sources → Add new data source → Prometheus

URL 填：

http://prometheus:9090

点击：

Save & test

看到连接成功即可



然后我是导入别人的仪表盘的，id是 9621

![image-20260723140309408](C:\Users\a1581\AppData\Roaming\Typora\typora-user-images\image-20260723140309408.png)





### Alertmanager：处理告警通知

Prometheus 负责判断告警条件，Alertmanager 负责接收告警、分组并发送通知。

在prometheus目录下创建/rules/application.yml

内容是:

```
groups:
  - name: demo-app-alerts
    rules:
      - alert: DemoAppDown
        expr: up{job="demo-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Demo app is down"
          description: "demo-app has been unavailable for more than one minute."

```

在compose文件里的 Prometheus 的 volumes 中追加

```
./prometheus/rules:/etc/prometheus/rules:ro
```



接下来我们验证一下

```
docker stop test-app

等待约 1 分钟，然后打开：

http://localhost:9090/alerts

确认 DemoAppDown 变成 FIRING 后，再执行：

docker start test-app

确认它恢复为 INACTIVE。
```

说明Prometheus 告警规则正常工作



接下来我们在在prometheus目录下创建alertmanager.yml文件

```
global:
  resolve_timeout: 5m

route:
  receiver: default

templates: []

receivers:
  - name: default
```

prometheus.yml配置文件也要修改一下

```
# ==========================================
# 告警配置 (Alerting Config) -
# ==========================================
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

# ==========================================
# 告警规则文件路径 (Rule Files) - 暂不开启
# ==========================================
rule_files:
  - /etc/prometheus/rules/*.yml  
```

然后修改 docker-compose.yml新增 Alertmanager 服务：

```
alertmanager:
  image: prom/alertmanager:v0.33.1
  container_name: test-alertmanager
  ports:
    - "9093:9093"
  networks:
    - app-network
  volumes:
    - ./prometheus/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    - alertmanager-data:/alertmanager
  command:
    - "--config.file=/etc/alertmanager/alertmanager.yml"
    - "--storage.path=/alertmanager"
```

同时新增数据卷：

```
alertmanager-data:
```



现在打开9093端口就能看到 Alertmanager 页面了：

```
http://localhost:9093
```



然后我们测试一遍:

```
docker stop test-app
等待 1 分钟，打开：
http://localhost:9093

应该能看到一个信息：
1 alert
2026-07-23T08:04:09.590Z
alertname="DemoAppDown"
instance="app:8044"
job="demo-app"
severity="critical"

恢复应用：
docker start test-app
等待约 1 分钟后发现Alertmanager 中的告警应显示恢复。
```



#### 配置 Gmail 邮件通知

这里我以邮件告警为例，邮件为Gmail（去gmail里获取16位App Password-记得把中间的空格去掉）

去alertmanager.yml文件修改

```
global:
  resolve_timeout: 5m

route:
  receiver: email
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 4h

receivers:
  - name: email
    email_configs:
      - to: 'arelittouwu@gmail.com'
        from: 'Prometheus Alert <arelittouwu@gmail.com>'
        smarthost: 'smtp.gmail.com:465'
        auth_username: 'arelittouwu@gmail.com'
        auth_password_file: '/etc/alertmanager/gmail-app-password.txt'
        require_tls: false
        force_implicit_tls: true
        send_resolved: true
```



不过我在这里会把那个16位密码用专门的文件来存放：

在/prometheus目录下的创建个gmail-app-password.txt里放密码

然后修改alertmanager.yml文件：

```
smtp_auth_password_file: '/etc/alertmanager/gmail-app-password.txt'
```

docker-compose里的alertmanager部分挂载加一个：

```
- ./prometheus/gmail-app-password.txt:/etc/alertmanager/gmail-app-password.txt:ro
```



好了，接下来就是重新加载测试一下:

```
docker compose up -d --force-recreate alertmanager
docker stop test-app
等待约 1 分钟，检查：
http://localhost:9093/#/alerts
```





![image-20260723175310968](C:\Users\a1581\AppData\Roaming\Typora\typora-user-images\image-20260723175310968.png)

回复后

![image-20260723175700012](C:\Users\a1581\AppData\Roaming\Typora\typora-user-images\image-20260723175700012.png)
