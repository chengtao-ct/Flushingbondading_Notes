


WiFi 层：我连上路由器了吗？
IP 层：我拿到 IP 了吗？
时间层：我知道当前时间了吗？
TLS 层：证书验证能过吗？
MQTT 层：我能连上 Broker 吗？
业务层：Topic 和 JSON 对吗？



设备上云，不是 WiFi 连上就能上云。
至少还需要：
1. IP 可用
2. DNS 可用
3. 时间可用
4. 证书/密钥可用
5. Broker 地址可达


阿里云 ProductKey       类似平台上的产品/项目编号
阿里云 DeviceName       类似 MQTT client_id 或设备 ID
阿里云 DeviceSecret     类似 MQTT password，但它不是直接原文发送，而是用于签名



1. 生成 MQTT 连接配置
2. 创建 MQTT client
3. 注册 MQTT 事件回调
4. 启动 MQTT client


MQTT 连接 = Broker地址 + 端口 + client_id + username + password + 证书 + 事件回调


MQTT 层负责收消息
JSON 层负责解析
业务层负责执行动作


阿里云：
/sys/ProductKey/DeviceName/thing/service/property/set

自建 Mosquitto：
esp32/device001/property/set


用户按了实体开关
设备上报开关状态

传感器温度变化
设备上报温度

电机故障
设备上报故障码

门锁被打开
设备上报开锁事件



区别只是 Topic 和 JSON 格式不同。

阿里云是：

`/sys/ProductKey/DeviceName/thing/event/property/post`

你自己的 Mosquitto 可以设计成：

`esp32/device001/property/post`



ALIOT_DM_POST
属性上报

ALIOT_DM_SET_ACK
属性设置回复

ALIOT_DM_EVENT
事件上报

ALIOT_OTA_VERSION
OTA 版本上报

ALIOT_OTA_PROGRESS
OTA 进度上报


aliot_dm.c 的本质不是 MQTT。
它只负责把设备属性/事件/OTA 信息，组装成阿里云规定的 JSON payload。


ProductKey
产品标识，表示这一类设备属于哪个阿里云产品。

DeviceName
设备名称，表示某一台具体设备。

DeviceSecret
设备密钥，表示这台设备的身份密码。

ProductSecret
产品密钥，通常用于动态注册，不建议随便放进量产固件。

RegionId
阿里云地域，比如 cn-shanghai。


先从 NVS 里读 DeviceSecret。
如果读不到，说明还没动态注册过。
后面 aliot_mqtt_run() 会调用 aliot_start_register() 去动态注册。


一机一密：
ProductKey + DeviceName + DeviceSecret 直接烧进固件或配置区。

一型一密动态注册：
固件里有 ProductKey + ProductSecret + DeviceName。
第一次上云时换取 DeviceSecret。
拿到后保存到 NVS。
以后用 DeviceSecret 正常登录。


读取 DeviceSecret
  |
  v
拼接 sign_content
  |
  v
HMAC_MD5(DeviceSecret, sign_content)
  |
  v
转成十六进制字符串
  |
  v
作为 MQTT password
  |
  v
连接阿里云 Broker


设备启动
  |
  v
检查 NVS 里有没有 DeviceSecret
  |
  v
没有 DeviceSecret
  |
  v
使用 ProductSecret 生成动态注册 password
  |
  v
以 authType=register 方式连接阿里云 MQTT
  |
  v
等待 /ext/register 返回 deviceSecret
  |
  v
保存 deviceSecret 到 NVS
  |
  v
断开注册连接
  |
  v
再走普通 MQTT 登录


阿里云 MQTT 通知有新固件
  |
  v
ESP32 收到固件下载 URL
  |
  v
ESP32 通过 HTTPS 下载固件并写入 OTA 分区
  |
  v
重启运行新固件


nvs,      data, nvs,     0x9000,  0x4000,
otadata,  data, ota,     0xd000,  0x2000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  app, factory,  , 1500K,
ota_0,    app, ota_0,    , 1500K,
ota_1,    app, ota_1,    , 1500K,



方向        Topic              Payload        作用
电脑->设备  esp32/led/set      ON/OFF         控制 LED
设备->电脑  esp32/led/state    ON/OFF         回报 LED 状态
设备->电脑  esp32/status       online         告诉电脑设备上线


程序启动
  |
  v
打印 app_main start
  |
  v
初始化 NVS
  |
  v
初始化 LED GPIO
  |
  v
启动 WiFi
  |
  v
等待 WiFi 获取 IP
  |
  v
启动 MQTT



WiFi 启动 != 网络可用
拿到 IP 才接近网络可用



wifi_event_handler()
  |
  | 拿到 IP 后 set bit
  v
app_main()
  |
  | wait bit 被唤醒
  v
mqtt_start()


回调函数负责发现状态变化
事件组负责通知主流程继续往下走



app_main()
  |
  v
等待 WiFi 获取 IP
  |
  v
mqtt_start()
  |
  v
配置 Broker URI
  |
  v
创建 MQTT client
  |
  v
注册 MQTT 事件回调
  |
  v
启动 MQTT client
  |
  v
等待 MQTT_EVENT_CONNECTED



第一个重点事件：连接成功。

`case MQTT_EVENT_CONNECTED: ESP_LOGI(TAG, "mqtt connected"); s_mqtt_connected = true; esp_mqtt_client_subscribe_single(s_mqtt_client, TOPIC_LED_SET, 1); esp_mqtt_client_publish(s_mqtt_client, TOPIC_STATUS, "online", 0, 1, 0); mqtt_publish_led_state(); break;`

MQTT 连接成功后，我们做了 4 件事：

`1. 标记 MQTT 已连接 2. 订阅 LED 控制 Topic 3. 发布设备在线状态 4. 发布当前 LED 状态`



esp_mqtt_client_publish(
    s_mqtt_client,
    TOPIC_STATUS,
    "online",
    0,
    1,
    0
);
含义是：

text


s_mqtt_client   MQTT 客户端句柄
TOPIC_STATUS    发布到哪个 Topic
"online"        消息内容
0               payload 长度，0 表示自动 strlen
1               QoS 1
0               retain 关闭
retain 先简单理解为：

text


是否让 Broker 保存最后一条消息



mqtt_start()
  |
  v
连接 Broker
  |
  v
MQTT_EVENT_CONNECTED
  |
  v
订阅 esp32/led/set
发布 esp32/status = online
发布 esp32/led/state
  |
  v
电脑发布 ON/OFF
  |
  v
MQTT_EVENT_DATA
  |
  v
判断 Topic 是否为 esp32/led/set
  |
  v
mqtt_handle_led_command()



WiFi 模块解决“设备进入局域网”
MQTT 模块解决“设备和外部世界交换消息”
LED 模块解决“本地硬件动作”
main 只负责“按顺序启动系统”