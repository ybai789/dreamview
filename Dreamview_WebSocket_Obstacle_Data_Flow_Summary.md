# Apollo Dreamview WebSocket 连接与 Obstacle 数据流总结

## 1. WebSocket 连接建立与 Metadata 消息发送

### 1.1 连接建立过程

**前端连接建立：**
- 通过 `WebSocketConnection.connect()` 方法建立与后端的WebSocket连接
- 连接状态通过 `ConnectionStatusEnum` 进行管理：`DISCONNECTED` → `CONNECTING` → `CONNECTED` → `METADATA`

**后端连接处理：**
- 后端使用 `RegisterConnectionReadyHandler` 注册连接就绪处理器
- 当新客户端连接时，后端会自动触发相应的处理逻辑

### 1.2 Metadata 消息自动发送机制

**是的，后端在连接建立时会自动发送metadata消息！**

从代码分析可以看出：

#### Dreamview Plus后端 (modules/dreamview_plus/backend/socket_manager/socket_manager.cc)：
```cpp
void SocketManager::RegisterMessageHandlers() {
  websocket_->RegisterConnectionReadyHandler(
      [this](const std::string& client_id) {
        // 连接建立时自动发送metadata
        SendMetadata(client_id);
      });
}
```

#### 传统Dreamview后端：
```cpp
void HMI::RegisterConnectionReadyHandler() {
  websocket_->RegisterConnectionReadyHandler(
      [this](const std::string& client_id) {
        // 连接建立时自动发送状态和车辆参数
        SendStatus(client_id);
        SendVehicleParam(client_id);
      });
}
```

## 2. 前端从后端获取 Perception Obstacle 的完整流程

### 2.1 系统架构概览

整个数据流涉及以下组件：
- **前端**: ObstacleService, ObstacleStore, ObstaclePanel
- **WebSocket连接**: WebSocketManager, StreamApi
- **后端**: ObstacleUpdater, SocketManager
- **Cyber通道**: PerceptionObstacles消息

### 2.2 详细数据流程

#### 阶段1: 前端订阅初始化

**1.1 组件初始化**
```typescript
// ObstacleStore/index.tsx
const { streamApi } = useWebSocketServices();
const [obstacleService] = useState(() => streamApi ? new ObstacleService(streamApi) : null);
```

**1.2 订阅障碍物数据**
```typescript
// ObstacleService.ts
subscribeToObstacles(channelName: string): Observable<ObstacleData> {
    return this.streamApi
        .subscribeToDataWithChannel<apollo.perception.IPerceptionObstacle>(
            channelName,
            StreamDataNames.Obstacle,
            this.obstacleUpdater
        );
}
```

#### 阶段2: 后端数据推送

**2.1 后端订阅Cyber通道**
```cpp
// ObstacleUpdater::Start() 在SocketManager::Start()中调用
void ObstacleUpdater::Start() {
    // 订阅perception obstacles通道
    cyber::Reader<apollo::perception::PerceptionObstacles> reader(
        node_, "/apollo/perception/obstacles", obstacleCallback);
}
```

**2.2 数据转换和推送**
```cpp
void ObstacleUpdater::obstacleCallback(
    const std::shared_ptr<apollo::perception::PerceptionObstacles>& obstacles) {
    // 转换数据格式
    auto obstacleData = ConvertObstacles(obstacles);
    
    // 通过WebSocket推送给前端
    websocket_->SendData(client_id, obstacleData);
}
```

#### 阶段3: 前端数据处理

**3.1 数据接收和更新**
```typescript
// ObstacleService.ts
private obstacleUpdater = (obstacleData: ObstacleData) => {
    // 更新障碍物数据
    this.obstacleDataSubject.next(obstacleData);
};
```

**3.2 状态管理**
```typescript
// ObstacleStore/index.tsx
const [obstacleData, setObstacleData] = useState<ObstacleData | null>(null);

useEffect(() => {
    if (obstacleService && currentChannel) {
        const subscription = obstacleService
            .subscribeToObstacles(currentChannel)
            .subscribe(setObstacleData);
        
        return () => subscription.unsubscribe();
    }
}, [obstacleService, currentChannel]);
```

## 3. ObstacleService.ts 订阅障碍物数据的启动过程

### 3.1 系统初始化阶段

#### 3.1.1 App.tsx - 应用启动
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/App.tsx
export function App() {
    const Providers = [
        <WebSocketManagerProvider key='WebSocketManagerProvider' />,
        <ObstacleProvider key='ObstacleProvider' />,  // 注册ObstacleProvider
        // ... 其他Provider
    ];
}
```

#### 3.1.2 WebSocketManagerProvider - WebSocket连接建立
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/WebSocketManagerStore/index.tsx
function WebSocketManagerInner() {
    useEffect(() => {
        store.mainApi.webSocketManager.connectMain().then(() => {
            // WebSocket连接建立成功
        });
    }, []);
}
```

#### 3.1.3 ObstacleProvider - 服务初始化
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/ObstacleStore/index.tsx
function ObstacleProviderInner() {
    const { streamApi } = useWebSocketServices();
    const [obstacleService] = useState(() => 
        streamApi ? new ObstacleService(streamApi) : null
    );
    
    // 提供obstacleService给子组件
    return (
        <ObstacleContext.Provider value={{ obstacleService }}>
            {children}
        </ObstacleContext.Provider>
    );
}
```

### 3.2 面板激活阶段

#### 3.2.1 ObstaclePanel 组件加载
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/ObstaclePanel/index.tsx
function ObstaclePanel() {
    const { obstacleService } = useObstacleService();
    const { currentChannel } = usePanelContext();
    
    // 当面板激活且currentChannel设置后，开始订阅数据
    useEffect(() => {
        if (obstacleService && currentChannel) {
            const subscription = obstacleService
                .subscribeToObstacles(currentChannel)
                .subscribe(handleObstacleData);
            
            return () => subscription.unsubscribe();
        }
    }, [obstacleService, currentChannel]);
}
```

#### 3.2.2 通道选择机制
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/base/ChannelSelect/index.tsx
function ChannelSelect() {
    const { currentChannel, updateChannel } = usePanelContext();
    
    const handleChannelChange = (newChannel: string) => {
        updateChannel(newChannel);  // 更新通道，触发重新订阅
    };
}
```

### 3.3 涉及的关键TypeScript文件

1. **App.tsx** - 应用入口，注册Provider
2. **WebSocketManagerStore/index.tsx** - WebSocket连接管理
3. **ObstacleStore/index.tsx** - ObstacleService实例化和状态管理
4. **ObstacleService.ts** - 核心服务类，处理数据订阅
5. **ObstaclePanel/index.tsx** - 面板组件，使用ObstacleService
6. **ChannelSelect/index.tsx** - 通道选择组件
7. **useRegisterNotifyInitialChanel.ts** - 初始通道注册hook
8. **Panel/index.tsx** - 面板基类，处理通道更新

## 4. ObstacleStore/index.tsx 启动时机

### 4.1 启动阶段划分

`ObstacleStore/index.tsx` 的启动分为两个不同的阶段：

#### 阶段1: ObstacleProvider 组件初始化（应用启动时）
#### 阶段2: ObstacleService 数据订阅启动（面板激活时）

### 4.2 详细启动流程

#### 4.2.1 应用启动阶段 - ObstacleProvider 初始化

**时机**: 应用启动时立即执行
**位置**: `App.tsx` → `CombineContext` → `ObstacleProvider`

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/App.tsx
export function App() {
    const Providers = [
        <WebSocketManagerProvider key='WebSocketManagerProvider' />,
        <ObstacleProvider key='ObstacleProvider' />,  // 立即初始化
        // ... 其他Provider
    ];
    
    return (
        <CombineContext providers={Providers}>
            <InitAppData />
            <PageLayout />
        </CombineContext>
    );
}
```

**ObstacleProvider 初始化过程**:
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/ObstacleStore/index.tsx
function ObstacleProviderInner() {
    const { streamApi } = useWebSocketServices();
    
    // 1. 创建ObstacleService实例
    const [obstacleService] = useState(() => 
        streamApi ? new ObstacleService(streamApi) : null
    );
    
    // 2. 提供context给子组件
    return (
        <ObstacleContext.Provider value={{ obstacleService }}>
            {children}
        </ObstacleContext.Provider>
    );
}
```

**注意**: 此时只是创建了ObstacleService实例，但还没有开始订阅数据。

#### 4.2.2 面板激活阶段 - ObstacleService 数据订阅启动

**时机**: 当ObstaclePanel组件被激活且currentChannel设置后
**触发条件**: 
1. 面板被添加到布局中
2. 面板的currentChannel被设置
3. WebSocket连接已建立

**启动过程**:
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/ObstaclePanel/index.tsx
function ObstaclePanel() {
    const { obstacleService } = useObstacleService();
    const { currentChannel } = usePanelContext();
    
    // 当面板激活且currentChannel设置后，开始订阅数据
    useEffect(() => {
        if (obstacleService && currentChannel) {
            // 开始订阅obstacle数据
            const subscription = obstacleService
                .subscribeToObstacles(currentChannel)
                .subscribe(handleObstacleData);
            
            return () => subscription.unsubscribe();
        }
    }, [obstacleService, currentChannel]);
}
```

### 4.3 启动时机总结

| 阶段 | 时机 | 状态 | 说明 |
|------|------|------|------|
| 1 | 应用启动 | ObstacleProvider初始化 | 创建ObstacleService实例，但未订阅数据 |
| 2 | 面板激活 | ObstacleService开始订阅 | 当面板激活且通道设置后，开始订阅obstacle数据 |

## 5. 面板激活和启动时机

### 5.1 面板激活的触发方式

面板激活主要有以下几种方式：

#### 5.1.1 用户主动添加面板
**方式1: 点击添加**
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/MenuDrawer/AddPanel/index.tsx
function DragItem(props: { panel: IPanelMetaInfo }) {
    const handleClickDragItem = () => {
        dispatch(
            addPanelFromOutside({
                mode: hmi.currentMode,
                path: [],
                position: 'left',
                originPanelConfig: props.panel,  // 面板配置
            }),
        );
    };
    
    return (
        <div ref={connectDragSource} onClick={handleClickDragItem}>
            <div className={classes['add-panel-item']}>
                {props.panel.title}
            </div>
        </div>
    );
}
```

**方式2: 拖拽添加**
```typescript
// 拖拽到布局区域
const handleDrop = (item: any, monitor: DropTargetMonitor) => {
    if (monitor.didDrop()) {
        dispatch(
            addPanelFromOutside({
                mode: hmi.currentMode,
                path: [],
                position: 'left',
                originPanelConfig: item.panel,
            }),
        );
    }
};
```

#### 5.1.2 系统自动添加面板
**方式1: 默认布局**
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/PanelLayoutStore/index.tsx
const defaultLayout = {
    left: [
        { id: 'obstacle-panel', type: 'ObstaclePanel' },
        // ... 其他默认面板
    ],
    right: [],
    bottom: []
};
```

**方式2: 模式切换**
```typescript
// 当HMI模式切换时，自动添加对应面板
const handleModeChange = (newMode: string) => {
    dispatch(setCurrentMode(newMode));
    // 根据新模式添加相应面板
    dispatch(addPanelFromOutside({
        mode: newMode,
        path: [],
        position: 'left',
        originPanelConfig: getPanelByMode(newMode),
    }));
};
```

### 5.2 面板激活的详细流程

#### 5.2.1 面板添加流程
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/PanelLayoutStore/index.tsx
function addPanelFromOutside(action: AddPanelFromOutsideAction) {
    const { mode, path, position, originPanelConfig } = action.payload;
    
    // 1. 创建面板实例
    const panelInstance = createPanelInstance(originPanelConfig);
    
    // 2. 添加到布局
    const newLayout = addPanelToLayout(currentLayout, panelInstance, position);
    
    // 3. 更新状态
    dispatch(setLayout(newLayout));
    
    // 4. 触发面板激活
    dispatch(activatePanel(panelInstance.id));
}
```

#### 5.2.2 面板激活流程
```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/PanelLayoutStore/index.tsx
function activatePanel(panelId: string) {
    // 1. 更新激活状态
    dispatch(setActivePanel(panelId));
    
    // 2. 设置当前通道
    const panel = getPanelById(panelId);
    if (panel && panel.channel) {
        dispatch(setCurrentChannel(panel.channel));
    }
    
    // 3. 触发面板渲染
    dispatch(renderPanel(panelId));
}
```

### 5.3 面板启动的关键时机

| 时机 | 触发条件 | 执行内容 |
|------|----------|----------|
| 应用启动 | 系统初始化 | 加载默认布局，但不激活面板 |
| 用户添加面板 | 点击或拖拽 | 创建面板实例，添加到布局，激活面板 |
| 模式切换 | HMI模式改变 | 根据新模式添加相应面板 |
| 面板切换 | 用户点击面板 | 激活目标面板，停用当前面板 |

### 5.4 面板激活后的数据流启动

当面板激活后，会触发以下数据流：

1. **通道设置**: `setCurrentChannel(channelName)`
2. **数据订阅**: `obstacleService.subscribeToObstacles(channelName)`
3. **WebSocket通信**: 向后端发送订阅请求
4. **数据接收**: 接收后端推送的obstacle数据
5. **状态更新**: 更新前端状态，触发UI重渲染

## 总结

整个Apollo Dreamview系统的数据流是一个完整的闭环：

1. **WebSocket连接建立** → 后端自动发送metadata
2. **面板激活** → 前端订阅obstacle数据
3. **后端数据推送** → 通过WebSocket发送obstacle数据
4. **前端数据处理** → 更新状态，渲染UI

这个流程确保了数据的实时性和准确性，为用户提供了完整的自动驾驶数据可视化体验。
