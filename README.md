# ThingsBoard CE 本地 Docker 环境速查

这个仓库用于在本机通过 Docker Compose 运行 ThingsBoard CE 学习环境。

当前环境包含两个服务：

- `postgres`：PostgreSQL 数据库，用来保存 ThingsBoard 数据。
- `thingsboard-ce`：ThingsBoard CE 平台服务，提供 Web UI、MQTT、REST API 等能力。

数据库数据保存在 Docker volume：`tb-postgres-data`。正常停止或重启不会丢数据。

## 1. 进入项目目录

每次操作前，先进入项目根目录：

```powershell
cd C:\gcj-programe\thingboard\thingboard
```

## 2. 第一次初始化数据库

只需要第一次搭建时执行一次。

推荐使用带演示数据的初始化方式，方便学习和观察 ThingsBoard 后台：

```powershell
docker compose run --rm -e INSTALL_TB=true -e LOAD_DEMO=true thingsboard-ce
```

如果只想做干净安装，不加载演示数据：

```powershell
docker compose run --rm -e INSTALL_TB=true thingsboard-ce
```

初始化完成后，临时容器会自动退出。

## 3. 启动环境

后台启动 ThingsBoard 和 PostgreSQL：

```powershell
docker compose up -d
```

查看容器状态：

```powershell
docker compose ps
```

启动后访问：

```text
http://localhost:8080
```

## 4. 暂停和下次运行

日常最推荐使用下面这一组命令。

停止环境，但保留数据库数据：

```powershell
docker compose down
```

下次重新运行：

```powershell
docker compose up -d
```

也可以使用更轻量的暂停方式：

```powershell
docker compose stop
```

恢复已停止的容器：

```powershell
docker compose start
```

区别：

```text
docker compose stop      只停止容器，容器还保留
docker compose start     启动已停止的容器

docker compose down      停止并删除容器，但保留数据库 volume
docker compose up -d     重新创建/启动容器，并继续使用原来的数据
```

## 5. 默认登录账号

如果初始化时使用了 `LOAD_DEMO=true`，可以使用这些账号登录：

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

如果是干净安装，通常先使用：

```text
sysadmin@thingsboard.org / sysadmin
```

这些是本地学习环境默认账号，不要暴露到公网环境。

## 6. 常用维护命令

查看 ThingsBoard 日志：

```powershell
docker compose logs -f thingsboard-ce
```

查看 PostgreSQL 日志：

```powershell
docker compose logs -f postgres
```

查看所有正在运行的 Docker 容器：

```powershell
docker ps
```

查看所有 Docker volume：

```powershell
docker volume ls
```

查看当前项目的容器状态：

```powershell
docker compose ps
```

## 7. 端口说明

当前 `docker-compose.yml` 暴露了这些端口：

```text
8080                 ThingsBoard Web UI 和 REST API
1883                 MQTT 明文接入端口
8883                 MQTT TLS 接入端口
7070                 ThingsBoard Edge RPC
5683-5688/udp        CoAP / LwM2M 相关端口
```

浏览器后台地址：

```text
http://localhost:8080
```

后续模拟设备使用 MQTT 时，通常会连接：

```text
localhost:1883
```

## 8. 端口 8080 被占用时

如果 `http://localhost:8080` 打不开，或者启动时报 8080 被占用，可以把 `docker-compose.yml` 中的：

```yaml
- "8080:8080"
```

改成：

```yaml
- "18080:8080"
```

然后重新启动：

```powershell
docker compose down
docker compose up -d
```

访问地址改为：

```text
http://localhost:18080
```

## 9. 清空并重建环境

普通学习和日常使用不要执行这个命令。

下面命令会删除容器和数据库 volume，ThingsBoard 里的设备、用户、Dashboard、遥测数据都会被清空：

```powershell
docker compose down -v
```

如果确实想从零开始，可以执行：

```powershell
docker compose down -v
docker compose run --rm -e INSTALL_TB=true -e LOAD_DEMO=true thingsboard-ce
docker compose up -d
```

再次提醒：`docker compose down -v` 会清空本地 ThingsBoard 数据。

## 10. 推荐日常流程

开始学习：

```powershell
cd C:\gcj-programe\thingboard\thingboard
docker compose up -d
docker compose ps
```

打开后台：

```text
http://localhost:8080
```

结束学习：

```powershell
cd C:\gcj-programe\thingboard\thingboard
docker compose down
```

