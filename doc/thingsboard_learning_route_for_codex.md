# ThingsBoard 平台学习路线与 Codex 协作说明

## 1. 学习目标

本学习路线用于从嵌入式开发视角入门 ThingsBoard 平台。

目标不是一次性学习 ThingsBoard 的全部功能，而是先跑通一个完整的 IoT 设备接入闭环：

```text
设备模拟程序 → MQTT → ThingsBoard → Dashboard → Rule Chain → Alarm → RPC / Attributes → OTA / Gateway
```

最终希望完成一个与实际项目接近的 Demo：

```text
毫米波雷达布防报警设备 Demo
```

设备模拟数据包括：

```json
{
  "target": true,
  "distance": 6.5,
  "speed": 0.7,
  "signal": 68,
  "battery": 86,
  "alarm": true
}
```

平台侧完成：

- 设备接入；
- 遥测数据展示；
- Dashboard 可视化；
- Shared Attributes 下发配置；
- RPC 控制布防、撤防、重启；
- Rule Chain 创建报警；
- OTA 流程理解；
- Gateway 子设备接入理解。

---

## 2. 推荐环境

### 2.1 本地运行环境

建议使用：

```text
Windows 电脑 + Docker Desktop + ThingsBoard CE + MQTTX / Python MQTT 脚本
```

不建议一开始直接在公司 ThingsBoard 平台上实验，因为会影响真实设备、规则链和数据。

本地 Docker 环境更适合学习：

- 可随意创建、删除设备；
- 可随意修改 Rule Chain；
- 可测试 OTA；
- 可测试 Gateway；
- 出错后可以重建环境。

---

## 3. 项目目录建议

建议在本地创建如下目录：

```text
thingsboard-study/
├── docker-compose.yml
├── README.md
├── mqtt-simulator/
│   ├── radar_simulator.py
│   ├── requirements.txt
│   └── config.example.json
├── payloads/
│   ├── telemetry_radar.json
│   ├── attributes_shared.json
│   └── rpc_examples.json
├── notes/
│   ├── 01_device.md
│   ├── 02_telemetry.md
│   ├── 03_attributes.md
│   ├── 04_rpc.md
│   ├── 05_rule_chain.md
│   ├── 06_dashboard.md
│   ├── 07_ota.md
│   └── 08_gateway.md
└── ota-test/
    ├── fake_firmware_v1.0.0.bin
    └── ota_downloader.py
```

Codex 可以协助生成：

- `docker-compose.yml`；
- MQTT 模拟设备脚本；
- 示例 Payload；
- OTA 模拟下载脚本；
- 学习笔记模板；
- README 使用说明。

---

## 4. 第一阶段：Docker 搭建 ThingsBoard CE

### 4.1 目标

在本机运行 ThingsBoard CE，并能登录后台。

### 4.2 docker-compose.yml

建议使用 ThingsBoard + PostgreSQL 的单实例镜像。

```yaml
services:
  mytb:
    restart: always
    image: "thingsboard/tb-postgres:latest"
    ports:
      - "8080:9090"
      - "1883:1883"
      - "7070:7070"
      - "5683-5688:5683-5688/udp"
    environment:
      TB_QUEUE_TYPE: in-memory
    volumes:
      - mytb-data:/data
      - mytb-logs:/var/log/thingsboard

volumes:
  mytb-data:
  mytb-logs:
```

### 4.3 启动命令

```powershell
docker compose up -d
```

查看日志：

```powershell
docker compose logs -f
```

浏览器访问：

```text
http://localhost:8080
```

常用默认账号：

```text
System Administrator:
sysadmin@thingsboard.org / sysadmin

Tenant Administrator:
tenant@thingsboard.org / tenant

Customer User:
customer@thingsboard.org / customer
```

学习时主要使用：

```text
tenant@thingsboard.org / tenant
```

### 4.4 验收标准

能进入 ThingsBoard 后台，并看到以下菜单：

- Devices；
- Dashboards；
- Rule Chains；
- Alarms；
- Device Profiles；
- OTA updates。

---

## 5. 第二阶段：创建第一个设备 radar_001

### 5.1 目标

理解 ThingsBoard 中 Device 的概念。

### 5.2 操作步骤

进入：

```text
Entities → Devices → Add new device
```

创建设备：

```text
radar_001
```

打开设备详情，找到：

```text
Access Token
```

这个 Token 是设备接入平台时使用的身份凭证。

### 5.3 需要记录

```text
Device Name: radar_001
Access Token: 手动复制
MQTT Host: localhost
MQTT Port: 1883
Telemetry Topic: v1/devices/me/telemetry
```

### 5.4 验收标准

知道 Access Token 的作用，并能在设备详情页找到 Latest telemetry。

---

## 6. 第三阶段：用 MQTT 模拟设备上报遥测

### 6.1 目标

跑通最小链路：

```text
模拟设备 → MQTT → ThingsBoard → Latest telemetry
```

### 6.2 MQTT 连接参数

```text
Host: localhost
Port: 1883
Username: radar_001 的 Access Token
Password: 留空
Topic: v1/devices/me/telemetry
```

### 6.3 示例 Payload

```json
{
  "target": true,
  "distance": 6.5,
  "speed": 0.7,
  "signal": 68,
  "battery": 86
}
```

### 6.4 使用 mosquitto_pub 测试

```powershell
mosquitto_pub -h localhost -p 1883 -t v1/devices/me/telemetry -u "你的设备Token" -m "{\"target\":true,\"distance\":6.5,\"speed\":0.7,\"signal\":68,\"battery\":86}"
```

### 6.5 Codex 任务

让 Codex 生成一个 Python MQTT 模拟脚本：

```text
请帮我在 mqtt-simulator 目录下生成 radar_simulator.py。
要求：
1. 使用 paho-mqtt。
2. 从 config.example.json 读取 host、port、access_token、publish_interval。
3. 周期性向 v1/devices/me/telemetry 发布模拟雷达数据。
4. distance、speed、signal、battery 使用随机值。
5. target 随机 true/false。
6. 当 target=true 且 distance < alarm_distance 时 alarm=true。
7. 控制台打印每次发布的 payload。
```

### 6.6 验收标准

ThingsBoard 设备详情页的 Latest telemetry 能看到：

- target；
- distance；
- speed；
- signal；
- battery；
- alarm。

---

## 7. 第四阶段：制作雷达报警 Dashboard

### 7.1 目标

理解数据展示流程：

```text
Telemetry → Timeseries → Dashboard Widget
```

### 7.2 建议展示项

| 字段 | 类型 | 展示方式 |
|---|---|---|
| target | Boolean | 状态灯 |
| alarm | Boolean | 报警状态灯 |
| distance | Number | 数值卡片 + 曲线 |
| speed | Number | 数值卡片 |
| signal | Number | 仪表盘 |
| battery | Number | 电量条 |

### 7.3 Dashboard 设计建议

页面可以分成三块：

```text
设备状态区：是否在线、是否布防、电量、信号强度
目标检测区：target、distance、speed
报警区：alarm、报警列表、历史趋势
```

### 7.4 验收标准

打开 Dashboard 后，可以直观看到：

- 是否检测到目标；
- 目标距离；
- 是否报警；
- 电池电量；
- 历史距离曲线。

---

## 8. 第五阶段：学习 Attributes

### 8.1 目标

理解 Telemetry 和 Attributes 的区别。

```text
Telemetry：设备实时数据，例如距离、速度、电量。
Attributes：设备配置或状态，例如布防状态、报警距离、灵敏度、固件版本。
```

### 8.2 三类属性

| 类型 | 方向 | 作用 |
|---|---|---|
| Client attributes | 设备 → 平台 | 设备主动上报的属性 |
| Shared attributes | 平台 → 设备 | 平台下发给设备的配置 |
| Server attributes | 平台内部 | 平台保存的内部属性 |

### 8.3 建议 Shared Attributes

```json
{
  "armed": true,
  "alarm_distance": 8,
  "sensitivity": 3
}
```

### 8.4 Codex 任务

在 `radar_simulator.py` 中加入 Shared Attributes 支持：

```text
请修改 radar_simulator.py：
1. 连接 ThingsBoard 后订阅 Shared Attributes 更新主题。
2. 支持 armed、alarm_distance、sensitivity 三个参数。
3. 收到 armed=false 时，设备仍然上报 target 和 distance，但 alarm 永远为 false。
4. 收到 alarm_distance 后更新本地报警距离阈值。
5. 控制台打印属性更新内容。
```

### 8.5 验收标准

在 ThingsBoard 修改 Shared Attributes 后，模拟设备能收到并改变报警逻辑。

---

## 9. 第六阶段：学习 RPC

### 9.1 目标

理解云端主动控制设备。

RPC 更像“立即执行命令”，Shared Attributes 更像“保存配置参数”。

### 9.2 建议 RPC 方法

```json
{
  "method": "setArmed",
  "params": true
}
```

```json
{
  "method": "setAlarmDistance",
  "params": 8
}
```

```json
{
  "method": "reboot",
  "params": {}
}
```

```json
{
  "method": "getStatus",
  "params": {}
}
```

### 9.3 Codex 任务

```text
请修改 radar_simulator.py，加入 ThingsBoard Server-side RPC 支持：
1. 订阅 v1/devices/me/rpc/request/+。
2. 支持 setArmed、setAlarmDistance、reboot、getStatus 四个方法。
3. setArmed 修改本地 armed 状态。
4. setAlarmDistance 修改本地 alarm_distance。
5. reboot 不需要真正重启，只打印“模拟重启”。
6. getStatus 返回当前 armed、alarm_distance、last_distance、last_alarm。
7. 对双向 RPC 返回响应到 v1/devices/me/rpc/response/{requestId}。
```

### 9.4 验收标准

在 ThingsBoard 页面发 RPC 后，模拟设备控制台能打印命令，并能返回结果。

---

## 10. 第七阶段：学习 Rule Chain

### 10.1 目标

理解平台侧规则处理。

数据链路：

```text
设备遥测上报 → Rule Chain 判断 → 创建 Alarm → Dashboard 展示
```

### 10.2 简单规则

第一版只判断：

```text
alarm == true
```

满足条件后创建 Alarm：

```text
Alarm Type: Intrusion Alarm
Alarm Severity: Major
```

### 10.3 进阶规则

第二版判断：

```text
target == true && distance < 8 && armed == true
```

注意：如果 armed 是 Shared Attribute，Rule Chain 需要能读取对应属性。

### 10.4 业务场景

用于毫米波雷达布防报警：

```text
目标被检测到
    ↓
距离小于报警阈值
    ↓
当前处于布防状态
    ↓
创建入侵报警
```

### 10.5 验收标准

当模拟设备上报：

```json
{
  "target": true,
  "distance": 6.5,
  "alarm": true
}
```

平台能自动创建报警。

---

## 11. 第八阶段：完整 Demo：毫米波雷达布防报警

### 11.1 目标

把前面的功能组合成一个完整项目。

### 11.2 Demo 功能清单

设备模拟端：

- 周期上报 target；
- 周期上报 distance；
- 周期上报 speed；
- 周期上报 signal；
- 周期上报 battery；
- 根据 armed 和 alarm_distance 计算 alarm；
- 接收 Shared Attributes；
- 接收 RPC。

ThingsBoard 平台端：

- 创建设备 radar_001；
- Dashboard 展示设备状态；
- Rule Chain 创建报警；
- RPC 控制布防/撤防；
- Shared Attributes 配置报警距离。

### 11.3 建议 README 展示内容

让 Codex 生成项目 README：

```text
请生成 README.md，内容包括：
1. 项目目的。
2. ThingsBoard Docker 启动方式。
3. 如何创建设备并获取 Access Token。
4. 如何配置 radar_simulator.py。
5. 如何运行模拟器。
6. 如何在 Dashboard 查看数据。
7. 如何测试 Shared Attributes。
8. 如何测试 RPC。
9. 如何测试报警规则链。
```

### 11.4 验收标准

可以演示以下流程：

```text
启动模拟设备
    ↓
ThingsBoard 显示数据
    ↓
Dashboard 显示目标状态
    ↓
平台 RPC 下发布防
    ↓
设备 target=true 且 distance < 阈值
    ↓
平台创建报警
```

---

## 12. 第九阶段：学习 OTA

### 12.1 目标

理解 ThingsBoard OTA 的平台侧和设备侧分工。

平台负责：

- 上传固件包；
- 管理版本；
- 分配固件给设备；
- 通知设备更新；
- 提供分片下载；
- 记录升级状态。

设备负责：

- 收到 OTA 属性；
- 判断版本；
- 下载固件；
- 校验 SHA-256；
- 写入 Flash 或文件；
- 设置升级标志；
- 重启；
- 成功确认；
- 失败回滚。

### 12.2 学习顺序

不要直接拿 STM32 做第一版 OTA。

建议顺序：

```text
PC 模拟 OTA 下载
    ↓
Linux 程序模拟 OTA
    ↓
真实设备 Flash 写入
    ↓
Bootloader A/B 分区
    ↓
断电保护和回滚
```

### 12.3 OTA 状态

```text
QUEUED       平台排队通知
INITIATED    平台已通知设备
DOWNLOADING  设备正在下载
DOWNLOADED   设备下载完成
VERIFIED     设备校验通过
UPDATING     设备正在升级/重启
UPDATED      升级成功
FAILED       升级失败
```

### 12.4 Codex 任务

```text
请生成 ota-test/ota_downloader.py，用于模拟 ThingsBoard OTA 下载流程：
1. 连接 ThingsBoard MQTT。
2. 订阅 OTA 相关 Shared Attributes。
3. 收到 fw_title、fw_version、fw_size、fw_checksum 后开始下载。
4. 使用 MQTT 分片请求固件。
5. 将固件保存为 downloaded_firmware.bin。
6. 计算 SHA-256 并和平台下发的 checksum 对比。
7. 按流程上报 DOWNLOADING、DOWNLOADED、VERIFIED、UPDATED 或 FAILED。
8. 暂时不做真实写 Flash，只做 PC 文件模拟。
```

### 12.5 验收标准

能完整解释：

```text
ThingsBoard 不是替你完成嵌入式 OTA，而是提供 OTA 管理和固件分发能力。
```

---

## 13. 第十阶段：学习 Gateway

### 13.1 目标

理解网关代理子设备接入平台。

常见结构：

```text
多个传感器 / MCU / 雷达节点
    ↓
网关
    ↓
ThingsBoard
```

### 13.2 普通网关和 Mesh 网关理解

普通网关：

```text
子设备 → 网关 → 云平台
```

Mesh 网关：

```text
子设备 → 子设备 → 子设备 → Mesh 网关 → 云平台
```

ThingsBoard 主要看到的是：

```text
网关通过 MQTT / HTTP 接入平台
```

至于网关下面是普通星型网络还是 Mesh 网络，是设备侧和网关侧的实现问题。

### 13.3 建议学习方向

后续可以学习：

- ThingsBoard IoT Gateway；
- MQTT 子设备接入；
- Modbus 子设备接入；
- BLE 子设备接入；
- 自己写 C/Python 网关程序；
- 网关代理 OTA。

### 13.4 Codex 任务

```text
请帮我写一份 notes/08_gateway.md，要求说明：
1. 网关的作用。
2. ThingsBoard Gateway 的基本概念。
3. 普通网关和 Mesh 网关的区别。
4. 子设备如何映射成 ThingsBoard Device。
5. 网关代理子设备上传遥测的流程。
6. 网关代理 OTA 的大概思路。
```

### 13.5 验收标准

能画出并解释：

```text
雷达节点 1
雷达节点 2
门磁节点
温湿度节点
    ↓
网关
    ↓
ThingsBoard
```

并知道平台侧如何分别查看这些子设备的数据。

---

## 14. Codex 协作建议

### 14.1 使用 Codex 的方式

不要一次让 Codex 写完全部东西。建议按阶段交给 Codex：

```text
第一步：生成 docker-compose.yml 和 README。
第二步：生成 MQTT 模拟设备脚本。
第三步：加入 Shared Attributes。
第四步：加入 RPC。
第五步：整理 Dashboard 配置说明。
第六步：整理 Rule Chain 配置说明。
第七步：生成 OTA 模拟下载脚本。
第八步：整理 Gateway 学习笔记。
```

### 14.2 给 Codex 的总提示词

可以复制下面这段给 Codex：

```text
我正在学习 ThingsBoard 平台，目标是从嵌入式开发视角跑通一个毫米波雷达布防报警 Demo。

请你基于当前目录，逐步帮助我生成学习项目。不要一次性生成过复杂的工程。

要求：
1. 使用 Docker Compose 运行 ThingsBoard CE。
2. 使用 Python + paho-mqtt 编写模拟设备 radar_simulator.py。
3. 模拟设备通过 MQTT 向 ThingsBoard 上报遥测数据。
4. 遥测字段包括 target、distance、speed、signal、battery、alarm。
5. 支持 Shared Attributes：armed、alarm_distance、sensitivity。
6. 支持 RPC：setArmed、setAlarmDistance、reboot、getStatus。
7. 后续再加入 OTA 模拟下载脚本。
8. 每一步都要写 README 或注释，方便我学习和记录。
9. 不要直接接真实硬件，先用 PC 模拟完整流程。
10. 代码要简单清晰，适合嵌入式软件工程师理解。

请先检查当前目录结构，然后只完成第一阶段：生成 docker-compose.yml、README.md 和目录结构。
```

### 14.3 代码风格要求

让 Codex 遵守：

- 代码简单，不要过度封装；
- 每个文件作用清晰；
- 配置和代码分离；
- Access Token 不要硬编码在代码里；
- 使用 `config.example.json` 做模板；
- 日志打印清楚；
- 出错要有提示；
- README 写清楚运行步骤。

---

## 15. 最终掌握标准

完成这条路线后，应能回答以下问题：

1. ThingsBoard 中 Device 是什么？
2. Access Token 有什么作用？
3. Telemetry 和 Attributes 有什么区别？
4. Shared Attributes 适合做什么？
5. RPC 和 Shared Attributes 有什么区别？
6. Dashboard 如何绑定设备数据？
7. Rule Chain 如何创建报警？
8. ThingsBoard OTA 能做什么，不能做什么？
9. 网关和普通设备直连有什么区别？
10. Mesh 网关和普通网关有什么区别？
11. 如何把一个嵌入式设备接入 ThingsBoard？
12. 如何把毫米波雷达报警项目抽象成 IoT 平台模型？

---

## 16. 推荐学习顺序总结

```text
01 Docker 跑起 ThingsBoard
02 创建设备 radar_001
03 MQTT 上报遥测
04 Dashboard 显示数据
05 Shared Attributes 下发配置
06 RPC 远程控制
07 Rule Chain 创建报警
08 完成雷达报警 Demo
09 OTA PC 模拟下载
10 Gateway 子设备接入理解
```

先把前 8 步跑通，再看 OTA 和 Gateway。

