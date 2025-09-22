# Frontend WelcomeGuide 改变 Mode 的 WebSocket 通信流程 - 完整问答文档

## 问题1: Frontend HMIStatus 通过什么 WebSocket 和后端通信？

### 答案：
Frontend HMIStatus 通过 **`mainConnection`** WebSocket 与后端通信。

**具体配置：**
```typescript
// 前端配置
export const config = {
    mainUrl: `${baseURL}/websocket`,  // HMIStatus 端点
    pluginUrl: `${baseURL}/plugin`,   // 前端请求和插件
};
```

**WebSocket 连接：**
- **连接名称**：`mainConnection`
- **端点**：`/websocket`
- **用途**：HMIStatus 和核心数据流
- **数据格式**：JSON

**订阅方式：**
```typescript
// 订阅 HMIStatus 数据流
streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data) => {
    dispatch(updateStatus({
        ...(data as any),
        frontendIsFromHmi: true,
    } as any));
});
```

---

## 问题2: 这是和前端 request 用一个 WebSocket 吗？还是有独立的 WebSocket？

### 答案：
**不是同一个 WebSocket**，Dreamview Plus 使用 **两个独立的 WebSocket 连接**：

### **A. mainConnection (`/websocket`)**
- **用途**：HMIStatus 和核心数据流
- **数据类型**：HMIStatus、SimWorld、Map、PointCloud 等
- **通信方向**：后端 → 前端（数据推送）

### **B. pluginConnection (`/plugin`)**
- **用途**：前端请求和插件通信
- **数据类型**：HMIAction、API 请求、插件数据
- **通信方向**：前端 → 后端（请求/响应）

**代码实现：**
```typescript
export class WebSocketManager {
    readonly mainConnection: WebSocketConnection;      // /websocket
    private readonly pluginConnection: WebSocketConnection;  // /plugin
    
    constructor(mainUrl: string = config.mainUrl, pluginUrl: string = config.pluginUrl) {
        this.mainConnection = new WebSocketConnection(mainUrl);
        this.pluginConnection = new WebSocketConnection(pluginUrl);
    }
}
```

---

## 问题3: 障碍物数据 - 感知数据也是通过 mainConnection 吗？

### 答案：
**不是**，障碍物数据使用 **独立的子 WebSocket 连接**。

### **障碍物数据流架构：**

#### **A. 前端实现**
```typescript
// 障碍物服务
export class ObstacleService {
    subscribeToObstacles(channelName: string): Observable<ObstacleData> {
        return this.streamApi
            .subscribeToDataWithChannel<apollo.perception.IPerceptionObstacles>(
                StreamDataNames.Obstacle,  // 使用独立的障碍物数据流
                channelName
            )
    }
}
```

#### **B. 后端实现**
```cpp
// 后端注册障碍物 WebSocket 处理器
server_->addWebSocketHandler("/obstacle", *obstacle_ws_);
```

#### **C. 数据流管理**
- **前端**：`ChildWsWorkerClass` 管理子 WebSocket 连接
- **后端**：`ObstacleUpdater` 处理障碍物数据更新
- **端点**：`/obstacle`
- **数据格式**：Protobuf 二进制

---

## 问题4: `/hmi` WebSocket 和 HMIStatus 是一个 WebSocket 吗？

### 答案：
**不是同一个**，后端 HMI 类使用 **两个 WebSocket**：

### **A. websocket_ (`/websocket`) - 主连接**
- **用途**：HMIStatus JSON 广播 + HMI 动作处理
- **数据格式**：JSON
- **功能**：
  - 接收 `HMIAction` 请求
  - 广播 `HMIStatus` JSON 数据
  - 处理模式切换请求

### **B. hmi_ws_ (`/hmi`) - 专用连接**
- **用途**：HMIStatus 二进制广播
- **数据格式**：Protobuf 二进制
- **功能**：高性能 HMIStatus 数据推送

**后端实现：**
```cpp
// 后端初始化
websocket_.reset(new WebSocketHandler("websocket"));  // 主连接
hmi_ws_.reset(new WebSocketHandler("HMI"));           // HMI 专用

// 创建 HMI 实例，传入两个 WebSocket
hmi_.reset(new HMI(websocket_.get(), map_service_.get(), hmi_ws_.get()));

// 注册处理器
server_->addWebSocketHandler("/websocket", *websocket_);  // 主连接
server_->addWebSocketHandler("/hmi", *hmi_ws_);           // HMI 专用
```

**前端主要使用 `/websocket` 订阅 HMIStatus。**

---

## 问题5: Frontend 在 WelcomeGuide 改变 mode 时，是通过哪个 WebSocket 发到后端？后端的 update 消息通过哪个 WebSocket 返回前端？

### 答案：

### **A. 前端发送请求**
**通过 `pluginConnection` (`/plugin`) 发送：**

```typescript
// 前端发送模式切换请求
changeSetupMode(mode: string) {
    return this.requestWithoutRes<HMIDataPayload>({
        data: {
            name: '',
            info: {
                action: HMIActions.ChangeMode,
                value: mode,
            },
        },
        type: MainApiTypes.HMIAction,
    });
}
```

**WebSocket 路径：** `pluginConnection` → `/plugin`

### **B. 后端处理请求**
**通过 `websocket_` (`/websocket`) 接收：**

```cpp
// 后端注册 HMIAction 处理器
websocket_->RegisterMessageHandler(
    "HMIAction",
    [this](const Json& json, WebSocketHandler::Connection* conn) {
        // 处理模式切换请求
        // 发送响应到 /websocket
    });
```

### **C. 后端返回响应**
**通过 `websocket_` (`/websocket`) 返回：**

```cpp
// 发送 HMI 动作响应
response["data"]["requestId"] = request_id;
response["data"]["info"]["data"]["isOk"] = true;
response["data"]["info"]["code"] = 0;
response["data"]["info"]["message"] = "Success";
websocket_->SendData(conn, response.dump());
```

### **D. 前端接收更新**
**通过 `mainConnection` (`/websocket`) 接收：**

```typescript
// 订阅 HMIStatus 更新
streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
    if (data?.currentMode === activeWelcomeGuideItemKey) {
        // 模式切换确认
        clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
        subscription.unsubscribe();
    }
});
```

**总结：**
- **发送请求**：`pluginConnection` (`/plugin`)
- **接收响应**：`mainConnection` (`/websocket`)
- **接收状态更新**：`mainConnection` (`/websocket`)

---

## 问题6: 后端接收 HMIAction 发送的响应是什么格式？举一个例子

### 答案：

### **A. 响应格式结构**
```json
{
    "action": "response",
    "data": {
        "requestId": "ChangeMode!abc123def456",
        "info": {
            "action": "ChangeMode",
            "value": "Perception",
            "data": {
                "isOk": true
            },
            "code": 0,
            "message": "Success"
        }
    }
}
```

### **B. 字段说明**
- **`action`**: 固定值 "response"
- **`data.requestId`**: 前端发送的请求 ID
- **`data.info.action`**: 动作类型（如 "ChangeMode"）
- **`data.info.value`**: 动作值（如目标模式 "Perception"）
- **`data.info.data.isOk`**: 操作是否成功
- **`data.info.code`**: 状态码（0=成功，-1=失败）
- **`data.info.message`**: 响应消息

### **C. 后端实现代码**
```cpp
void HMI::RegisterMessageHandlers() {
    websocket_->RegisterMessageHandler(
        "HMIAction",
        [this](const Json& json, WebSocketHandler::Connection* conn) {
            Json response;
            response["action"] = "response";
            
            // 获取请求 ID
            std::string request_id;
            if (!JsonUtil::GetStringByPath(json, "data.requestId", &request_id)) {
                response["data"]["info"]["code"] = -1;
                response["data"]["info"]["message"] = "Miss requestId";
                websocket_->SendData(conn, response.dump());
                return;
            }
            response["data"]["requestId"] = request_id;
            
            // 处理模式切换
            if (hmi_action == HMIAction::CHANGE_MODE) {
                response["data"]["info"]["data"]["isOk"] = true;
                response["data"]["info"]["code"] = 0;
                response["data"]["info"]["message"] = "Success";
                websocket_->SendData(conn, response.dump());
            }
        });
}
```

---

## 问题7: requestId 从哪里获得？

### 答案：

### **A. 前端生成 requestId**
**使用 `genereateRequestId` 工具函数：**

```typescript
// 工具函数
import shortUUID from 'short-uuid';

export const genereateRequestId = (type: string) => {
    const replaceUid = type.replace(/!.*$/, '');
    return `${replaceUid}!${shortUUID.generate()}`;
};
```

### **B. WebSocketManager 中的使用**
```typescript
export class WebSocketManager {
    public request<Req, Res>(
        requestData: RequestDataType<Req>,
        socket: SocketNameEnum = SocketNameEnum.MAIN,
        requestId: string | 'noResponse' = genereateRequestId(requestData.type),
    ): Promise<ResponseMessage<Res>> {
        // 如果没有提供 requestId，自动生成
        if (requestId === 'noResponse') {
            this.sendMessage<Req>({
                ...requestData,
                data: {
                    ...requestData.data,
                    requestId,  // 使用生成的 requestId
                },
                action: RequestMessageActionEnum.REQUEST_MESSAGE_TYPE,
            }, socket);
            return Promise.resolve(null);
        }
    }
}
```

### **C. 实际生成示例**
```typescript
// 对于 ChangeMode 请求
const requestId = genereateRequestId('ChangeMode');
// 结果：'ChangeMode!abc123def456'

// 对于其他请求类型
const requestId = genereateRequestId('GetMap');
// 结果：'GetMap!xyz789uvw012'
```

### **D. 请求 ID 格式**
- **格式**：`{请求类型}!{短UUID}`
- **示例**：`ChangeMode!abc123def456`
- **用途**：唯一标识每个请求，用于匹配请求和响应

---

## 问题8: 用户在 frontend 改变 mode，收到 HMI action response 后会做什么？

### 答案：

### **A. 前端不等待 HMI Action Response**
**关键点：前端使用 `requestWithoutRes`，不等待响应！**

```typescript
// 前端发送请求（不等待响应）
changeSetupMode(mode: string) {
    return this.requestWithoutRes<HMIDataPayload>({
        data: {
            name: '',
            info: {
                action: HMIActions.ChangeMode,
                value: mode,
            },
        },
        type: MainApiTypes.HMIAction,
    });
}
```

### **B. 前端立即执行的操作**
```typescript
const clickEnterThisMode = () => {
    // 1. 立即发送模式切换请求
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey as CURRENT_MODE));
    
    // 2. 立即保存本地状态
    setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
    
    // 3. 立即订阅 HMIStatus 等待确认
    const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
        if (data?.currentMode === activeWelcomeGuideItemKey) {
            // 4. 模式切换确认后执行后续操作
            clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
            subscription.unsubscribe();
        }
    });
};
```

### **C. 前端执行流程**
1. **立即发送请求**：不等待响应
2. **保存本地状态**：记录用户选择
3. **订阅状态流**：等待后端确认
4. **状态确认后**：执行 UI 更新
5. **取消订阅**：避免内存泄漏

### **D. 为什么这样设计？**
- **响应速度快**：用户立即看到反馈
- **可靠性高**：通过状态流确认实际结果
- **错误处理**：可以检测到模式切换失败
- **用户体验好**：避免等待时间

---

## 问题9: 订阅 HMIStatus 等待 mode 更新确认，这是订阅 hmi status stream 吗？

### 答案：

**是的**，这确实是订阅 **HMI Status Stream**。

### **A. 订阅机制**
```typescript
// 订阅 HMIStatus 数据流
const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
    if (data?.currentMode === activeWelcomeGuideItemKey) {
        // 模式切换确认
        clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
        subscription.unsubscribe();
    }
});
```

### **B. 数据流处理**
```typescript
// 初始化 HMIStatus 订阅
function useInitHmiStatus() {
    const { isMainConnected, metadata, streamApi } = useWebSocketServices();
    const [, dispatch] = usePickHmiStore();

    useEffect(() => {
        if (!isMainConnected) return noop;

        const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data) => {
            dispatch(updateStatus({
                ...(data as any),
                frontendIsFromHmi: true,
            } as any));
        });
        
        return () => {
            subscription?.unsubscribe();
        };
    }, [metadata]);
}
```

### **C. 后端数据流**
```cpp
// 后端广播 HMIStatus
hmi_worker_->RegisterStatusUpdateHandler(
    [this](const bool status_changed, HMIStatus* status) {
        if (!status_changed) {
            return;
        }
        // 通过 /websocket 广播 HMIStatus JSON
        websocket_->BroadcastData(
            JsonUtil::ProtoToTypedJson("HMIStatus", *status).dump());
    });
```

### **D. 数据流特点**
- **实时性**：状态变化立即推送
- **可靠性**：通过 WebSocket 保证数据到达
- **完整性**：包含完整的 HMI 状态信息
- **持久性**：持续订阅直到模式切换完成

---

## 问题10: 为了简化 HMIStatus 是否可以不订阅 hmi status stream？

### 答案：

**可以**，有几种简化方案，但需要权衡利弊。

### **A. 简化方案对比**

#### **方案1：直接等待 HMI Action Response**
```typescript
// 修改为等待响应
changeSetupMode(mode: string) {
    return this.request<HMIDataPayload>({  // 改为 request 而不是 requestWithoutRes
        data: {
            name: '',
            info: {
                action: HMIActions.ChangeMode,
                value: mode,
            },
        },
        type: MainApiTypes.HMIAction,
    });
}

// 使用方式
const clickEnterThisMode = async () => {
    try {
        const response = await dispatch(changeMode(mainApi, activeWelcomeGuideItemKey));
        if (response?.data?.info?.data?.isOk) {
            // 模式切换成功
            clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
        }
    } catch (error) {
        // 处理错误
    }
};
```

**优点：**
- 代码简单
- 逻辑清晰
- 减少订阅

**缺点：**
- 响应时间可能较长
- 网络问题可能导致超时
- 无法检测到状态变化

#### **方案2：Promise + 超时机制**
```typescript
const clickEnterThisMode = () => {
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey));
    setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
    
    // 使用 Promise + 超时
    const timeout = new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), 5000)
    );
    
    const statusCheck = new Promise((resolve) => {
        const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
            if (data?.currentMode === activeWelcomeGuideItemKey) {
                subscription.unsubscribe();
                resolve(true);
            }
        });
    });
    
    Promise.race([statusCheck, timeout])
        .then(() => clickEnterThisModeToGuide(activeWelcomeGuideItemKey))
        .catch(() => console.error('Mode change timeout'));
};
```

**优点：**
- 有超时保护
- 逻辑相对简单
- 避免无限等待

**缺点：**
- 仍然需要订阅
- 代码复杂度增加

#### **方案3：本地状态管理**
```typescript
const clickEnterThisMode = () => {
    // 立即更新本地状态
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey));
    setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
    
    // 假设模式切换成功，立即更新 UI
    clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
    
    // 后台异步检查状态
    setTimeout(() => {
        // 检查实际状态是否匹配
        if (currentMode !== activeWelcomeGuideItemKey) {
            // 回滚状态
            console.warn('Mode change failed, rolling back');
        }
    }, 1000);
};
```

**优点：**
- 响应最快
- 无需订阅
- 代码最简单

**缺点：**
- 可能状态不一致
- 错误处理复杂
- 用户体验可能不好

### **B. 推荐方案**

**最佳实践：方案1 + 方案2 结合**

```typescript
const clickEnterThisMode = async () => {
    try {
        // 1. 立即更新本地状态
        setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
        
        // 2. 发送请求并等待响应
        const response = await dispatch(changeMode(mainApi, activeWelcomeGuideItemKey));
        
        // 3. 检查响应
        if (response?.data?.info?.data?.isOk) {
            // 4. 立即更新 UI
            clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
        } else {
            // 5. 处理失败情况
            console.error('Mode change failed:', response?.data?.info?.message);
        }
    } catch (error) {
        // 6. 错误处理
        console.error('Mode change error:', error);
    }
};
```

**这种方案的优势：**
- 代码简洁
- 响应快速
- 错误处理完善
- 无需复杂订阅

---

## 问题11: setupMode 就是切换 WelcomeGuide 中的 Mode 吗？

### 答案：

**是的**，`setupMode` 直接对应切换 WelcomeGuide 中的 Mode。

### **A. 完整流程**

#### **1. 用户选择模式**
```typescript
// WelcomeGuide 中用户点击模式
const clickEnterThisMode = () => {
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey as CURRENT_MODE));
    // activeWelcomeGuideItemKey 就是用户选择的模式
};
```

#### **2. 前端发送请求**
```typescript
// 调用 changeSetupMode
export const changeMode = (
    mainApi: MainApi,
    payload: TYPES.ChangeModePayload,
    callback?: (mode: CURRENT_MODE) => void,
): AsyncAction<TYPES.IInitState, ChangeModeAction> => {
    return async (_dispatch, state) => {
        await mainApi.changeSetupMode(payload);  // 调用 setupMode
        if (callback) {
            callback(payload);
        }
    };
};
```

#### **3. 后端处理请求**
```cpp
// 后端接收 HMIAction
if (hmi_action == HMIAction::CHANGE_MODE) {
    // 处理模式切换
    is_ok = hmi_worker_->Trigger(hmi_action, value);
    
    // 发送响应
    response["data"]["info"]["data"]["isOk"] = true;
    response["data"]["info"]["code"] = 0;
    response["data"]["info"]["message"] = "Success";
    websocket_->SendData(conn, response.dump());
}
```

#### **4. 前端状态更新**
```typescript
// 订阅 HMIStatus 确认模式切换
const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
    if (data?.currentMode === activeWelcomeGuideItemKey) {
        // 模式切换确认，执行后续操作
        clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
        subscription.unsubscribe();
    }
});
```

### **B. 模式对应关系**

| WelcomeGuide 模式 | CURRENT_MODE 枚举 | 后端处理 |
|------------------|------------------|----------|
| Default | DEFAULT | 默认模式 |
| Perception | PERCEPTION | 感知模式 |
| PNC | PNC | 规划控制模式 |
| Vehicle Test | VEHICLE_TEST | 车辆测试模式 |
| Offline Analysis | OFFLINE_ANALYSIS | 离线分析模式 |

### **C. 关键代码位置**

#### **前端 WelcomeGuide**
```typescript
// 文件：WelcomeGuide/index.tsx
const clickEnterThisMode = () => {
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey as CURRENT_MODE));
    // activeWelcomeGuideItemKey 来自用户选择
};
```

#### **前端 API 调用**
```typescript
// 文件：services/api/main.ts
changeSetupMode(mode: string) {
    return this.requestWithoutRes<HMIDataPayload>({
        data: {
            name: '',
            info: {
                action: HMIActions.ChangeMode,
                value: mode,  // 这里就是 WelcomeGuide 中选择的模式
            },
        },
        type: MainApiTypes.HMIAction,
    });
}
```

#### **后端处理**
```cpp
// 文件：backend/hmi/hmi.cc
if (hmi_action == HMIAction::CHANGE_MODE) {
    // value 就是前端传递的模式名称
    is_ok = hmi_worker_->Trigger(hmi_action, value);
}
```

### **D. 总结**

**`setupMode` 就是 WelcomeGuide 中的 Mode 切换：**

1. **用户选择**：在 WelcomeGuide 中选择模式
2. **前端发送**：调用 `changeSetupMode` 发送请求
3. **后端处理**：处理 `CHANGE_MODE` 动作
4. **状态更新**：更新 HMI 状态并广播
5. **前端确认**：订阅 HMIStatus 确认切换成功
6. **UI 更新**：执行后续的 UI 更新操作

---

## 问题12: 切换到 Perception Mode 后会发生什么？

### 答案：

切换到 Perception Mode 后会发生一系列复杂的系统状态变化和 UI 更新。

### **A. 前端 UI 状态更新**

#### **1. WelcomeGuide 消失**
```typescript
// WelcomeGuide 组件隐藏
const clickEnterThisModeToGuide = (mode: CURRENT_MODE) => {
    // 隐藏 WelcomeGuide
    setShowWelcomeGuide(false);
    // 显示主界面
    setShowMainInterface(true);
};
```

#### **2. 主界面显示**
```typescript
// PageLayout 显示主界面
function PageLayout() {
    const [showWelcomeGuide, setShowWelcomeGuide] = useLocalStorageState('showWelcomeGuide', true);
    
    return (
        <div className={c['dv-root']}>
            {showWelcomeGuide ? (
                <WelcomeGuide />
            ) : (
                <div className={c['dv-layout-content']}>
                    <PanelLayout />  {/* 显示面板布局 */}
                </div>
            )}
        </div>
    );
}
```

### **B. 面板激活和布局**

#### **1. 相关面板激活**
```typescript
// 根据模式激活相关面板
const getPanelsForMode = (mode: CURRENT_MODE) => {
    switch (mode) {
        case CURRENT_MODE.PERCEPTION:
            return [
                PanelType.VehicleViz,      // 3D 可视化
                PanelType.CameraView,      // 摄像头视图
                PanelType.PointCloud,      // 点云数据
                PanelType.ObstaclePanel,   // 障碍物面板
                PanelType.Console,         // 控制台
            ];
        // ... 其他模式
    }
};
```

#### **2. 面板布局更新**
```typescript
// PanelLayout 更新布局
function DreamviewWindow() {
    const [hmi] = usePickHmiStore();
    const layout = useGetCurrentLayout();
    
    // 根据当前模式获取布局配置
    const currentLayout = layout[hmi.currentMode] || defaultLayout;
    
    return (
        <Mosaic<string>
            value={currentLayout}
            renderTile={renderTile}
            onChange={onMosaicChange}
        />
    );
}
```

### **C. 后端数据流激活**

#### **1. 感知数据流启动**
```cpp
// 后端启动感知相关数据流
class PerceptionCameraUpdater {
    void Start() {
        // 启动摄像头数据流
        camera_reader_->SetReaderCallback(
            [this](const std::shared_ptr<apollo::drivers::Image>& msg) {
                // 处理摄像头数据
                UpdateCameraData(msg);
            });
    }
};

class ObstacleUpdater {
    void Start() {
        // 启动障碍物数据流
        obstacle_reader_->SetReaderCallback(
            [this](const std::shared_ptr<apollo::perception::PerceptionObstacles>& msg) {
                // 处理障碍物数据
                UpdateObstacleData(msg);
            });
    }
};
```

#### **2. 数据流注册**
```cpp
// 注册数据流处理器
server_->addWebSocketHandler("/camera", *camera_ws_);      // 摄像头数据
server_->addWebSocketHandler("/obstacle", *obstacle_ws_);  // 障碍物数据
server_->addWebSocketHandler("/pointcloud", *pointcloud_ws_); // 点云数据
```

### **D. 前端数据订阅**

#### **1. 障碍物数据订阅**
```typescript
// ObstacleStore 订阅障碍物数据
export const ObstacleProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
    const [obstacleService] = useState(() => streamApi ? new ObstacleService(streamApi) : null);

    useEffect(() => {
        if (!obstacleService || !state.currentChannel) return;
        
        const subscription = obstacleService
            .subscribeToObstacles(state.currentChannel)
            .subscribe({
                next: (data: ObstacleData) => {
                    dispatch({ type: 'SET_OBSTACLES', payload: data.obstacles });
                },
                error: (error) => {
                    dispatch({ type: 'SET_ERROR', payload: error.message });
                }
            });
        
        return () => subscription.unsubscribe();
    }, [obstacleService, state.currentChannel]);
};
```

#### **2. 摄像头数据订阅**
```typescript
// CameraView 订阅摄像头数据
function CameraView() {
    const { streamApi } = useWebSocketServices();
    
    useEffect(() => {
        const subscription = streamApi?.subscribeToData(StreamDataNames.CAMERA).subscribe((data) => {
            // 更新摄像头图像
            setCameraData(data);
        });
        
        return () => subscription?.unsubscribe();
    }, []);
}
```

#### **3. 点云数据订阅**
```typescript
// PointCloud 订阅点云数据
function PointCloud() {
    const { streamApi } = useWebSocketServices();
    
    useEffect(() => {
        const subscription = streamApi?.subscribeToData(StreamDataNames.POINT_CLOUD).subscribe((data) => {
            // 更新点云数据
            setPointCloudData(data);
        });
        
        return () => subscription?.unsubscribe();
    }, []);
}
```

### **E. 3D 可视化更新**

#### **1. VehicleViz 激活**
```typescript
// VehicleViz 开始渲染感知数据
function Viz() {
    const [carviz, uid] = useCarViz();
    
    // 订阅感知数据
    useEffect(() => {
        const subscription = streamApi?.subscribeToData(StreamDataNames.SIM_WORLD).subscribe((data) => {
            // 更新 3D 场景中的障碍物、路径等
            carviz.updatePerceptionData(data.perception);
            carviz.updatePlanningData(data.planning);
        });
        
        return () => subscription?.unsubscribe();
    }, []);
}
```

#### **2. 3D 场景更新**
```typescript
// 3D 场景渲染感知结果
function renderPerceptionData(obstacles: ObstacleData[]) {
    obstacles.forEach(obstacle => {
        // 渲染障碍物
        renderObstacle(obstacle);
        // 渲染预测轨迹
        renderPrediction(obstacle.prediction);
        // 渲染规划路径
        renderPlanning(obstacle.planning);
    });
}
```

### **F. 用户界面变化**

#### **1. 工具栏更新**
```typescript
// 根据模式更新工具栏
const getToolbarForMode = (mode: CURRENT_MODE) => {
    switch (mode) {
        case CURRENT_MODE.PERCEPTION:
            return [
                'view-control',    // 视图控制
                'layer-manager',   // 图层管理
                'camera-switch',   // 摄像头切换
                'obstacle-filter', // 障碍物过滤
            ];
    }
};
```

#### **2. 菜单更新**
```typescript
// 更新菜单选项
const getMenuForMode = (mode: CURRENT_MODE) => {
    switch (mode) {
        case CURRENT_MODE.PERCEPTION:
            return [
                'perception-settings',  // 感知设置
                'camera-calibration',   // 摄像头标定
                'obstacle-tracking',    // 障碍物跟踪
                'data-export',          // 数据导出
            ];
    }
};
```

### **G. 系统状态监控**

#### **1. 控制台日志**
```typescript
// Console 面板显示感知相关日志
function Console() {
    useEffect(() => {
        const subscription = streamApi?.subscribeToData(StreamDataNames.SIM_WORLD).subscribe((data) => {
            data?.notification?.forEach((item) => {
                if (item.item.msg.includes('perception')) {
                    // 显示感知相关日志
                    insertList({
                        level: item.item.logLevel,
                        text: item.item.msg,
                        time: timestampMsToTimeString(item.timestampSec * 1000),
                    });
                }
            });
        });
        
        return () => subscription?.unsubscribe();
    }, []);
}
```

#### **2. 状态监控**
```typescript
// 监控感知系统状态
function monitorPerceptionStatus() {
    const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data) => {
        if (data.currentMode === 'Perception') {
            // 监控感知系统健康状态
            checkPerceptionSystemHealth();
        }
    });
}
```

### **H. 数据流架构**

#### **1. 数据流图**
```
后端感知系统 → WebSocket → 前端面板
     ↓
[摄像头数据] → /camera → CameraView
[障碍物数据] → /obstacle → ObstaclePanel
[点云数据] → /pointcloud → PointCloud
[感知结果] → /websocket → VehicleViz
[系统日志] → /websocket → Console
```

#### **2. 实时数据更新**
- **摄像头图像**：实时显示摄像头画面
- **障碍物检测**：实时显示检测到的障碍物
- **点云数据**：实时显示激光雷达点云
- **3D 可视化**：实时更新 3D 场景
- **系统状态**：实时监控系统健康状态

### **I. 总结**

**切换到 Perception Mode 后的完整流程：**

1. **UI 状态更新**：WelcomeGuide 消失，主界面显示
2. **面板激活**：激活 VehicleViz、CameraView、PointCloud、ObstaclePanel 等
3. **后端数据流**：启动感知相关的数据流处理器
4. **前端订阅**：订阅摄像头、障碍物、点云等数据流
5. **3D 可视化**：开始渲染感知结果和障碍物
6. **用户界面**：更新工具栏、菜单等界面元素
7. **系统监控**：显示感知系统的运行状态和日志
8. **实时更新**：持续接收和显示感知数据

**Perception Mode 是一个完整的感知系统监控界面，为用户提供实时的感知数据可视化和系统状态监控功能。**

---

## 总结

本文档详细记录了从 Frontend WelcomeGuide 改变 Mode 的 WebSocket 通信流程开始的所有问答内容，涵盖了：

1. **WebSocket 通信架构**：mainConnection 和 pluginConnection 的职责分工
2. **数据流管理**：HMIStatus、障碍物数据、感知数据的不同处理方式
3. **请求响应机制**：HMIAction 的完整处理流程
4. **状态管理**：前端如何通过订阅确认模式切换
5. **系统集成**：Perception Mode 的完整激活流程

这些内容为理解 Apollo Dreamview Plus 的 WebSocket 通信机制和模式切换流程提供了全面的技术参考。
