

设备属性上报
周期任务
JSON 数据建模
MQTT Topic 规范
设备在线状态
EMQX Dashboard 调试
后端数据流


连接管理：ESP32 是否连上 EMQX
Topic 调试：telemetry 是否到达
认证机制：后面加用户名密码
规则引擎：后面把数据转发到数据库
数据桥接：后面接 InfluxDB/Grafana


连接 Broker
发布 status
发布 telemetry
处理 MQTT 事件


MQTT 偶尔断线是运行期常态
不应该因为一次 publish 失败就让整个程序 abort


APP_MQTT_BROKER_URI "mqtt://192.168.x.x:1883"
如果 EMQX 是 Docker 跑在电脑本机上，ESP32 要连接的是：

text


电脑局域网 IP:1883
不是：

text


localhost
127.0.0.1
Docker 容器 IP


telemetry_app 生成数据
mqtt_app 发送数据
main 负责调度流程
EMQX 负责接收和转发消息


电脑/EMQX 发布：
  topic: esp32/device001/property/set
  payload: {"period_ms":10000}

ESP32 收到后：
  解析 period_ms
  修改 telemetry 上报周期
  后续从 5 秒上报一次变成 10 秒上报一次
