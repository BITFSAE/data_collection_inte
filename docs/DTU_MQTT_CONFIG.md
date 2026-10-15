# DTU MQTT 配置说明

本文说明车端 DTU（USR-DR154）连入车队 MQTT 服务器（bitfsae.com）的参数、每台设备的 Client ID 分配规则，以及配置完成后的验证方法。给一台新 DTU 配置时，照第 1 节的表逐项填写，再按第 3 节的步骤写入设备。

## 1. 参数表

### MQTT 参数

| 参数 | 值 |
|---|---|
| MQTT 服务器地址 | bitfsae.com |
| MQTT 服务器端口 | 1883 |
| MQTT 用户名 | vehicle_gateway |
| MQTT 密码 | 见 `docs/secrets/DTU_MQTT.local.md`（不入库） |
| MQTT Client ID | 按设备分配，见第 2 节，禁止两台设备相同 |
| MQTT 协议版本 | 3.1.1 |
| MQTT 串口传输模式 | 纯透传模式 |
| MQTT 心跳包时间 | 60 秒 |
| 启用清除会话 | 开 |
| 启用遗嘱消息 | 关 |
| 订阅主题 1 | `fsae/command`，QoS 0 |
| 发布主题 1 | `fsae/telemetry`，QoS 0，消息保留关 |
| 订阅/发布主题 2～4 | 关 |

服务器要求账号密码鉴权。用户名 `vehicle_gateway` 是车端 DTU 专用账号，不要用在命令发布端或数据消费端，那些场景另有账号。

### SSL/TLS 参数

| 参数 | 值 |
|---|---|
| SSL/TLS 加密 | 关 |

### 串口参数

| 参数 | 值 |
|---|---|
| 串口波特率 | 115200 |
| 串口校验位 | NONE（无校验） |
| 串口数据位 | 8 |
| 串口停止位 | 1 |
| 串口打包时间 | 50 毫秒 |
| 串口打包长度 | 1024 Bytes |

这组串口参数与固件 USART1 的初始化一致（`Core/Src/usart.c` 中 `MX_USART1_UART_Init`），不要单独改任何一边。

### 全局参数

| 参数 | 值 |
|---|---|
| Socket 重连间隔 | 10 秒 |
| Socket 重连次数 | 60 |
| 命令字密码 | 保持出厂默认 |
| 开机启动信息 | 保持出厂默认 |
| 无数据 & 无连接重启时间 | 300 秒 |
| 开启指令回显 | 开 |
| 开启串口数据缓存 | 开 |
| 开启串口 AT 指令 | 开 |
| 开启网络 AT 指令 | 开 |
| 开启 NTP 功能 | 关 |
| SIM 卡模式切换 | 外置卡优先模式 |

“开启串口数据缓存”保持在开：MCU 在 DTU 联网完成前发出的遥测字节会先缓存在 DTU 里，联网后补发，不会丢。

### APN 参数

| 参数 | 值 |
|---|---|
| APN 名称 | CMNET |
| APN 用户名 / 密码 | 空 |
| 鉴权方式 | NONE |

## 2. Client ID 分配规则

每台设备一个唯一 Client ID，格式 `vehicle_` 加两位编号：

| 设备 | Client ID |
|---|---|
| 主车采集盒 | vehicle_01 |
| 备车采集盒 | vehicle_02 |
| 台架调试盒 | vehicle_bench_01 |

新增设备时在本文登记，编号向后递增。调试盒统一用 `vehicle_bench_` 前缀，方便云端按前缀区分赛车数据和台架数据。

两台设备用同一个 Client ID 会互相踢下线。MQTT 3.1.1 规定同 ID 的新连接会顶掉旧连接，表现是两台设备的数据断断续续、时有时无。换板子或换 DTU 时，先查这张表，确认没有占用再分配。

## 3. 配置步骤

1. DTU 的串口（或调试口）接 USB 转 TTL，接到电脑。
2. 打开 USR 官方配置工具（官网下载，型号选 USR-DR154），串口波特率按设备当前设置连接。
3. 读取设备当前配置，按第 1 节的表逐项修改。重点核对三处：服务器地址、用户名密码、Client ID。
4. 保存配置并让设备重启。
5. 等待约 30～60 秒，设备完成注网和 MQTT 连接。用第 4 节的方法验证。

## 4. 验证方法

任选其一：

- 云端 Grafana 打开遥测面板，看 `fsae/telemetry` 是否有新数据。
- 电脑上订阅观察（密码见本地密码文件）：

```bash
mosquitto_sub -h bitfsae.com -p 1883 -u vehicle_gateway -P '<密码>' -t fsae/telemetry -v
```

- 串口环回：DTU 串口随便发几个字节，订阅端应立刻收到同样内容的 MQTT 消息。这一步能同时验证透传方向和上行链路。

## 5. 固件配套说明

固件用 `Core/Inc/usart.h` 里的开关选择上行硬件：

```c
#define Y100M_DTU_TRANSPARENT_MODE 1U
```

置 1 时 USART1 接 USR DTU，固件行为有三点变化：

- 启动时不再发送任何 AT 指令。DTU 透传模式下，串口收到的每个字节都会被发布到 `fsae/telemetry`，AT 字符串上去就是脏数据。
- `Y100M_MqttPublish()` 直接把 protobuf 原始字节写入 USART1，不再组 MQTT PUBLISH 包、不再走 `AT+CIPSEND`。
- `Y100M_IsReady()` 在启动后直接置位。DTU 掉线时 MCU 无法感知，靠 DTU 自身重连和数据缓存兜底。

DTU 配置了订阅 `fsae/command`，下行命令会从串口进入 MCU。固件目前没有解析这个方向的逻辑，收到的字节只落在 USART1 接收缓冲里。需要下行命令时再增加解析，入口参考 README 的 RS485 章节。

置 0 时恢复旧路径：固件自己跑 AT 引导，TCP 直连 `82.157.204.124:1883`，手搓无鉴权 MQTT 包。该服务器已由 bitfsae.com 取代，旧路径只留作对照，不再维护。

## 6. 密码管理

- 密码存放在 `docs/secrets/DTU_MQTT.local.md`，已被 `.gitignore` 排除，不会提交。
- 不要把密码写进代码、文档或提交信息。本仓库是公开仓库。
- 需要把密码给其他队员时，走队内私有渠道（组织私有仓库、群聊），不发 GitHub。
- 密码疑似泄露时，找服务器管理员重置 `vehicle_gateway` 账号，再更新各设备的配置。
