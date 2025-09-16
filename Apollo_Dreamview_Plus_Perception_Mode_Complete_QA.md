# Apollo Dreamview Plus Perception Mode 完整问答总结

## 1. Frontend中用户选择perception mode时相关的代码文件

### 1.1 模式选择UI组件

#### 1.1.1 欢迎引导页面
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/WelcomeGuide/index.tsx`**
  - 首次进入时的模式选择界面
  - 包含Perception Mode的标签页和描述
  - 处理模式切换逻辑：`dispatch(changeMode(mainApi, activeWelcomeGuideItemKey))`

#### 1.1.2 模式设置组件
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/MenuDrawer/ModeSetting/index.tsx`**
  - 模式选择下拉菜单
  - 处理模式切换的onChange事件
  - 调用`changeMode` action进行模式切换

### 1.2 状态管理文件

#### 1.2.1 HMI状态管理
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actionTypes.ts`**
  - 定义`CURRENT_MODE`枚举：`PERCEPTION = 'Perception'`
  - 定义HMI状态接口和类型

- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/reducer.ts`**
  - 初始化状态：`currentMode: CURRENT_MODE.NONE`
  - 处理模式切换的状态更新

- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/reducerHandler.ts`**
  - 处理`changeMode` action的具体逻辑
  - 更新当前模式状态

#### 1.2.2 页面布局管理
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/PageLayout/index.tsx`**
  - 根据当前模式渲染不同的引导组件
  - 当`currentMode === CURRENT_MODE.PERCEPTION`时渲染`PerceptionGuide`

### 1.3 Perception Mode引导组件

#### 1.3.1 感知模式引导
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/Guide/PerceptionGuide/index.tsx`**
  - Perception Mode的完整引导流程
  - 包含5个步骤的引导教程
  - 处理模块选择、操作选择、资源管理等引导

### 1.4 语言文件

#### 1.4.1 多语言支持
- **`modules/dreamview_plus/frontend/packages/dreamview-lang/en/guide.ts`**
  - 英文版本的perception mode相关文本
  - 包含模式描述、模块列表、面板列表等

- **`modules/dreamview_plus/frontend/packages/dreamview-lang/zh/guide.ts`**
  - 中文版本的perception mode相关文本

### 1.5 相关服务文件

#### 1.5.1 WebSocket服务
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/services/hooks/useWebSocketServices.ts`**
  - 提供WebSocket连接服务
  - 支持模式切换时的连接管理

#### 1.5.2 障碍物服务
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/store/ObstacleStore/index.tsx`**
  - Perception Mode下障碍物数据的处理
  - 与perception mode紧密相关的数据流

### 1.6 样式文件

#### 1.6.1 组件样式
- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/Guide/PerceptionGuide/useStyle.ts`**
  - PerceptionGuide组件的样式定义

- **`modules/dreamview_plus/frontend/packages/dreamview-core/src/components/WelcomeGuide/useStyle.ts`**
  - WelcomeGuide组件的样式定义

## 2. 用户选择perception mode并点击"enter this mode"后前端和后端的执行流程

### 2.1 前端执行的操作

#### 2.1.1 模式选择触发
**位置**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/WelcomeGuide/index.tsx`

```typescript
const clickEnterThisMode = () => {
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey as CURRENT_MODE));
    setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
    // 监听模式切换完成
    const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
        if (data?.currentMode === activeWelcomeGuideItemKey) {
            clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
            subscription.unsubscribe();
        }
    });
};
```

#### 2.1.2 状态管理更新
**位置**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actions.ts`

```typescript
export const changeMode = (mainApi: MainApi, payload: TYPES.ChangeModePayload) => {
    return async (_dispatch, state) => {
        logger.debug('changeMode', { state, payload });
        await mainApi.changeSetupMode(payload);  // 发送WebSocket消息到后端
    };
};
```

#### 2.1.3 WebSocket消息发送
**位置**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/services/api/main.ts`

```typescript
changeSetupMode(mode: string) {
    return this.requestWithoutRes<HMIDataPayload>({
        data: {
            name: '',
            info: {
                action: HMIActions.ChangeMode,  // CHANGE_MODE action
                value: mode,  // "Perception"
            },
        },
        type: MainApiTypes.HMIAction,
    });
}
```

#### 2.1.4 UI状态更新
- 更新HMI store中的`currentMode`为`CURRENT_MODE.PERCEPTION`
- 触发`PageLayout`组件重新渲染
- 显示`PerceptionGuide`引导组件
- 启动perception相关的数据订阅服务

#### 2.1.5 数据服务启动
**位置**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/ObstacleStore/index.tsx`

```typescript
// 订阅障碍物数据
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
}, [obstacleService, state.currentChannel]);
```

### 2.2 后端执行的操作

#### 2.2.1 WebSocket消息接收
**位置**: `modules/dreamview_plus/backend/hmi/hmi.cc`

```cpp
// 接收CHANGE_MODE消息
if (hmi_action == HMIAction::CHANGE_OPERATION || hmi_action == HMIAction::CHANGE_MODE) {
    response["data"]["info"]["data"]["isOk"] = true;
    response["data"]["info"]["code"] = 0;
    response["data"]["info"]["message"] = "Success";
    websocket_->SendData(conn, response.dump());
}
```

#### 2.2.2 模式切换处理
**位置**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
case HMIAction::CHANGE_MODE:
    ChangeMode(value);  // 调用ChangeMode方法
    break;
```

#### 2.2.3 模式配置加载
**位置**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
void HMIWorker::ChangeMode(const std::string &mode_name) {
    // 1. 验证模式是否存在
    if (!ContainsKey(config_.modes(), mode_name)) {
        AERROR << "Cannot change to unknown mode " << mode_name;
        return;
    }
    
    // 2. 检查模式是否真的需要切换
    {
        RLock rlock(status_mutex_);
        if (status_.current_mode() == mode_name) {
            return;  // 模式相同，直接返回
        }
    }
    
    // 3. 重置当前模式（停止所有模块）
    ResetMode();
    
    // 4. 加载新模式配置
    std::string default_operation_str = "";
    {
        WLock wlock(status_mutex_);
        status_.set_current_mode(mode_name);  // 设置当前模式为"Perception"
        current_mode_ = util::HMIUtil::LoadMode(config_.modes().at(mode_name));
        
        // 5. 加载车辆特定配置
        HMIMode vehicle_defined_mode;
        const std::string *vehicle_dir = FindOrNull(config_.vehicles(), status_.current_vehicle());
        if (vehicle_dir != nullptr && 
            LoadVehicleDefinedMode(config_.modes().at(mode_name), *vehicle_dir, &vehicle_defined_mode)) {
            MergeToCurrentMode(&vehicle_defined_mode);
        }
        
        // 6. 更新模块和监控组件
        UpdateModeModulesAndMonitoredComponents();
        
        // 7. 更新其他组件状态
        status_.clear_other_components();
        for (const auto &iter : current_mode_.other_components()) {
            status_.mutable_other_components()->insert({iter.first, {}});
        }
        
        // 8. 更新操作列表
        status_.clear_operations();
        status_.mutable_operations()->CopyFrom(current_mode_.operations());
        
        status_changed_ = true;  // 标记状态已改变
    }
    
    // 9. 执行默认操作（如果有）
    if (!default_operation_str.empty()) {
        ChangeOperation(default_operation_str);
    }
    
    // 10. 更新监控和存储
    FuelMonitorManager::Instance()->SetCurrentMode(mode_name);
    KVDB::Put(FLAGS_current_mode_db_key, mode_name);
}
```

#### 2.2.4 感知相关模块启动
根据perception mode的配置，后端会：

- **启动感知模块**: 启动perception相关的cyber模块
- **启动传感器模块**: 启动Lidar、Radar、Camera等传感器模块
- **启动数据流**: 开始处理perception obstacles数据流
- **更新监控组件**: 更新需要监控的组件列表

#### 2.2.5 数据流处理
**位置**: `modules/dreamview_plus/backend/simulation_world/simulation_world_service.cc`

```cpp
// 处理perception obstacles数据
void SimulationWorldService::UpdatePerceptionObstacles(const PerceptionObstacles &obstacles) {
    for (const auto &obstacle : obstacles.perception_obstacle()) {
        // 处理每个障碍物数据
        auto &world_obj = CreateWorldObjectIfAbsent(obstacle);
        // 更新障碍物信息
    }
}
```

#### 2.2.6 WebSocket数据推送
后端开始向前端推送：
- **HMI状态更新**: 包含当前模式信息
- **感知数据**: PerceptionObstacles消息
- **点云数据**: PointCloud数据
- **传感器数据**: Camera、Lidar、Radar数据

## 3. 前端向后端发送的消息格式

### 3.1 消息结构概览

前端发送的所有WebSocket消息都遵循统一的结构格式：

```typescript
interface RequestMessage<T> {
    type: string;                    // 消息类型
    action: RequestMessageActionEnum; // 操作类型
    data: PayloadField<T>;          // 数据负载
}
```

### 3.2 数据负载字段结构

```typescript
type PayloadField<T> = {
    name: string;        // 字段名称
    source: string;      // 源（固定为'dreamview'）
    info: T;            // 具体信息
    target: string;      // 目标（固定为'dreamview'）
    sourceType: string;  // 源类型（固定为'module'）
    targetType: string;  // 目标类型（固定为'dreamview'）
    requestId: string;   // 请求ID
};
```

### 3.3 Perception Mode切换消息格式

当用户选择perception mode并点击"enter this mode"时，前端发送的具体消息格式：

#### 3.3.1 原始请求数据
```typescript
// 在 changeSetupMode 方法中构建的原始数据
{
    data: {
        name: '',
        info: {
            action: 'CHANGE_MODE',  // HMIActions.ChangeMode
            value: 'Perception',    // 模式名称
        },
        requestId?: string;         // 可选的请求ID
    },
    type: 'HMIAction'               // MainApiTypes.HMIAction
}
```

#### 3.3.2 经过generateRequest处理后的完整消息
```json
{
    "type": "HMIAction",
    "action": "request",
    "data": {
        "name": "",
        "source": "dreamview",
        "info": {
            "action": "CHANGE_MODE",
            "value": "Perception"
        },
        "target": "dreamview",
        "sourceType": "module",
        "targetType": "dreamview",
        "requestId": "unique_request_id_here"
    }
}
```

### 3.4 其他常见HMI操作消息格式

#### 3.4.1 切换地图
```json
{
    "type": "HMIAction",
    "action": "request",
    "data": {
        "name": "",
        "source": "dreamview",
        "info": {
            "action": "CHANGE_MAP",
            "value": "map_name"
        },
        "target": "dreamview",
        "sourceType": "module",
        "targetType": "dreamview",
        "requestId": "request_id"
    }
}
```

#### 3.4.2 切换车辆
```json
{
    "type": "HMIAction",
    "action": "request",
    "data": {
        "name": "",
        "source": "dreamview",
        "info": {
            "action": "CHANGE_VEHICLE",
            "value": "vehicle_name"
        },
        "target": "dreamview",
        "sourceType": "module",
        "targetType": "dreamview",
        "requestId": "request_id"
    }
}
```

#### 3.4.3 启动模块
```json
{
    "type": "HMIAction",
    "action": "request",
    "data": {
        "name": "",
        "source": "dreamview",
        "info": {
            "action": "START_MODULE",
            "value": "module_name"
        },
        "target": "dreamview",
        "sourceType": "module",
        "targetType": "dreamview",
        "requestId": "request_id"
    }
}
```

### 3.5 消息类型枚举

#### 3.5.1 操作类型 (RequestMessageActionEnum)
```typescript
enum RequestMessageActionEnum {
    REQUEST_MESSAGE_TYPE = 'request',        // 请求消息
    SUBSCRIBE_MESSAGE_TYPE = 'subscribe',    // 订阅消息
    UNSUBSCRIBE_MESSAGE_TYPE = 'unsubscribe' // 取消订阅消息
}
```

#### 3.5.2 HMI操作类型 (HMIActions)
```typescript
enum HMIActions {
    ChangeMode = 'CHANGE_MODE',           // 切换模式
    ChangeMap = 'CHANGE_MAP',             // 切换地图
    ChangeVehicle = 'CHANGE_VEHICLE',     // 切换车辆
    StartModule = 'START_MODULE',         // 启动模块
    StopModule = 'STOP_MODULE',           // 停止模块
    ChangeOperation = 'CHANGE_OPERATION', // 切换操作
    // ... 其他操作
}
```

## 4. 后端收到perception mode切换请求的处理流程

### 4.1 消息接收和解析阶段

#### 4.1.1 WebSocket消息处理器
**文件**: `modules/dreamview_plus/backend/hmi/hmi.cc`

```cpp
// 注册HMIAction消息处理器
websocket_->RegisterMessageHandler(
    "HMIAction",
    [this](const Json& json, WebSocketHandler::Connection* conn) {
        // 1. 提取requestId
        std::string request_id;
        if (!JsonUtil::GetStringByPath(json, "data.requestId", &request_id)) {
            AERROR << "Failed to start hmi action: requestId not found.";
            // 返回错误响应
            return;
        }
        
        // 2. 提取action字段
        std::string action;
        if (!JsonUtil::GetStringByPath(json, "data.info.action", &action)) {
            AERROR << "Truncated HMIAction request.";
            return;
        }
        
        // 3. 解析HMIAction枚举
        HMIAction hmi_action;
        if (!HMIAction_Parse(action, &hmi_action)) {
            AERROR << "Invalid HMIAction string: " << action;
            return;
        }
        
        // 4. 提取value字段（模式名称）
        std::string value;
        if (JsonUtil::GetStringByPath(json, "data.info.value", &value)) {
            is_ok = hmi_worker_->Trigger(hmi_action, value);  // 调用HMIWorker处理
        }
        
        // 5. 发送响应给前端
        if (hmi_action == HMIAction::CHANGE_MODE) {
            response["data"]["info"]["data"]["isOk"] = true;
            response["data"]["info"]["code"] = 0;
            response["data"]["info"]["message"] = "Success";
            websocket_->SendData(conn, response.dump());
        }
    });
```

### 4.2 HMI Worker处理阶段

#### 4.2.1 触发模式切换
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
bool HMIWorker::Trigger(const HMIAction action, const std::string &value) {
    AINFO << "HMIAction " << HMIAction_Name(action) << "(" << value << ") was triggered!";
    bool ret = true;
    switch (action) {
        case HMIAction::CHANGE_MODE:
            ChangeMode(value);  // 调用ChangeMode方法
            break;
        // ... 其他action处理
    }
    return ret;
}
```

#### 4.2.2 模式切换核心逻辑
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
void HMIWorker::ChangeMode(const std::string &mode_name) {
    // 1. 验证模式是否存在
    if (!ContainsKey(config_.modes(), mode_name)) {
        AERROR << "Cannot change to unknown mode " << mode_name;
        return;
    }
    
    // 2. 检查模式是否真的需要切换
    {
        RLock rlock(status_mutex_);
        if (status_.current_mode() == mode_name) {
            return;  // 模式相同，直接返回
        }
    }
    
    // 3. 重置当前模式（停止所有模块）
    ResetMode();
    
    // 4. 加载新模式配置
    std::string default_operation_str = "";
    {
        WLock wlock(status_mutex_);
        status_.set_current_mode(mode_name);  // 设置当前模式为"Perception"
        current_mode_ = util::HMIUtil::LoadMode(config_.modes().at(mode_name));
        
        // 5. 加载车辆特定配置
        HMIMode vehicle_defined_mode;
        const std::string *vehicle_dir = FindOrNull(config_.vehicles(), status_.current_vehicle());
        if (vehicle_dir != nullptr && 
            LoadVehicleDefinedMode(config_.modes().at(mode_name), *vehicle_dir, &vehicle_defined_mode)) {
            MergeToCurrentMode(&vehicle_defined_mode);
        }
        
        // 6. 更新模块和监控组件
        UpdateModeModulesAndMonitoredComponents();
        
        // 7. 更新其他组件状态
        status_.clear_other_components();
        for (const auto &iter : current_mode_.other_components()) {
            status_.mutable_other_components()->insert({iter.first, {}});
        }
        
        // 8. 更新操作列表
        status_.clear_operations();
        status_.mutable_operations()->CopyFrom(current_mode_.operations());
        
        status_changed_ = true;  // 标记状态已改变
    }
    
    // 9. 执行默认操作（如果有）
    if (!default_operation_str.empty()) {
        ChangeOperation(default_operation_str);
    }
    
    // 10. 更新监控和存储
    FuelMonitorManager::Instance()->SetCurrentMode(mode_name);
    KVDB::Put(FLAGS_current_mode_db_key, mode_name);
}
```

### 4.3 模式重置阶段

#### 4.3.1 重置当前模式
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
void HMIWorker::ResetMode() {
    {
        // 设置后端关闭标志，触发自动关闭
        WLock wlock(status_mutex_);
        status_.set_backend_shutdown(true);
    }
    
    // 停止所有当前模式的模块
    for (const auto &iter : current_mode_.modules()) {
        {
            WLock wlock(status_mutex_);
            auto modules = status_.modules();
            if (modules[iter.first]) {
                LockModule(iter.first, true);  // 锁定模块
            }
        }
        System(iter.second.stop_command());  // 执行停止命令
    }
    record_count_ = 0;
}
```

### 4.4 配置更新阶段

#### 4.4.1 更新模块和监控组件
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
void HMIWorker::UpdateModeModulesAndMonitoredComponents() {
    // 清空并重新设置模块状态
    status_.clear_modules();
    status_.clear_modules_lock();
    previous_modules_lock_.clear();
    
    // 为新模式的所有模块设置初始状态
    for (const auto &iter : current_mode_.modules()) {
        status_.mutable_modules()->insert({iter.first, false});  // 模块未启动
        status_.mutable_modules_lock()->insert({iter.first, false});  // 模块未锁定
        previous_modules_lock_.insert({iter.first, false});
    }
    
    // 更新监控组件
    status_.clear_monitored_components();
    for (const auto &iter : current_mode_.monitored_components()) {
        status_.mutable_monitored_components()->insert({iter.first, {}});
    }
}
```

### 4.5 状态更新和通知阶段

#### 4.5.1 状态更新线程
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

```cpp
void HMIWorker::StatusUpdateThreadLoop() {
    constexpr int kLoopIntervalMs = 200;
    while (!stop_) {
        std::this_thread::sleep_for(std::chrono::milliseconds(kLoopIntervalMs));
        UpdateComponentStatus();
        
        bool status_changed = false;
        {
            WLock wlock(status_mutex_);
            status_changed = status_changed_;  // 检查状态是否改变
            status_changed_ = false;
        }
        
        // 如果状态改变，触发状态更新处理器
        if (status_changed) {
            HMIStatus status = GetStatus();
            for (const auto handler : status_update_handlers_) {
                handler(status_changed, &status);  // 通知所有处理器
            }
        }
    }
}
```

### 4.6 相关代码文件总结

#### 4.6.1 核心处理文件
- **`modules/dreamview_plus/backend/hmi/hmi.cc`**: WebSocket消息接收和分发
- **`modules/dreamview_plus/backend/hmi/hmi_worker.cc`**: 模式切换核心逻辑
- **`modules/dreamview_plus/backend/hmi/hmi_worker.h`**: HMI Worker类定义

#### 4.6.2 配置相关文件
- **HMI模式配置文件**: 通过`FLAGS_dv_plus_hmi_modes_config_path`指定
- **车辆配置文件**: 通过`config_.vehicles()`获取
- **模块配置文件**: 通过`current_mode_.modules()`获取

#### 4.6.3 工具类文件
- **`modules/dreamview_plus/backend/hmi/util/hmi_util.h`**: HMI工具类，包含`LoadMode`等方法

### 4.7 处理流程总结

1. **消息接收**: WebSocket接收前端消息并解析JSON
2. **参数验证**: 验证requestId、action、value等参数
3. **模式验证**: 检查目标模式是否存在于配置中
4. **模式重置**: 停止当前模式的所有模块
5. **配置加载**: 加载新模式配置和车辆特定配置
6. **状态更新**: 更新模块状态、监控组件、操作列表等
7. **响应发送**: 向前端发送成功响应
8. **状态通知**: 通过状态更新线程通知所有监听器

## 5. 添加Offline模式的完整实现指南

### 5.1 前端修改

#### 5.1.1 添加模式枚举定义
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actionTypes.ts`

```typescript
export enum CURRENT_MODE {
    NONE = 'none',
    DEFAULT = 'Default',
    PERCEPTION = 'Perception',
    PNC = 'Pnc',
    VEHICLE_TEST = 'Vehicle Test',
    OFFLINE = 'Offline',  // 新增Offline模式
    MAP_COLLECT = 'Map Collect',
    MAP_EDITOR = 'Map Editor',
    CAMERA_CALIBRATION = 'Camera Calibration',
    LiDAR_CALIBRATION = 'Lidar Calibration',
    DYNAMICS_CALIBRATION = 'Dynamics Calibration',
    CANBUS_DEBUG = 'Canbus Debug',
}
```

#### 5.1.2 更新欢迎引导页面
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/WelcomeGuide/index.tsx`

```typescript
const WelcomeGuideTabsItem = [
    {
        key: CURRENT_MODE.DEFAULT,
        label: 'Default Mode',
        children: (
            <WelcomeGuideTabsContent
                contentImageUrl={welcomeGuideTabsImageDefault}
                clickEnterThisMode={clickEnterThisMode}
                currentMode={CURRENT_MODE.DEFAULT}
            />
        ),
    },
    {
        key: CURRENT_MODE.PERCEPTION,
        label: 'Perception Mode',
        children: (
            <WelcomeGuideTabsContent
                contentImageUrl={welcomeGuideTabsImagePerception}
                clickEnterThisMode={clickEnterThisMode}
                currentMode={CURRENT_MODE.PERCEPTION}
            />
        ),
    },
    {
        key: CURRENT_MODE.OFFLINE,  // 新增Offline模式标签页
        label: 'Offline Mode',
        children: (
            <WelcomeGuideTabsContent
                contentImageUrl={welcomeGuideTabsImageOffline}  // 需要添加图片
                clickEnterThisMode={clickEnterThisMode}
                currentMode={CURRENT_MODE.OFFLINE}
            />
        ),
    },
    // ... 其他模式
];
```

#### 5.1.3 更新页面布局处理
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/PageLayout/index.tsx`

```typescript
const clickEnterThisModeToGuide = useCallback(
    (useGuideMode: CURRENT_MODE) => {
        if (useGuideMode === CURRENT_MODE.DEFAULT) {
            setEnterGuideState({ currentMode: CURRENT_MODE.DEFAULT });
        }
        if (useGuideMode === CURRENT_MODE.PERCEPTION) {
            setEnterGuideState({ currentMode: CURRENT_MODE.PERCEPTION });
        }
        if (useGuideMode === CURRENT_MODE.PNC) {
            setEnterGuideState({ currentMode: CURRENT_MODE.PNC });
        }
        if (useGuideMode === CURRENT_MODE.VEHICLE_TEST) {
            setEnterGuideState({ currentMode: CURRENT_MODE.VEHICLE_TEST });
        }
        if (useGuideMode === CURRENT_MODE.OFFLINE) {  // 新增Offline模式处理
            setEnterGuideState({ currentMode: CURRENT_MODE.OFFLINE });
        }
    },
    [isMainConnected],
);

// 在渲染部分添加
{enterGuideState.currentMode === CURRENT_MODE.OFFLINE && (
    <OfflineGuide />  // 需要创建OfflineGuide组件
)}
```

#### 5.1.4 创建Offline模式引导组件
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/Guide/OfflineGuide/index.tsx`

```typescript
import React from 'react';
import { useTranslation } from 'react-i18next';

export function OfflineGuide() {
    const { t } = useTranslation('guide');
    
    return (
        <div className="offline-guide">
            <h2>{t('offlineMode')}</h2>
            <p>{t('offlineModeDesc')}</p>
            {/* 添加Offline模式特定的引导内容 */}
        </div>
    );
}

export default React.memo(OfflineGuide);
```

#### 5.1.5 更新语言文件
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-lang/en/guide.ts`

```typescript
export const guide = {
    // ... 现有内容
    offlineMode: 'Offline Mode',
    offlineModeDesc: 'The offline mode is suitable for data replay and analysis scenarios. In this mode, you can load and analyze recorded data without real-time sensor input.',
    OfflineModules: 'Data Recorder、Record Player、Data Analysis',
    OfflinePanel: 'Vehicle Visualization、Console、Module Delay、Data Analysis',
    // ... 其他内容
};
```

**文件**: `modules/dreamview_plus/frontend/packages/dreamview-lang/zh/guide.ts`

```typescript
export const guide = {
    // ... 现有内容
    offlineMode: '离线模式',
    offlineModeDesc: '离线模式适用于数据回放和分析场景。在此模式下，您可以加载和分析录制的数据，无需实时传感器输入。',
    OfflineModules: '数据录制器、记录播放器、数据分析',
    OfflinePanel: '车辆可视化、控制台、模块延时、数据分析',
    // ... 其他内容
};
```

#### 5.1.6 添加Offline模式图片资源
**文件**: `modules/dreamview_plus/frontend/packages/dreamview-ui/src/components/ImagePrak/type.ts`

```typescript
export type IImageName =
    // ... 现有图片
    | 'welcome_guide_tabs_image_offline'  // 新增Offline模式图片
    | 'offline_mode_illustrator';
```

### 5.2 后端修改

#### 5.2.1 创建Offline模式配置文件
**文件**: `modules/dreamview_plus/backend/hmi/testdata/offline_mode.pb.txt`

```protobuf
modules {
  key: "DataRecorder"
  value {
    start_command: "cyber_launch start modules/data_recorder/launch/data_recorder.launch"
    stop_command: "cyber_launch stop modules/data_recorder/launch/data_recorder.launch"
  }
}
modules {
  key: "RecordPlayer"
  value {
    start_command: "cyber_launch start modules/record_player/launch/record_player.launch"
    stop_command: "cyber_launch stop modules/record_player/launch/record_player.launch"
  }
}
monitored_components {
  key: "DataRecorder"
  value {
    process_status {
      message: ""
      status: UNKNOWN
    }
    resource_status {
      message: ""
      status: UNKNOWN
    }
  }
}
monitored_components {
  key: "RecordPlayer"
  value {
    process_status {
      message: ""
      status: UNKNOWN
    }
    resource_status {
      message: ""
      status: UNKNOWN
    }
  }
}
operations: RECORD
operations: PLAY
default_operation: RECORD
```

#### 5.2.2 更新HMI主配置文件
**文件**: `modules/dreamview_plus/backend/hmi/testdata/hmi_modes_config.pb.txt`

```protobuf
modes {
  key: "Default"
  value: "modules/dreamview_plus/backend/hmi/testdata/default_mode.pb.txt"
}
modes {
  key: "Perception"
  value: "modules/dreamview_plus/backend/hmi/testdata/perception_mode.pb.txt"
}
modes {
  key: "Offline"  // 新增Offline模式配置
  value: "modules/dreamview_plus/backend/hmi/testdata/offline_mode.pb.txt"
}
// ... 其他模式配置
```

#### 5.2.3 更新后端代码以支持Offline模式
**文件**: `modules/dreamview_plus/backend/hmi/hmi_worker.cc`

在`ChangeMode`方法中添加Offline模式的特殊处理：

```cpp
void HMIWorker::ChangeMode(const std::string &mode_name) {
    // ... 现有代码 ...
    
    // 为Offline模式添加特殊处理
    if (mode_name == "Offline") {
        // 停止实时数据流
        StopRealTimeDataStreams();
        // 启动离线数据播放器
        StartOfflineDataPlayer();
    }
    
    // ... 其他现有代码 ...
}
```

#### 5.2.4 添加Offline模式特定的数据处理器
**文件**: `modules/dreamview_plus/backend/offline_data/offline_data_updater.cc`

```cpp
#include "modules/dreamview_plus/backend/offline_data/offline_data_updater.h"

namespace apollo {
namespace dreamview {

OfflineDataUpdater::OfflineDataUpdater(WebSocketHandler *websocket)
    : websocket_(websocket),
      node_(cyber::CreateNode("offline_data_updater")) {
}

void OfflineDataUpdater::StartOfflineMode() {
    // 启动离线数据播放逻辑
    AINFO << "Starting offline mode data processing";
}

void OfflineDataUpdater::StopOfflineMode() {
    // 停止离线数据播放
    AINFO << "Stopping offline mode data processing";
}

}  // namespace dreamview
}  // namespace apollo
```

### 5.3 测试和验证

#### 5.3.1 前端测试
```bash
cd modules/dreamview_plus/frontend
npm run build
npm run start
```

#### 5.3.2 后端测试
```bash
cd modules/dreamview_plus/backend
bazel build //modules/dreamview_plus/backend:hmi
./bazel-bin/modules/dreamview_plus/backend/hmi
```

### 5.4 文件修改总结

#### 5.4.1 前端文件列表
1. **`actionTypes.ts`** - 添加OFFLINE枚举
2. **`WelcomeGuide/index.tsx`** - 添加Offline标签页
3. **`PageLayout/index.tsx`** - 添加Offline模式处理
4. **`OfflineGuide/index.tsx`** - 创建Offline引导组件
5. **`guide.ts` (中英文)** - 添加Offline模式描述
6. **`ImagePrak/type.ts`** - 添加Offline模式图片资源

#### 5.4.2 后端文件列表
1. **`offline_mode.pb.txt`** - Offline模式配置
2. **`hmi_modes_config.pb.txt`** - 主配置文件
3. **`hmi_worker.cc`** - 添加Offline模式处理逻辑
4. **`offline_data_updater.cc`** - Offline数据处理器

### 5.5 实现步骤

1. **第一步**: 修改前端枚举定义和UI组件
2. **第二步**: 创建后端配置文件和处理器
3. **第三步**: 更新语言文件和图片资源
4. **第四步**: 测试前后端集成
5. **第五步**: 验证Offline模式功能

### 5.6 注意事项

1. 确保所有新增的枚举值和字符串与后端配置保持一致
2. 图片资源需要按照现有规范添加到相应的图片包中
3. 语言文件需要同时更新中英文版本
4. 后端配置文件需要遵循protobuf格式规范
5. 测试时需要验证模式切换的完整流程

## 总结

本文档详细介绍了Apollo Dreamview Plus中perception mode相关的所有内容，包括：

1. **前端相关代码文件** - 完整的文件列表和功能说明
2. **前后端执行流程** - 从用户选择到系统切换的完整流程
3. **WebSocket消息格式** - 详细的消息结构和示例
4. **后端处理流程** - 后端接收和处理请求的完整过程
5. **Offline模式实现** - 添加新模式的完整指南

通过这个文档，开发者可以全面了解Apollo Dreamview Plus的模式切换机制，并能够成功添加新的模式支持。
