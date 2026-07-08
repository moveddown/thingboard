# 第一阶段：用 Docker 搭建本地 ThingsBoard CE 环境

本阶段目标很简单：先把 ThingsBoard CE 在本机跑起来，能打开网页后台并登录。不要急着接设备、写 MQTT、做 Dashboard；第一阶段只解决“平台环境可用”这件事。

参考资料：

- 当前学习路线：`doc/thingsboard_learning_route_for_codex.md`
- ThingsBoard CE Docker Windows 官方文档：https://thingsboard.io/docs/installation/docker-windows/

## 1. 本阶段要完成什么

你需要完成下面几件事：

1. 确认电脑已经安装并启动 Docker Desktop。
2. 在项目根目录准备 `docker-compose.yml`。
3. 使用 Docker Compose 拉取并启动 ThingsBoard CE 和 PostgreSQL。
4. 初始化 ThingsBoard 数据库。
5. 打开 `http://localhost:8080`，进入 ThingsBoard 登录页。
6. 用默认账号登录后台。
7. 确认后台菜单里能看到设备、仪表盘、规则链、告警等入口。
8. 学会停止、重启、查看日志这几个基础命令。

完成后，你就有了一个本地 IoT 平台沙盒，后面才能继续做设备接入、遥测上报、Dashboard、RPC、Rule Chain 等练习。

## 2. 环境准备

推荐环境：

```text
Windows + Docker Desktop + PowerShell + 浏览器
```

最低建议资源：

```text
CPU: 1 core 以上
RAM: 4 GB 以上
```

检查 Docker 是否可用：

```powershell
docker --version
docker compose version
docker ps
```

如果 `docker ps` 能正常输出，说明 Docker Desktop 已经启动。

## 3. 先理解这两个容器

当前官方 Docker 方式会启动两个核心服务：

```text
postgres        PostgreSQL 数据库，用来保存设备、用户、遥测、规则链等数据
thingsboard-ce  ThingsBoard CE 平台本体，提供 Web UI、MQTT 接入、规则引擎等能力
```

第一阶段你暂时不需要 Kafka。学习和 PoC 阶段使用默认的 `in-memory` 队列就够了。

## 4. 建议创建的 docker-compose.yml

在项目根目录创建：

```text
D:\gcj-program\project-Manager\thingboard\docker-compose.yml
```

建议内容如下：

```yaml
services:
  postgres:
    restart: always
    image: "postgres:18"
    ports:
      - "5432"
    environment:
      POSTGRES_DB: thingsboard
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d thingsboard"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  thingsboard-ce:
    restart: always
    image: "thingsboard/tb-node:4.3.1.3"
    ports:
      - "8080:8080"
      - "7070:7070"
      - "1883:1883"
      - "8883:8883"
      - "5683-5688:5683-5688/udp"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
    environment:
      TB_SERVICE_ID: tb-ce-node
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/thingsboard
      SPRING_DATASOURCE_PASSWORD: postgres
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres-data:
    name: tb-postgres-data
    driver: local
```

说明：

- `8080`：ThingsBoard Web UI 和 REST API。
- `1883`：MQTT 明文接入端口，后面模拟雷达设备会用到。
- `7070`：ThingsBoard Edge RPC，第一阶段先不用。
- `5683-5688/udp`：CoAP / LwM2M 相关端口，第一阶段先不用。
- `postgres-data`：数据库持久化卷，容器重启后数据还在。

## 5. 初始化数据库

第一次启动 ThingsBoard 前，需要初始化数据库。

进入项目目录：

```powershell
cd D:\gcj-program\project-Manager\thingboard
```

初始化方式有两种。

方式一：带演示数据，适合初学者先看看平台长什么样。

```powershell
docker compose run --rm -e INSTALL_TB=true -e LOAD_DEMO=true thingsboard-ce
```

方式二：干净安装，只初始化系统数据。

```powershell
docker compose run --rm -e INSTALL_TB=true thingsboard-ce
```

建议你第一次学习用方式一，带演示数据更容易观察设备、Dashboard、告警这些概念。

初始化完成后，这个临时容器会自动退出。

## 6. 启动 ThingsBoard

启动：

```powershell
docker compose up -d
```

查看容器状态：

```powershell
docker compose ps
```

查看 ThingsBoard 日志：

```powershell
docker compose logs -f thingsboard-ce
```

日志里看到类似启动完成的信息后，浏览器访问：

```text
http://localhost:8080
```

## 7. 默认登录账号

如果你使用了 `LOAD_DEMO=true`，可以用下面账号登录：

```text
System Administrator:
sysadmin@thingsboard.org / sysadmin

Tenant Administrator:
tenant@thingsboard.org / tenant

Customer User:
customer@thingsboard.org / customer
```

学习阶段优先使用：

```text
tenant@thingsboard.org / tenant
```

如果你选择的是干净安装，通常只有系统管理员账号可用：

```text
sysadmin@thingsboard.org / sysadmin
```

注意：这些是本地学习环境默认密码。不要把默认密码暴露到公网环境。

## 8. 第一阶段验收标准

做到下面这些，就算第一阶段完成：

- `docker compose ps` 里能看到 `postgres` 和 `thingsboard-ce` 正常运行。
- 浏览器能打开 `http://localhost:8080`。
- 可以登录 ThingsBoard 后台。
- 能在后台看到这些入口中的大部分：
  - Devices
  - Dashboards
  - Rule Chains
  - Alarms
  - Device Profiles
  - OTA updates
- 你知道 `1883` 是后续 MQTT 设备接入端口。
- 你知道 PostgreSQL 数据保存在 Docker volume `tb-postgres-data` 中。

## 9. 常用维护命令

停止容器，但保留数据：

```powershell
docker compose down
```

重新启动：

```powershell
docker compose up -d
```

查看日志：

```powershell
docker compose logs -f thingsboard-ce
```

查看所有容器：

```powershell
docker ps
```

查看 Docker 卷：

```powershell
docker volume ls
```

彻底删除本地 ThingsBoard 数据需要删除 volume，这会清空平台数据。练习时不要随手执行：

```powershell
docker compose down -v
```

## 10. 常见问题

### 端口 8080 被占用

如果 `8080` 已经被其他软件占用，可以把 compose 里的：

```yaml
- "8080:8080"
```

改成：

```yaml
- "18080:8080"
```

然后浏览器访问：

```text
http://localhost:18080
```

### 镜像拉取很慢

这是 Docker 镜像网络问题，不是 ThingsBoard 配置问题。可以先确认 Docker Desktop 正常运行，再尝试重新执行：

```powershell
docker compose pull
```

### 初始化后登录没有 tenant 账号

通常是因为初始化时没有加 `LOAD_DEMO=true`。干净安装只适合从零配置平台。初学阶段建议带演示数据初始化。

如果你想重来，可以删除 volume 后重新初始化，但这会清空数据：

```powershell
docker compose down -v
docker compose run --rm -e INSTALL_TB=true -e LOAD_DEMO=true thingsboard-ce
docker compose up -d
```

## 11. 练习记录表

你可以按这个顺序练习，并在完成后打勾：

```text
[ ] Docker Desktop 已启动
[ ] docker compose version 正常
[ ] docker-compose.yml 已创建
[ ] 数据库初始化完成
[ ] docker compose up -d 启动成功
[ ] http://localhost:8080 能打开
[ ] tenant 账号能登录
[ ] 看到了 Devices / Dashboards / Rule Chains / Alarms
[ ] 会使用 docker compose logs -f thingsboard-ce 看日志
[ ] 会使用 docker compose down 停止环境
```

## 12. 完成本阶段后再做什么

下一阶段再进入 ThingsBoard 后台创建第一个真实练习设备：

```text
Device Name: radar_001
```

然后复制它的 Access Token，为后面的 Python MQTT 雷达模拟器做准备。
