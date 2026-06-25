ESP32-S3
  -> 上传 JPEG
上位机 / Docker 服务
  -> 接收 JPEG
  -> AI / OpenCV 检测
  -> 返回 JSON
ESP32-S3
  -> 根据 target 控制 L1/L2...

HTTP 上传成功
  -> Python 能收到 JPEG
  -> Python 能解码 JPEG
  -> Python 返回模拟结果
  -> ESP32-S3 解析 JSON
  -> ESP32-S3 控制灯
  -> 最后替换成真正 AI


推荐第一版：

`ESP32-S3 -> HTTP POST image/jpeg Python/FastAPI 上位机服务 -> 接收图片 -> 返回 {"target": false}`

EMQX 后面再加：

`AI 服务 -> 把检测结果 publish 到 EMQX PC 面板 / 日志工具 -> 订阅检测结果`

**为什么要上传云端**  
因为 AI 检测是重任务。ESP32-S3 做 USB Host、Wi-Fi、JPEG 缓存、灯控已经够忙了；让它跑复杂视觉模型不划算。上传的目的就是把图片交给上位机：



state: init -> auth
state: auth -> assoc
state: assoc -> run
connected with REDMI K80 Pro
security: WPA2-PSK
rssi: -31


内部用 ESP-IDF 的 esp_http_client：

`esp_http_client_init() esp_http_client_set_method(HTTP_METHOD_POST) esp_http_client_set_header("Content-Type", "application/json") esp_http_client_set_post_field() esp_http_client_perform() esp_http_client_get_status_code() esp_http_client_read_response() esp_http_client_cleanup()`

**你要特别注意网络拓扑**  
你现在 ESP 连的是 REDMI K80 Pro 热点，ESP 的 IP 是：

`10.172.24.66`

如果你的 Docker/HTTP 服务在电脑上，电脑也必须和 ESP 在同一个网络里。也就是说电脑也要连这个热点，或者 ESP 和电脑在同一个路由器下。否则 ESP32-S3 找不到电脑。

电脑端服务要监听：

`0.0.0.0:8000`

不要只监听：

`127.0.0.1:8000`

因为 127.0.0.1 只允许电脑自己访问，ESP 访问不到。

**验收标准**  
ESP monitor 里看到类似：

`app_state: current stage: HTTP_TEST wifi_sta: got ip: 10.172.24.66 cloud_http: POST http://电脑IP:8000/api/test cloud_http: status=200 cloud_http: response: {"ok":true,"message":"received"} app_state: HTTP test stage passed`



ESP32-S3
  -> 打开 UVC
  -> 获取并缓存一帧 JPEG
  -> 关闭 UVC stream
  -> Wi-Fi 连接
  -> HTTP POST JPEG
电脑 HTTP 服务
  -> 收到 image/jpeg
  -> 打印图片长度
  -> 可选保存 test.jpg
  -> 返回 {"ok": true, "target": false}
ESP32-S3
  -> 打印 status 和 response


**HTTP 配置**  
project_config.h 里新增：

`#define APP_HTTP_JPEG_UPLOAD_URL "http://10.172.24.56:8000/api/upload"`

保留原来的：

`APP_HTTP_TEST_URL`

这样小 JSON 测试和 JPEG 上传测试互不干扰。

**HTTP header**  
上传 JPEG 时设置：

`Content-Type: image/jpeg`

可以额外加几个调试 header：

`X-Frame-Seq: 291 X-Timestamp-Ms: 11446`

这不是必须，但很适合调试。

**服务器端预期**  
电脑 Python 服务新增 /api/upload：

`POST /api/upload Content-Type: image/jpeg body: JPEG bytes`

服务器做三件事：

`1. 打印 len 2. 检查开头 FF D8、结尾 FF D9 3. 保存为 latest.jpg`


现在先做周期性单帧上传，等它稳定后，再升级成“连续采集 + 抽帧上传”。这是嵌入式里很稳的爬坡方式。


案 B：连续采集后台任务 + 主任务定时上传，推荐

`UVC task: 持续收帧 持续更新 latest JPEG cache Upload loop: 每 5 秒复制 latest JPEG HTTP 上传 打印 seq / len / result`

优点是架构更接近最终项目，后面接 AI、LCD、灯控都顺。缺点是要引入 FreeRTOS task，复杂一点点。

**推荐设计**

我们采用方案 B。

新增阶段可以叫：


这里的关键思想叫：**生产者/消费者模型**。

- 生产者：摄像头任务，不断生产 JPEG，写入 latest cache。
- 消费者：上传逻辑，每隔一段时间取 latest cache 上传。
- 共享区：camera_uvc_client 内部的 latest JPEG cache。
- 保护机制：你已经有 s_latest_mutex，所以读写 latest cache 是线程安全的。

后面我们应该改 cloud_http_client_post_latest_jpeg()：

`允许 copy 后拿到更新的一帧 用 copied_len / copied_sequence / copied_timestamp_ms 作为最终上传信息 重新检查 copied JPEG 的 FF D8 和 FF D9 然后上传`



**方案对比**

方案 A：HTTP 端允许 seq 变化

`先 get meta 再 copy 如果 copy 后 seq 变了，也接受 copy 后的新 seq`

优点：改动小。  
缺点：逻辑有点绕，HTTP 端还要重新校验 JPEG SOI/EOI，容易把 camera cache 的职责泄漏到 HTTP 模块。

方案 B：camera 模块提供“一次性快照接口”，推荐

`camera_uvc_copy_latest_jpeg() 在 mutex 内同时复制 data / len / seq / timestamp 返回一个自洽快照`

HTTP 模块只相信 copy 函数返回的结果，不再先读 meta 再要求一致。

优点：职责清楚。  
缺点：要稍微调整 HTTP 上传逻辑。

方案 C：上传期间暂停采集

`停止 UVC 更新 cache 复制 JPEG 上传完成再恢复`

不推荐。它会破坏连续采集意义，而且 HTTP 上传可能很慢，会让摄像头链路频繁停顿。



malloc 96 KB
camera_uvc_copy_latest_jpeg(buffer, 96 KB, &len, &seq, &timestamp)
检查 len > 0
检查 buffer[0..1] == FF D8
检查是否存在 FF D9
POST buffer[0..len]
free buffer