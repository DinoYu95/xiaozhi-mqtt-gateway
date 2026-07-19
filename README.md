# MQTT+UDP 到 WebSocket 桥接服务

## 项目概述

本项目是基于虾哥开源的 [MQTT+UDP 到 WebSocket 桥接服务](https://github.com/78/xiaozhi-mqtt-gateway)，进行了修改，以适应[xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server)。

## 部署使用
部署使用请[参考这里](https://github.com/xinnan-tech/xiaozhi-esp32-server/blob/main/docs/mqtt-gateway-integration.md)。

## 设备管理接口说明

### 接口认证

API请求需要在请求头中包含有效的`Authorization: Bearer xxx`令牌，令牌生成规则如下：

1. 获取当前日期，格式为`yyyy-MM-dd`（例如：2025-09-15）
2. 获取.env文件中配置的`MQTT_SIGNATURE_KEY`值
3. 将日期字符串与MQTT_SIGNATURE_KEY连接（格式：`日期+MQTT_SIGNATURE_KEY`）
4. 对连接后的字符串进行SHA256哈希计算
5. 哈希结果即为当日有效的Bearer令牌

**注意**：服务启动时会自动计算并打印当日的临时密钥，方便测试使用。


### 接口1 设备指令下发API，支持MCP指令并返回设备响应
``` shell
curl --location --request POST 'http://localhost:8007/api/commands/lichuang-dev@@@a0_85_e3_f4_49_34@@@aeebef32-f0ef-4bce-9d8a-894d91bc6932' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer your_daily_token' \
--data-raw '{"type": "mcp", "payload": {"jsonrpc": "2.0", "id": 1, "method": "tools/call", "params": {"name": "self.get_device_status", "arguments": {}}}}'
```

### 接口1b 家长远程看娃 parent_snapshot（MQTT 直推设备，异步 HTTP 回图）

与 MCP 共用 `POST /api/commands/{clientId}`，无需 WS bridge。设备需固件支持 `type: parent_snapshot`。

``` shell
curl --location --request POST 'http://localhost:8007/api/commands/GID_default@@@11_22_33_44_55_66@@@11_22_33_44_55_66' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer your_daily_token' \
--data-raw '{
  "type": "parent_snapshot",
  "payload": {
    "requestId": "snap_demo001",
    "uploadUrl": "https://your-manager-api/parent-api/chat/snapshot/device-upload",
    "uploadToken": "one_time_token_from_prepare",
    "maxWidth": 640,
    "jpegQuality": 80
  }
}'
```

网关经 MQTT 向 `devices/p2p/{mac}` 下发同结构 JSON；设备拍照后 POST `uploadUrl`。

### 接口2 设备状态查询API，支持查询设备是否在线

``` shell
curl --location --request POST 'http://localhost:8007/api/devices/status' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer your_daily_token' \
--data-raw '{
    "clientIds": [
        "lichuang-dev@@@a0_85_e3_f4_49_34@@@aeebef32-f0ef-4bce-9d8a-894d91bc6932"
    ]
}'
```

返回字段：`isAlive` = MQTT 长连接在线（空闲无对话时也应为 true）；`bridgeAlive` = 是否正在 WS 会话中；`exists` = 网关是否见过该 clientId 连接。

Provide Chinese summary for user.