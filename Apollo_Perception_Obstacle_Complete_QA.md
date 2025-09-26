# Apollo Perception Obstacle 完整问答文档

## 目录
1. [Perception Obstacle Streaming 工作机制](#1-perception-obstacle-streaming-工作机制)
2. [数据格式详解](#2-数据格式详解)
3. [前端数据转换机制](#3-前端数据转换机制)
4. [订阅与推送流程](#4-订阅与推送流程)
5. [主连接控制消息](#5-主连接控制消息)
6. [Perception Obstacle Panel](#6-perception-obstacle-panel)
7. [实时障碍物显示](#7-实时障碍物显示)
8. [视觉效果详解](#8-视觉效果详解)

---

## 1. Perception Obstacle Streaming 工作机制

### 1.1 整体架构
Apollo的perception obstacle streaming采用**WebSocket + Cyber RT**的架构，实现了从感知模块到前端界面的实时障碍物数据流传输。

```
感知模块 → Cyber通道 → 后端ObstacleUpdater → WebSocket推送 → 前端订阅 → UI渲染
```

### 1.2 核心组件
- **ObstacleUpdater**: 负责订阅Cyber通道中的障碍物数据并推送给前端
- **SocketManager**: 管理WebSocket连接和消息处理
- **UpdaterManager**: 管理各种数据更新器

### 1.3 数据流程
1. **Cyber通道订阅**: `ObstacleUpdater`订阅`/apollo/perception/obstacles`通道
2. **数据转换**: 将`PerceptionObstacles`消息转换为前端可用的格式
3. **WebSocket推送**: 通过`/obstacle`端点推送给前端

### 1.4 关键文件
- **后端**: `obstacle_updater.cc`, `socket_manager.cc`, `data_handler.conf`
- **前端**: `obstacle.service.ts`, `ObstacleStore`, `websocket-manager.service.ts`
- **消息定义**: `perception_obstacle.proto`

---

## 2. 数据格式详解

### 2.1 数据流中的格式转换过程
整个数据流经历了多个格式转换阶段：

```
感知模块 → Cyber通道 → 后端转换 → WebSocket传输 → 前端反序列化 → UI显示
```

### 2.2 各阶段的数据格式

#### 2.2.1 感知模块输出格式 (PerceptionObstacles)
**源格式**: `apollo.perception.PerceptionObstacles` (Protobuf)

```protobuf
message PerceptionObstacles {
  repeated PerceptionObstacle perception_obstacle = 1;  // 障碍物数组
  optional apollo.common.Header header = 2;             // 消息头
  optional apollo.common.ErrorCode error_code = 3;     // 错误码
  optional LaneMarkers lane_marker = 4;                // 车道标记
  optional CIPVInfo cipv_info = 5;                     // 最近路径车辆
  repeated PerceptionWaste perception_waste = 6;       // 垃圾检测
}
```

#### 2.2.2 后端转换格式 (Object)
**中间格式**: `apollo.dreamview.Object` (Dreamview专用格式)

```protobuf
message Object {
  string id = 1;                    // 障碍物ID
  string type = 2;                  // 障碍物类型
  string sub_type = 3;              // 子类型
  apollo.common.Point3D position = 4; // 3D位置
  double length = 5;                // 长度
  double width = 6;                 // 宽度
  double height = 7;                // 高度
  double theta = 8;                 // 朝向角
  apollo.common.Point3D velocity = 9; // 速度
  double confidence = 10;           // 置信度
}
```

#### 2.2.3 WebSocket传输格式 (StreamData)
**传输格式**: `apollo.dreamview.StreamData` (二进制Protobuf)

```protobuf
message StreamData {
  string name = 1;                  // 数据流名称
  bytes data = 2;                   // 二进制数据
  string channel = 3;               // 通道名称
  int64 timestamp = 4;              // 时间戳
}
```

---

## 3. 前端数据转换机制

### 3.1 前端数据转换的整体架构
前端数据转换采用**多层级、异步、Worker化**的架构：

```
WebSocket接收 → Worker反序列化 → RxJS流处理 → 业务层转换 → UI状态更新
```

### 3.2 数据转换的详细流程

#### 3.2.1 WebSocket数据接收阶段
**位置**: `WebSocketManager.connectChildSocket()`

```typescript
this.activeWorkers[name].socketMessage$
    .pipe(throttle(() => timer(this.throttleDuration.value)))
    .subscribe((message) => {
        if (isMessageType(message, 'SOCKET_MESSAGE')) {
            const { data } = message.payload as StreamMessage;
            
            // 将二进制数据发送到Worker进行反序列化
            this.workerPoolManager.dispatchTask({
                type: 'DECODE_MESSAGE',
                payload: data,
                transferList: [data.buffer]
            });
        }
    });
```

#### 3.2.2 Worker反序列化阶段
**位置**: `decoder.worker.ts`

```typescript
// Worker中的反序列化逻辑
self.onmessage = function(e) {
    const { type, payload } = e.data;
    
    if (type === 'DECODE_MESSAGE') {
        try {
            // 使用Protobuf解码
            const message = StreamData.decode(payload);
            const obstacleData = PerceptionObstacles.decode(message.data);
            
            // 发送解码后的数据
            self.postMessage({
                type: 'DECODE_SUCCESS',
                payload: obstacleData
            });
        } catch (error) {
            self.postMessage({
                type: 'DECODE_ERROR',
                payload: error.message
            });
        }
    }
};
```

#### 3.2.3 业务层转换阶段
**位置**: `ObstacleService.subscribeToObstacles()`

```typescript
subscribeToObstacles(channelName: string): Observable<ObstacleData> {
    return this.streamApi
        .subscribeToDataWithChannel<apollo.perception.IPerceptionObstacles>(
            StreamDataNames.Obstacle, 
            channelName
        )
        .pipe(
            // 第一层转换：Protobuf → 业务数据格式
            map((data: apollo.perception.IPerceptionObstacles) => ({
                obstacles: data.perception_obstacle || [],
                timestamp: data.header?.timestamp_sec || Date.now() / 1000,
                channelName
            })),
            // 第二层转换：过滤和验证
            filter(data => data.obstacles.length > 0),
            // 第三层转换：错误处理
            catchError(error => {
                console.error('Obstacle data error:', error);
                return EMPTY;
            })
        );
}
```

---

## 4. 订阅与推送流程

### 4.1 前端订阅流程详解

#### 4.1.1 应用初始化阶段
**步骤1: 应用启动**
```typescript
// App.tsx
export function App() {
    const Providers = [
        <WebSocketManagerProvider key='WebSocketManagerProvider' />,
        <ObstacleProvider key='ObstacleProvider' />,
        // ... 其他Provider
    ];
    return <CombineContext providers={Providers}>...</CombineContext>;
}
```

**步骤2: WebSocket连接建立**
```typescript
// WebSocketManager初始化
constructor(mainUrl: string = config.mainUrl, pluginUrl: string = config.pluginUrl) {
    this.mainConnection = new WebSocketConnection(mainUrl);  // 主连接: 控制消息
    this.pluginConnection = new WebSocketConnection(pluginUrl); // 插件连接: 插件数据
}
```

#### 4.1.2 障碍物数据订阅
**步骤3: 创建子连接**
```typescript
// 创建障碍物数据流连接
const childSocket = this.connectChildSocket('obstacle', {
    name: StreamDataNames.Obstacle,
    channel: channelName
});
```

**步骤4: 数据订阅**
```typescript
// ObstacleStore中的数据订阅
useEffect(() => {
    if (!obstacleService || !state.currentChannel) return;
    
    dispatch({ type: 'SET_LOADING', payload: true });
    
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
```

### 4.2 后端推送流程详解

#### 4.2.1 Cyber通道订阅
```cpp
// modules/dreamview_plus/backend/obstacle_updater/obstacle_updater.cc
void ObstacleUpdater::OnObstacles(
    const std::shared_ptr<PerceptionObstacles>& obstacles,
    const std::string& channel_name) {
    
    // 转换数据格式
    auto simulation_world = std::make_shared<SimulationWorld>();
    SetObstacleInfo(obstacles, simulation_world.get());
    
    // 推送到WebSocket
    SendToClients(simulation_world);
}
```

#### 4.2.2 WebSocket推送
```cpp
void ObstacleUpdater::SendToClients(
    const std::shared_ptr<SimulationWorld>& simulation_world) {
    
    // 序列化为二进制数据
    std::string serialized_data;
    simulation_world->SerializeToString(&serialized_data);
    
    // 通过WebSocket推送
    socket_manager_->SendToClients("/obstacle", serialized_data);
}
```

---

## 5. 主连接控制消息

### 5.1 主连接架构
**主连接** (`mainConnection`) 是Apollo Dreamview Plus的核心控制通道，负责处理**非流式数据**的控制消息：

```typescript
// WebSocketManager初始化
constructor(mainUrl: string = config.mainUrl, pluginUrl: string = config.pluginUrl) {
    this.mainConnection = new WebSocketConnection(mainUrl);  // 主连接: 控制消息
    this.pluginConnection = new WebSocketConnection(pluginUrl); // 插件连接: 插件数据
}
```

### 5.2 控制消息处理流程

#### 5.2.1 消息发送机制
**位置**: `WebSocketConnection.sendMessage()`

```typescript
sendMessage<T>(message: RequestMessage<T> | RequestStreamMessage) {
    this.messageQueue.enqueue(message);  // 消息入队
    if (this.isConnected()) {
        this.consumeMessageQueue();      // 消费消息队列
    }
}

private consumeMessageQueue() {
    const idleConsume = () => {
        while (!this.messageQueue.isEmpty() && this.isConnected()) {
            const message = this.messageQueue.dequeue();
            if (message) {
                this.socket.next(message);  // 发送到WebSocket
            }
        }
    };
    requestIdleCallback(idleConsume, { timeout: 2000 }); // 浏览器空闲时处理
}
```

### 5.3 具体控制消息例子

#### 5.3.1 HMI模式切换控制消息
**场景**: 用户切换自动驾驶模式（如从Default切换到Perception模式）

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actions.ts
export const changeMode = (
    mainApi: MainApi,
    payload: CURRENT_MODE,  // 'Perception', 'Pnc', 'Vehicle Test' 等
    callback?: (mode: CURRENT_MODE) => void,
): AsyncAction<IInitState, ChangeModeAction> => {
    return async (_dispatch, state) => {
        logger.debug('changeMode', { state, payload });
        await mainApi.changeSetupMode(payload);  // 发送控制消息到后端
        if (callback) {
            callback(payload);
        }
    };
};
```

#### 5.3.2 面板插件初始化控制消息
**场景**: 获取远程面板插件配置

```typescript
// PanelCatalogProvider中
useEffect(() => {
    if (isMainConnected) {
        mainApi
            .getPanelPluginInitData()  // 发送控制消息获取面板配置
            .then((remotePanels) =>
                remotePanels.reduce((result: any, panels: any) => [...result, ...panels.value], [])
            )
            .then((remotePanels) => getAllPanels(remotePanels, t, imgSrc))
            .catch(() => getAllPanels([], t, imgSrc))
            .then((panels) => {
                setAllPanel(panels);  // 更新面板列表
            });
    }
}, [isMainConnected, mainApi, t, imgSrc]);
```

---

## 6. Perception Obstacle Panel

### 6.1 面板概述
**Perception Obstacle Panel** 是Apollo Dreamview Plus中专门用于**实时显示和管理感知障碍物数据**的核心面板组件。

### 6.2 面板架构

#### 6.2.1 组件层次结构
```
ObstaclePanelWrapper
├── ObstaclePanel (主组件)
│   ├── Panel Header (面板头部)
│   │   ├── 标题显示
│   │   └── 控制按钮
│   ├── Filter Panel (过滤器面板)
│   ├── Obstacle List (障碍物列表)
│   └── Details Panel (详情面板)
└── Panel (基础面板包装器)
```

#### 6.2.2 核心组件
**主组件**: `ObstaclePanel`
```typescript
interface ObstaclePanelProps {
  onObstacleSelect?: (obstacle: apollo.perception.IPerceptionObstacle) => void;
  className?: string;
}
```

### 6.3 功能特性

#### 6.3.1 实时障碍物显示
**数据源**: 通过`ObstacleService`订阅实时障碍物数据流

```typescript
// 订阅障碍物数据
const subscription = obstacleService
  .subscribeToObstacles(state.currentChannel)
  .subscribe({
    next: (data: ObstacleData) => {
      dispatch({ type: 'SET_OBSTACLES', payload: data.obstacles });
    }
  });
```

**显示信息**:
- **障碍物ID**: 唯一标识符
- **类型**: 车辆、行人、自行车等
- **子类型**: 轿车、卡车、公交车等
- **位置**: 3D坐标 (x, y, z)
- **置信度**: 检测置信度百分比

#### 6.3.2 交互式障碍物选择
```typescript
// 障碍物点击处理
const handleObstacleClick = (obstacle: apollo.perception.IPerceptionObstacle) => {
  dispatch({ type: 'SELECT_OBSTACLE', payload: obstacle });
  onObstacleSelect?.(obstacle);  // 回调通知父组件
};

// 障碍物悬停处理
const handleObstacleHover = (obstacle: apollo.perception.IPerceptionObstacle | null) => {
  setHoveredObstacle(obstacle);
};
```

#### 6.3.3 智能过滤系统
**过滤器类型**:
- **类型过滤**: 按障碍物类型筛选
- **置信度过滤**: 设置最小置信度阈值
- **距离过滤**: 设置最大检测距离

```typescript
// 过滤器处理
const handleFilterChange = (filterType: string, value: any) => {
  dispatch({ 
    type: 'SET_FILTERS', 
    payload: { [filterType]: value } 
  });
};

// 过滤后的障碍物
const filteredObstacles = useMemo(() => {
  return filterObstacles(state.obstacles, state.filters);
}, [state.obstacles, state.filters]);
```

### 6.4 状态管理

#### 6.4.1 状态结构
```typescript
interface ObstacleState {
  obstacles: apollo.perception.IPerceptionObstacle[];  // 障碍物列表
  selectedObstacle: apollo.perception.IPerceptionObstacle | null;  // 选中障碍物
  isVisible: boolean;         // 可见性
  currentChannel: string | null;  // 当前通道
  loading: boolean;           // 加载状态
  error: string | null;       // 错误信息
  filters: {                  // 过滤器配置
    types: string[];
    minConfidence: number;
    maxDistance: number;
  };
}
```

---

## 7. 实时障碍物显示

### 7.1 实时障碍物显示的主要Panel
Apollo中实时障碍物显示主要在**两个Panel**中实现：

#### 7.1.1 ObstaclePanel (障碍物列表面板)
- **位置**: Perception模式布局的右下角
- **功能**: 以列表形式显示障碍物详细信息
- **显示方式**: 文本列表 + 颜色编码

#### 7.1.2 VehicleViz (车辆可视化面板) 
- **位置**: Perception模式布局的左上角（主要3D可视化区域）
- **功能**: 3D场景中实时渲染障碍物
- **显示方式**: 3D图形渲染 + 空间位置

### 7.2 ObstaclePanel 的实时显示机制

#### 7.2.1 数据订阅流程
```typescript
// ObstaclePanel中的数据订阅
const subscription = obstacleService
  .subscribeToObstacles(state.currentChannel)  // 订阅障碍物数据流
  .subscribe({
    next: (data: ObstacleData) => {
      dispatch({ type: 'SET_OBSTACLES', payload: data.obstacles });
    },
    error: (error) => {
      dispatch({ type: 'SET_ERROR', payload: error.message });
    }
  });
```

#### 7.2.2 实时渲染实现
```typescript
// 障碍物列表实时渲染
<div className="obstacle-list">
  {filteredObstacles.map((obstacle) => {
    const bounds = calculateObstacleBounds(obstacle);  // 计算边界框
    const color = getObstacleColor(obstacle);          // 获取颜色
    const isSelected = state.selectedObstacle?.id === obstacle.id;
    const isHovered = hoveredObstacle?.id === obstacle.id;
    
    return (
      <div
        key={obstacle.id}
        className={`obstacle-item ${isSelected ? 'selected' : ''} ${isHovered ? 'hovered' : ''}`}
        onClick={() => handleObstacleClick(obstacle)}
        onMouseEnter={() => handleObstacleHover(obstacle)}
        onMouseLeave={() => handleObstacleHover(null)}
        style={{ borderLeftColor: color }}  // 颜色编码边框
      >
        <div className="obstacle-info">
          <div className="obstacle-id">ID: {obstacle.id || 'N/A'}</div>
          <div className="obstacle-type">
            {getObstacleTypeDisplayName(obstacle.type || 'UNKNOWN')}
            {obstacle.sub_type && (
              <span className="obstacle-subtype">
                ({getObstacleSubTypeDisplayName(obstacle.sub_type)})
              </span>
            )}
          </div>
          <div className="obstacle-position">
            ({bounds.center.x?.toFixed(2)}, {bounds.center.y?.toFixed(2)})
          </div>
          <div className="obstacle-confidence">
            {((obstacle.confidence || 0) * 100).toFixed(1)}%
          </div>
        </div>
      </div>
    );
  })}
</div>
```

### 7.3 颜色编码系统

#### 7.3.1 障碍物类型颜色映射
```typescript
export const OBSTACLE_TYPE_COLORS = {
  UNKNOWN: '#808080',        // 灰色 - 未知类型
  UNKNOWN_MOVABLE: '#FFA500', // 橙色 - 未知可移动
  UNKNOWN_UNMOVABLE: '#8B4513', // 棕色 - 未知不可移动
  PEDESTRIAN: '#FF0000',     // 红色 - 行人
  BICYCLE: '#00FF00',        // 绿色 - 自行车
  VEHICLE: '#0000FF',         // 蓝色 - 车辆
};
```

#### 7.3.2 障碍物子类型颜色映射
```typescript
export const OBSTACLE_SUBTYPE_COLORS = {
  ST_UNKNOWN: '#808080',     // 灰色
  ST_CAR: '#0000FF',         // 蓝色 - 轿车
  ST_VAN: '#4169E1',         // 皇家蓝 - 面包车
  ST_TRUCK: '#191970',       // 深蓝色 - 卡车
  ST_BUS: '#000080',         // 海军蓝 - 公交车
  ST_CYCLIST: '#00FF00',     // 绿色 - 骑行者
  ST_MOTORCYCLIST: '#32CD32', // 酸橙绿 - 摩托车手
  ST_PEDESTRIAN: '#FF0000',  // 红色 - 行人
  ST_TRAFFICCONE: '#FFD700', // 金色 - 交通锥
};
```

---

## 8. 视觉效果详解

### 8.1 整体视觉设计

#### 8.1.1 面板外观
```
┌─────────────────────────────────────────────────────────────┐
│  Perception Obstacles (5)                    [🔍] [✓] Show │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 🟦 ID: 12345                                           │ │
│  │    Vehicle (Car)                                       │ │
│  │    (12.34, -5.67)                                      │ │
│  │    85.2%                                               │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ �� ID: 12346                                           │ │
│  │    Pedestrian                                          │ │
│  │    (8.91, 2.34)                                        │ │
│  │    92.1%                                               │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 🟩 ID: 12347                                           │ │
│  │    Bicycle                                             │ │
│  │    (-3.45, 7.89)                                       │ │
│  │    78.5%                                               │ │
│  └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Selected Obstacle Details                                  │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ ID: 12345                                               │ │
│  │ Type: VEHICLE                                           │ │
│  │ Speed: 15.23 m/s                                        │ │
│  │ Confidence: 85.2%                                       │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 颜色编码系统

#### 8.2.1 障碍物类型颜色映射
```typescript
// 左侧边框颜色编码
🟦 蓝色 (#0000FF) - VEHICLE (车辆)
�� 红色 (#FF0000) - PEDESTRIAN (行人)  
🟩 绿色 (#00FF00) - BICYCLE (自行车)
🟨 橙色 (#FFA500) - UNKNOWN_MOVABLE (未知可移动)
🟫 棕色 (#8B4513) - UNKNOWN_UNMOVABLE (未知不可移动)
⚪ 灰色 (#808080) - UNKNOWN (未知)
```

### 8.3 用户能看到的具体信息

#### 8.3.1 每个障碍物项目显示的信息
**障碍物卡片结构**:
```
┌─────────────────────────────────────────────────────────────┐
│ 🟦 ID: 12345                    ← 障碍物唯一标识符          │
│    Vehicle (Car)                ← 类型 + 子类型             │
│    (12.34, -5.67)              ← 3D坐标位置 (x, y)         │
│    85.2%                        ← 检测置信度百分比          │
└─────────────────────────────────────────────────────────────┘
```

**详细信息**:
- **ID**: 障碍物唯一标识符 (如: 12345)
- **类型**: 主要类型 (Vehicle, Pedestrian, Bicycle等)
- **子类型**: 具体子类型 (Car, Truck, Bus等)
- **位置**: 3D坐标 (x, y, z) 精确到小数点后2位
- **置信度**: 检测置信度百分比 (0-100%)

### 8.4 交互视觉效果

#### 8.4.1 悬停效果 (Hover)
```css
&:hover {
  background: #333;           /* 背景变亮 */
  transform: translateX(2px); /* 向右移动2px */
}
```
**视觉效果**: 鼠标悬停时，障碍物卡片背景变亮，并向右轻微移动

#### 8.4.2 选中效果 (Selected)
```css
&.selected {
  background: #1e3a8a;        /* 蓝色背景 */
  border-left-color: #3b82f6; /* 蓝色左边框 */
}
```
**视觉效果**: 选中的障碍物卡片背景变为深蓝色，左边框变为亮蓝色

#### 8.4.3 悬停阴影效果
```css
&.hovered {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3); /* 阴影效果 */
}
```
**视觉效果**: 悬停时添加阴影，增强立体感

### 8.5 状态指示

#### 8.5.1 加载状态
```
┌─────────────────────────────────────────────────────────────┐
│                    ⏳ Loading obstacles...                  │
└─────────────────────────────────────────────────────────────┘
```

#### 8.5.2 错误状态
```
┌─────────────────────────────────────────────────────────────┐
│                    ⚠️ Error: Connection failed              │
└─────────────────────────────────────────────────────────────┘
```

#### 8.5.3 无数据状态
```
┌─────────────────────────────────────────────────────────────┐
│                    📭 No obstacles detected                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

Apollo Perception Obstacle系统通过以下核心机制实现：

### 核心架构
1. **WebSocket + Cyber RT**: 实现实时数据流传输
2. **双Panel协作**: ObstaclePanel + VehicleViz 协同显示
3. **分层数据转换**: 多层级格式转换确保数据完整性
4. **状态管理**: React状态管理 + RxJS流处理

### 关键特性
- **实时性**: 10-30Hz的实时数据更新
- **可视化**: 颜色编码 + 3D渲染 + 交互选择
- **过滤功能**: 类型、置信度、距离多维度过滤
- **错误处理**: 完善的错误恢复和状态指示

### 用户体验
- **直观显示**: 深色主题 + 专业界面设计
- **交互友好**: 悬停、选择、过滤等丰富交互
- **信息丰富**: ID、类型、位置、置信度等详细信息
- **响应式**: 适配不同屏幕尺寸

这个系统为Apollo自动驾驶平台提供了**高效、直观、可靠**的障碍物监控解决方案。
