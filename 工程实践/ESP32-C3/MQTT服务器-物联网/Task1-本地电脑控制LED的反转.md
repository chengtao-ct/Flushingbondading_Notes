---
aliases:
  - 本地 MQTT
  - ESP32 MQTT LED
  - Mosquitto 闭环
  - 本地电脑控制 LED
tags:
  - 工程实践
  - ESP32
  - ESP-IDF
  - MQTT
  - Mosquitto
  - WiFi
  - JSON
  - 物联网
status: project-note
date: 2026-05-25
related:
  - "[[MQTT协议]]"
  - "[[WIFI]]"
  - "[[IOT连接]]"
---

# Task1：本地电脑控制 LED 的反转

> [!abstract] 核心摘要
> 这个任务完成了一个最小但完整的物联网闭环：电脑运行 Mosquitto Broker，ESP32 连接同一个局域网，通过 MQTT 订阅控制 Topic，接收电脑发布的 LED 指令，控制板载 LED，并把当前状态以 JSON 格式发布回电脑。它不是单纯的“点灯 Demo”，而是一次从能跑到工程化的演进：WiFi、MQTT、设备业务、GPIO、命令解析、配置参数被拆成清晰边界，为后续迁移到自建服务器或阿里云物模型做准备。

> [!tip] 学习主线
> 本任务的核心问题不是“怎么让 LED 亮”，而是：**一个物联网设备如何进入局域网、连接消息中转站、接收远端控制、执行本地动作、再把状态回传出去？**

---

## 1. 任务目标

本阶段目标是完成本地闭环：

```text
电脑 MQTT 客户端
  -> 本机 Mosquitto Broker
  -> ESP32 MQTT Client
  -> device_app 设备业务层
  -> led_app GPIO 驱动层
  -> ESP32 LED 状态变化
  -> MQTT 状态回传
```

这个闭环至少包含三条链路：

| 链路 | 说明 |
| --- | --- |
| 网络链路 | ESP32 连接手机热点/局域网，拿到 IP |
| 消息链路 | ESP32 连接电脑上的 Mosquitto Broker |
| 业务链路 | 电脑发布 LED 指令，ESP32 执行动作并回传状态 |

第一阶段不做 TLS、不做账号密码、不做云平台物模型，目的是先把 MQTT 的发布/订阅模型、Topic/Payload、事件回调和设备控制闭环跑通。

---

## 2. 系统链路

### 2.1 本地网络结构

```text
手机热点 / 局域网
├── 电脑
│   ├── Mosquitto Broker: 1883
│   ├── mosquitto_sub
│   └── mosquitto_pub
└── ESP32
    ├── WiFi STA
    ├── MQTT Client
    └── LED GPIO
```

电脑在这个阶段承担两个角色：

1. Broker：运行 Mosquitto，作为 MQTT 消息中转站。
2. 调试客户端：用 `mosquitto_pub` / `mosquitto_sub` 发布和观察消息。

ESP32 不直接连接电脑上的某个应用程序，而是连接 Broker。电脑发布消息到 Broker，Broker 再把消息转发给订阅了对应 Topic 的 ESP32。

### 2.2 Topic 设计

当前 Topic 设计保持简单：

| Topic | 方向 | 含义 |
| --- | --- | --- |
| `esp32/status` | ESP32 -> 电脑 | 设备在线状态 |
| `esp32/led/set` | 电脑 -> ESP32 | 设置 LED 状态 |
| `esp32/led/state` | ESP32 -> 电脑 | LED 当前状态回传 |

Payload 已经从纯字符串升级到 JSON：

```json
{"led":"ON"}
```

状态回传示例：

```text
esp32/led/state {"led":"ON"}
esp32/led/state {"led":"OFF"}
```

控制命令当前兼容两类格式：

```text
ON
OFF
1
0
{"led":"ON"}
{"led":"OFF"}
{"led":true}
{"led":false}
{"led":1}
{"led":0}
```

---

## 3. 环境搭建与验证命令

### 3.1 Mosquitto 本地配置

Mosquitto 安装路径：

```text
D:\TOOL\Mosquitto
```

本地测试阶段推荐监听所有网卡：

```conf
listener 1883 0.0.0.0
allow_anonymous true
persistence false
log_type error
log_type warning
log_type notice
log_type information
connection_messages true
```

这样 ESP32 可以通过电脑局域网 IP 访问 Broker，而不是只能访问 `127.0.0.1`。

### 3.2 常用命令

启动 Mosquitto：

```powershell
D:\TOOL\Mosquitto\mosquitto.exe -c D:\TOOL\Mosquitto\mosquitto-iot-test.conf -v
```

订阅所有 ESP32 消息：

```powershell
D:\TOOL\Mosquitto\mosquitto_sub.exe -h 电脑局域网IP -t esp32/# -v
```

发布 LED 打开命令：

```powershell
D:\TOOL\Mosquitto\mosquitto_pub.exe -h 电脑局域网IP -t esp32/led/set -m "{\"led\":\"ON\"}"
```

发布 LED 关闭命令：

```powershell
D:\TOOL\Mosquitto\mosquitto_pub.exe -h 电脑局域网IP -t esp32/led/set -m "{\"led\":false}"
```

烧录和串口监视：

```powershell
idf.py -p COM4 flash monitor
```

> [!warning] COM 口不是固定的
> 之前出现过 `COM3` 不存在或被占用，后来实际使用的是 `COM4`。烧录失败时先确认设备管理器里的端口号，再检查串口监视器是否占用了端口。

---

## 4. 工程架构演进

这个任务一开始可以写成单文件 Demo，但为了工程训练，最终拆成了多个模块。

```text
IOT-Test.c
├── app_config.h
├── wifi_app
├── mqtt_app
├── command_parser
├── device_app
└── led_app
```

### 4.1 main 只负责编排

`app_main()` 不直接处理 WiFi 细节、MQTT 细节、GPIO 细节，只负责启动顺序：

```c
void app_main(void)
{
    ESP_LOGI(TAG, "app_main start");

    ESP_ERROR_CHECK(command_parser_self_test() == 0 ? ESP_OK : ESP_FAIL);
    ESP_ERROR_CHECK(storage_init());
    ESP_ERROR_CHECK(led_app_init());
    ESP_ERROR_CHECK(device_app_init());
    ESP_ERROR_CHECK(wifi_app_start());

    ESP_LOGI(TAG, "waiting for wifi ip...");
    ESP_ERROR_CHECK(wifi_app_wait_connected(pdMS_TO_TICKS(APP_WIFI_CONNECT_TIMEOUT_MS)));

    ESP_LOGI(TAG, "wifi ready, starting mqtt...");
    ESP_ERROR_CHECK(mqtt_app_start());
}
```

这个结构的好处是入口函数非常清楚：

```text
初始化存储
初始化硬件
初始化设备状态
连接 WiFi
等待 IP
启动 MQTT
```

### 4.2 模块职责边界

| 模块 | 职责 |
| --- | --- |
| `app_config.h` | 统一管理 WiFi、MQTT、GPIO、Topic 等配置 |
| `wifi_app` | WiFi STA 初始化、事件处理、连接等待、失败重试 |
| `mqtt_app` | MQTT 连接、订阅、发布、事件回调 |
| `command_parser` | 把 Payload 解析成业务命令 |
| `device_app` | 维护设备业务状态，调用底层驱动 |
| `led_app` | 只负责 GPIO 输出 |

最重要的边界是：

```text
mqtt_app 不直接操作 GPIO
led_app 不知道 MQTT 存在
command_parser 不执行硬件动作
device_app 才负责把业务命令变成设备状态变化
```

这就是从 Demo 走向工程项目的关键一步。

---

## 5. 关键知识点

### 5.1 WiFi 不是同步函数，而是事件系统

ESP32 WiFi 连接不是“调用一个函数然后立刻得到结果”，而是：

```text
esp_wifi_start()
  -> WIFI_EVENT_STA_START
  -> esp_wifi_connect()
  -> IP_EVENT_STA_GOT_IP
  -> 设置 WIFI_CONNECTED_BIT
```

因此 `wifi_app` 使用 FreeRTOS EventGroup，把异步事件转换成主流程可以等待的同步结果。

工程化后增加了：

```text
WIFI_CONNECTED_BIT：连接成功
WIFI_FAIL_BIT：连接失败
APP_WIFI_MAX_RETRY：最大重试次数
APP_WIFI_CONNECT_TIMEOUT_MS：等待超时时间
```

这避免了 WiFi 密码错误或热点关闭时程序无限卡死。

### 5.2 MQTT 的核心是发布/订阅

MQTT 不要求发送方知道接收方是谁，只需要约定 Topic：

```text
电脑 publish: esp32/led/set
ESP32 subscribe: esp32/led/set
```

ESP32 状态回传也是同理：

```text
ESP32 publish: esp32/led/state
电脑 subscribe: esp32/#
```

这个模式比“电脑直接调用 ESP32 函数”更适合物联网，因为设备和控制端可以解耦。

### 5.3 Payload 要逐步结构化

第一阶段使用：

```text
ON
OFF
```

后续升级成：

```json
{"led":"ON"}
```

JSON 的意义不是为了看起来高级，而是为了后续扩展：

```json
{"led":"ON","brightness":80,"source":"mqtt"}
```

当前项目使用受控小解析器，只解析约定的 LED JSON 命令。真正接阿里云物模型或复杂业务时，应该引入成熟 JSON 库。

### 5.4 配置集中化

当前配置集中在 `app_config.h`：

```c
#define APP_WIFI_MAX_RETRY           5
#define APP_WIFI_CONNECT_TIMEOUT_MS  15000
#define APP_MQTT_BROKER_URI          "mqtt://10.172.24.56"
#define APP_MQTT_QOS                 1
#define APP_MQTT_RETAIN              0
#define APP_TOPIC_LED_SET            "esp32/led/set"
#define APP_TOPIC_LED_STATE          "esp32/led/state"
```

工程原则：

```text
模块负责行为
配置负责参数
不要把可变参数散落在业务代码里
```

以后从手机热点迁移到自建 MQTT 服务器时，优先改配置文件，而不是全工程搜索 IP、Topic、QoS。

### 5.5 错误返回链

模块接口从 `void` 改成 `esp_err_t` 后，错误可以向上传递：

```text
led_app_init -> device_app_init -> wifi_app_start -> mqtt_app_start -> app_main
```

底层模块不直接决定系统是否崩溃，而是把错误返回给上层。当前 `app_main()` 使用 `ESP_ERROR_CHECK()` 统一处理，后续可以替换成重试、降级或进入离线模式。

---

## 6. 踩坑与排错记录

### 6.1 校园网隔离问题

校园网可能存在客户端隔离，导致电脑和 ESP32 虽然都联网，但互相访问不到。本任务中使用手机热点更稳定，因为电脑和 ESP32 在同一个简单局域网内。

排查思路：

```text
电脑 IP 是否正确
ESP32 是否拿到同网段 IP
Broker 是否监听 0.0.0.0:1883
防火墙是否阻止 1883 端口
```

### 6.2 Mosquitto 只能本机访问

如果 Mosquitto 只监听 localhost，ESP32 无法访问。配置中要使用：

```conf
listener 1883 0.0.0.0
```

电脑端测试也建议显式指定局域网 IP：

```powershell
mosquitto_sub.exe -h 电脑局域网IP -t esp32/# -v
```

### 6.3 下载模式错误

烧录时出现：

```text
Wrong boot mode detected
The chip needs to be in download mode
```

处理方式：按住 BOOT，再执行 flash，看到连接后松开。不同开发板自动下载电路不一定稳定。

### 6.4 COM 口错误或被占用

典型错误：

```text
Could not open COM3, the port is busy or doesn't exist
```

排查顺序：

```text
设备管理器确认端口号
关闭其他串口监视器
确认 USB 线支持数据传输
改用 idf.py -p COM4 flash monitor
```

### 6.5 MQTT 组件依赖

ESP-IDF v6 中 MQTT 通过 managed component 引入，`idf_component.yml` 中需要：

```yaml
dependencies:
  espressif/mqtt: "^1.0.0"
```

`CMakeLists.txt` 中需要把 `mqtt` 加入依赖。

### 6.6 JSON 组件不可用

尝试直接依赖 `json` 组件时，ESP-IDF v6 环境提示组件不存在。因此当前版本没有引入完整 JSON 库，而是实现了受控小解析器，只解析本任务约定的 LED 命令。

这个选择适合 Task1，但不是复杂产品的长期方案。

---

## 7. 当前闭环验证清单

电脑订阅：

```powershell
D:\TOOL\Mosquitto\mosquitto_sub.exe -h 电脑局域网IP -t esp32/# -v
```

ESP32 上线后应看到：

```text
esp32/status online
esp32/led/state {"led":"OFF"}
```

发布打开命令：

```powershell
D:\TOOL\Mosquitto\mosquitto_pub.exe -h 电脑局域网IP -t esp32/led/set -m "{\"led\":\"ON\"}"
```

预期回传：

```text
esp32/led/state {"led":"ON"}
```

发布关闭命令：

```powershell
D:\TOOL\Mosquitto\mosquitto_pub.exe -h 电脑局域网IP -t esp32/led/set -m "{\"led\":false}"
```

预期回传：

```text
esp32/led/state {"led":"OFF"}
```

---

## 8. 后续演进方向

Task1 的价值是建立本地闭环。后面可以沿着四条路线继续升级：

| 方向 | 目标 |
| --- | --- |
| 自建 MQTT 服务器 | 把 Broker 从电脑迁移到云服务器或局域网服务器 |
| 账号密码 | 给 Mosquitto 增加认证，理解 MQTT 安全边界 |
| TLS | 使用 `mqtts://` 和证书，理解加密链路 |
| 阿里云物模型 | 把 Topic 和 JSON 格式迁移到云平台规范 |
| NVS 配置化 | WiFi、Broker、DeviceName 等参数不再硬编码 |

> [!summary] 本任务结论
> 本地电脑控制 LED 不是最终目标，它只是一个入口。真正学到的是 IoT 设备的最小工程闭环：网络接入、消息中转、命令解析、业务执行、状态上报、异常处理和模块边界。只要这个闭环清楚，后面换成继电器、电机、传感器、阿里云、自建服务器，本质都只是替换 Topic、Payload 和业务层逻辑。
