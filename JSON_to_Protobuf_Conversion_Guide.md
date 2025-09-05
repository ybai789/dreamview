# JSON 转 Protobuf 转换指南

## 概述

本文档总结了在 Apollo Dreamview Plus 项目中将 JSON 格式消息转换为 Protobuf 消息的方法，以 obstacle 数据为例。

## 核心转换函数

### 1. JsonStringToMessage

```cpp
#include "google/protobuf/util/json_util.h"

// 函数签名
google::protobuf::util::Status JsonStringToMessage(
    const std::string& json_string, 
    google::protobuf::Message* message
);
```

**功能**: 将 JSON 字符串直接转换为 Protobuf 消息

**返回值**: `google::protobuf::util::Status` 对象，用于检查转换是否成功

## 消息定义

### 1. PerceptionObstacles 定义

**位置**: `modules/common_msgs/perception_msgs/perception_obstacle.proto`

```protobuf
message PerceptionObstacles {
  repeated PerceptionObstacle perception_obstacle = 1;  // 障碍物数组
  optional apollo.common.Header header = 2;             // 消息头
  optional apollo.common.ErrorCode error_code = 3 [default = OK];
  optional LaneMarkers lane_marker = 4;                 // 车道标记
  optional CIPVInfo cipv_info = 5;                      // 最近路径车辆信息
  repeated PerceptionWaste perception_waste = 6;        // 垃圾检测数组
}
```

### 2. StreamData 定义

**位置**: `modules/dreamview_plus/proto/data_handler.proto`

```protobuf
message StreamData {
  optional string type = 1;        // 数据类型，如 "obstacle"
  optional string action = 2;      // 操作类型，如 "stream"
  optional string data_name = 3;   // 数据名称，如 "obstacle"
  optional string channel_name = 4; // 频道名称
  optional bytes data = 5;         // 实际的数据内容（二进制）
}
```

## 转换实现

### 1. 基本转换示例

```cpp
#include "google/protobuf/util/json_util.h"
#include "modules/common_msgs/perception_msgs/perception_obstacle.pb.h"
#include "modules/dreamview_plus/proto/data_handler.pb.h"

// JSON 格式的障碍物数据
std::string json_obstacle_data = R"({
  "perception_obstacle": [
    {
      "id": 12345,
      "position": {
        "x": 10.5,
        "y": 20.3,
        "z": 0.0
      },
      "theta": 1.57,
      "velocity": {
        "x": 15.2,
        "y": 0.0,
        "z": 0.0
      },
      "length": 4.5,
      "width": 2.0,
      "height": 1.8,
      "timestamp": 1234567890.123,
      "confidence": 0.95,
      "type": "VEHICLE",
      "sub_type": "ST_CAR"
    }
  ],
  "header": {
    "timestamp_sec": 1234567890.123,
    "module_name": "perception",
    "sequence_num": 1
  }
})";

// 转换为 PerceptionObstacles Protobuf 消息
PerceptionObstacles obstacles_proto;
google::protobuf::util::Status status = 
    JsonStringToMessage(json_obstacle_data, &obstacles_proto);

if (!status.ok()) {
  AERROR << "Failed to convert JSON to Protobuf: " << status.message();
  return;
}

// 现在 obstacles_proto 就是转换后的 Protobuf 消息
```

### 2. 在 Updater 中的完整实现

```cpp
void ObstacleUpdater::PublishMessage(const std::string &channel_name) {
  // 1. 获取 JSON 格式的障碍物数据
  std::string json_obstacle_data = GetObstacleJsonData();
  
  // 2. 转换为 Protobuf 消息
  PerceptionObstacles obstacles_proto;
  google::protobuf::util::Status status = 
      JsonStringToMessage(json_obstacle_data, &obstacles_proto);
  
  if (!status.ok()) {
    AERROR << "Failed to convert JSON to Protobuf: " << status.message();
    return;
  }
  
  // 3. 使用现有的处理逻辑
  ObstacleChannelUpdater* channel_updater = GetChannelUpdater(channel_name);
  if (channel_updater) {
    channel_updater->UpdateData(obstacles_proto);
  }
  
  // 4. 封装为 StreamData 并发送
  StreamData stream_data;
  stream_data.set_action("stream");
  stream_data.set_data_name("obstacle");
  stream_data.set_type("obstacle");
  stream_data.set_channel_name("/apollo/perception/obstacles");
  
  // 将 Protobuf 消息序列化为二进制数据
  std::string obstacles_string;
  obstacles_proto.SerializeToString(&obstacles_string);
  std::vector<uint8_t> byte_data(obstacles_string.begin(), obstacles_string.end());
  stream_data.set_data(&(byte_data[0]), byte_data.size());
  
  // 发送数据
  std::string stream_data_string;
  stream_data.SerializeToString(&stream_data_string);
  websocket_->BroadcastBinaryData(stream_data_string);
}
```

## 错误处理

### 1. 状态检查

```cpp
google::protobuf::util::Status status = 
    JsonStringToMessage(json_data, &protobuf_message);

if (!status.ok()) {
  AERROR << "JSON to Protobuf conversion failed: " << status.message();
  // 处理错误情况
  return;
}
```

### 2. 常见错误类型

- **JSON 格式错误**: JSON 字符串格式不正确
- **字段类型不匹配**: JSON 中的字段类型与 Protobuf 定义不匹配
- **必需字段缺失**: 缺少 Protobuf 中定义的必需字段
- **枚举值无效**: JSON 中的枚举值在 Protobuf 中不存在

## 相关工具函数

### 1. JsonUtil 工具类

```cpp
#include "modules/common/util/json_util.h"

// JSON 路径解析
std::string request_id;
if (!JsonUtil::GetStringByPath(json, "data.requestId", &request_id)) {
  // 处理错误
}

// 数值提取
double x, y;
if (!JsonUtil::GetNumber(point, "x", &x) || 
    !JsonUtil::GetNumber(point, "y", &y)) {
  // 处理错误
}

// Protobuf 转 JSON
Json json_data = JsonUtil::ProtoToJson(protobuf_message);
```

### 2. 双向转换

```cpp
// JSON → Protobuf
google::protobuf::util::Status JsonStringToMessage(
    const std::string& json_string, 
    google::protobuf::Message* message
);

// Protobuf → JSON
google::protobuf::util::Status MessageToJsonString(
    const google::protobuf::Message& message, 
    std::string* json_string
);
```

## 最佳实践

### 1. 错误处理

- 始终检查 `JsonStringToMessage` 的返回值
- 提供有意义的错误信息
- 在转换失败时优雅地处理

### 2. 性能考虑

- 对于频繁转换的场景，考虑缓存转换结果
- 避免在循环中进行重复的 JSON 解析
- 使用适当的数据类型避免不必要的转换

### 3. 代码组织

- 将转换逻辑封装在单独的函数中
- 使用统一的错误处理模式
- 保持代码的可读性和可维护性

## 示例：完整的转换流程

```cpp
class JsonToProtobufConverter {
public:
  static bool ConvertObstacleJsonToProtobuf(
      const std::string& json_data, 
      PerceptionObstacles* obstacles) {
    
    google::protobuf::util::Status status = 
        JsonStringToMessage(json_data, obstacles);
    
    if (!status.ok()) {
      AERROR << "Failed to convert obstacle JSON to Protobuf: " 
             << status.message();
      return false;
    }
    
    return true;
  }
  
  static bool ConvertAndSendObstacleData(
      const std::string& json_data,
      WebSocketHandler* websocket) {
    
    PerceptionObstacles obstacles;
    if (!ConvertObstacleJsonToProtobuf(json_data, &obstacles)) {
      return false;
    }
    
    // 封装为 StreamData
    StreamData stream_data;
    stream_data.set_action("stream");
    stream_data.set_data_name("obstacle");
    stream_data.set_type("obstacle");
    
    std::string obstacles_string;
    obstacles.SerializeToString(&obstacles_string);
    std::vector<uint8_t> byte_data(obstacles_string.begin(), obstacles_string.end());
    stream_data.set_data(&(byte_data[0]), byte_data.size());
    
    // 发送数据
    std::string stream_data_string;
    stream_data.SerializeToString(&stream_data_string);
    websocket->BroadcastBinaryData(stream_data_string);
    
    return true;
  }
};
```

## 总结

使用 `JsonStringToMessage` 函数是将 JSON 格式消息转换为 Protobuf 消息的最直接、最简单的方法。这种方法：

1. **简单易用**: 一行代码完成转换
2. **类型安全**: 自动进行类型检查和转换
3. **错误处理**: 提供详细的错误信息
4. **无需额外依赖**: 使用 Google Protobuf 自带的工具

通过这种方式，可以轻松地将 JSON 格式的障碍物数据（或其他数据）转换为标准的 Protobuf 消息格式，并在 Dreamview Plus 系统中使用。
