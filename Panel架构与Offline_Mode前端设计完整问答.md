# Apollo Dreamview Plus Panel架构与Offline Mode前端设计完整问答

## 目录
1. [Panel 和 Page Layout 关系分析](#1-panel-和-page-layout-关系分析)
2. [Panel 架构深入解析](#2-panel-架构深入解析)
3. [VehicleViz 面板功能详解](#3-vehicleviz-面板功能详解)
4. [俯视渲染实现原理](#4-俯视渲染实现原理)
5. [Offline Mode 完整设计方案](#5-offline-mode-完整设计方案)

---

## 1. Panel 和 Page Layout 关系分析

### 1.1 架构层次关系

```
App (应用根组件)
├── PageLayout (页面布局容器)
    ├── Menu (菜单栏)
    ├── PanelLayout (面板布局管理器)
    │   └── Mosaic (拖拽布局组件)
    │       └── RenderTile (面板渲染器)
    │           └── Panel (面板容器)
    │               └── PanelComponent (具体面板组件)
    └── BottomBar (底部控制栏)
```

### 1.2 各组件职责分工

#### **PageLayout** - 页面级布局容器
```typescript
// 位置: modules/dreamview_plus/frontend/packages/dreamview-core/src/components/PageLayout/index.tsx

function PageLayout() {
    return (
        <div>
            <div className={c['dv-layout-menu']}>
                <Menu /> {/* 顶部菜单 */}
            </div>
            <div className={c['dv-layout-content']}>
                <div className={c['dv-layout-window']}>
                    <MenuDrawer /> {/* 侧边抽屉 */}
                    <div className={c['dv-layout-panellayout']}>
                        <PanelLayout /> {/* 面板布局区域 */}
                    </div>
                </div>
                <div className={c['dv-layout-playercontrol']}>
                    <BottomBar /> {/* 底部控制栏 */}
                </div>
            </div>
        </div>
    );
}
```

**职责**:
- 管理整个页面的布局结构
- 包含菜单、面板区域、底部控制栏
- 处理页面级别的状态和逻辑
- 管理引导流程和加载状态

#### **PanelLayout** - 面板布局管理器
```typescript
// 位置: modules/dreamview_plus/frontend/packages/dreamview-core/src/components/PanelLayout/index.tsx

function DreamviewWindow() {
    return (
        <Mosaic<string>
            className='my-mosaic'
            mosaicId={mosaicId}
            renderTile={renderTile}  // 渲染每个面板
            value={layout}           // 布局配置
            onChange={onMosaicChange} // 布局变化回调
        />
    );
}
```

**职责**:
- 使用`react-mosaic-component`实现可拖拽的面板布局
- 管理面板的排列、分割、拖拽等操作
- 通过`RenderTile`渲染每个面板
- 处理布局状态的变化和持久化

#### **Panel** - 面板容器组件
```typescript
// 位置: modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/base/Panel.tsx

export default function Panel(panelProps: PanelProps) {
    const { PanelComponent, panelId, subscribeInfo, panelMetaData } = panelProps;
    
    return (
        <PanelWrapper>
            {/* 数据订阅管理 */}
            {/* 全屏功能 */}
            {/* 快捷键注册 */}
            {/* 大小监听 */}
            <PanelComponent panelId={panelId} />
        </PanelWrapper>
    );
}
```

**职责**:
- 为所有面板提供统一的功能抽象
- 管理数据订阅、全屏、快捷键、大小监听等
- 包装具体的业务面板组件
- 提供面板生命周期管理

### 1.3 数据流向关系

#### **状态管理层次**
```
PageLayoutStore (页面级状态)
├── 菜单状态
├── 引导流程状态
└── 页面布局状态

PanelLayoutStore (面板布局状态)
├── 当前布局配置
├── 面板排列信息
└── 拖拽状态

PanelInfoStore (面板信息状态)
├── 面板选中状态
├── 快捷键注册
└── 面板元数据
```

#### **组件通信**
```typescript
// PageLayout 传递给 PanelLayout
<PanelLayout />

// PanelLayout 传递给 RenderTile
const renderTile = useCallback(
    (panelId: string, path: MosaicPath) => 
        <RenderTile panelId={panelId} path={path} />,
    []
);

// RenderTile 传递给 Panel
<PanelComponent panelId={panelId} />
```

### 1.4 渲染流程

#### **初始化流程**
1. **App** 渲染 **PageLayout**
2. **PageLayout** 渲染 **PanelLayout**
3. **PanelLayout** 根据布局配置渲染多个 **RenderTile**
4. **RenderTile** 根据panelId渲染对应的 **Panel**
5. **Panel** 渲染具体的业务组件

#### **动态更新流程**
1. 用户拖拽面板 → **PanelLayout** 更新布局状态
2. 布局变化 → **Mosaic** 重新渲染 **RenderTile**
3. 面板数据更新 → **Panel** 重新渲染内容
4. 状态变化 → 通过Context传递给子组件

### 1.5 关键设计模式

#### **容器-展示模式**
- **PageLayout**: 容器组件，管理页面结构
- **PanelLayout**: 容器组件，管理面板布局
- **Panel**: 容器组件，管理面板功能
- **PanelComponent**: 展示组件，具体业务逻辑

#### **渲染属性模式**
```typescript
// PanelLayout 使用 renderTile 属性渲染面板
<Mosaic
    renderTile={(panelId, path) => <RenderTile panelId={panelId} path={path} />}
/>
```

#### **Context模式**
```typescript
// 通过Context传递面板相关状态
<PanelTileProvider>
    <PanelComponent />
</PanelTileProvider>
```

---

## 2. Panel 架构深入解析

### 2.1 Panel 是项目自定义组件

**Panel不是React自带的，而是这个项目自己开发的组件**。

#### **项目中的Panel定义**
Panel组件定义在：
```
modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/base/Panel.tsx
```

这是一个完全自定义的React组件，不是React框架提供的。

#### **Panel的依赖关系**
Panel组件依赖的是React的基础功能和一些第三方库：

```typescript
import React, { useCallback, useContext, useEffect, useMemo, useRef, useState } from 'react';
import { MosaicContext, MosaicDirection, MosaicWindowActions, MosaicWindowContext } from 'react-mosaic-component';
import { useResizeDetector } from 'react-resize-detector';
import { Subscription } from 'rxjs';
```

### 2.2 项目中的Panel系统架构

这个项目构建了一个完整的面板系统：

```
Panel系统组件层次：
├── Panel (自定义容器组件)
├── RenderTile (自定义渲染器)
├── PanelLayout (自定义布局管理器)
├── PanelCatalogStore (自定义状态管理)
└── 各种具体面板组件 (PncMonitor, VehicleViz等)
```

### 2.3 与React的关系

Panel组件是**基于React构建的**，使用了React的：
- 函数组件和Hooks
- Context API
- 生命周期管理
- 状态管理

但它本身是**业务层面的抽象**，不是React框架的一部分。

### 2.4 项目中的Panel类型定义

```typescript
// 项目自定义的面板类型枚举
export enum PanelType {
    Console = 'console',
    ModuleDelay = 'moduleDelay',
    VehicleViz = 'vehicleViz',
    CameraView = 'cameraView',
    PointCloud = 'pointCloud',
    DashBoard = 'dashBoard',
    PncMonitor = 'pncMonitor',
    Components = 'components',
    MapCollect = 'MapCollect',
    Charts = 'charts',
    TerminalWin = 'terminalWin',
    ObstaclePanel = 'obstaclePanel',
}
```

### 2.5 自定义的Panel功能

Panel组件实现了许多项目特定的功能：

- **数据订阅机制**: 基于WebSocket的数据流订阅
- **布局管理**: 基于react-mosaic-component的拖拽布局
- **全屏功能**: 自定义的全屏进入/退出逻辑
- **快捷键系统**: 基于RxJS的快捷键管理
- **面板状态管理**: 选中、激活等状态管理

### 2.6 为什么要使用Panel而不是直接渲染？

#### **1. 统一的功能抽象**

Panel提供了所有面板都需要的基础功能：

```typescript
// 数据订阅管理
const { isMainConnected, streamApi, metadata } = useWebSocketServices();

// 全屏功能
const { enterFullScreen, exitFullScreen, fullScreenFnObj } = usePanelTileContext();

// 快捷键注册
const setKeyDownHandlers = useCallback((handlers: KeyHandlers[]) => {
    // 注册快捷键逻辑
}, []);

// 面板大小监听
const { ref: panelRootRef } = useResizeDetector({
    onResize: (width: number, height: number) => {
        // 处理大小变化
    },
});
```

#### **2. 数据订阅机制**

Panel提供了统一的数据订阅接口，支持：
- 声明式数据订阅配置
- 动态添加/更新/取消订阅
- 自动处理WebSocket连接状态
- 数据缓存和状态管理

```typescript
// 面板初始化订阅
useEffect(() => {
    if (subscribeInfo) {
        for (const curSubcibe of subscribeInfo) {
            const connectedSubj = handleSubscribe(curSubcibe);
            // 处理订阅逻辑
        }
    }
}, [metadata, isMainConnected]);
```

#### **3. 布局管理能力**

通过Mosaic组件实现：
- 拖拽调整面板大小
- 面板分割和合并
- 布局状态持久化
- 响应式布局适配

#### **4. 错误边界和加载状态**

```typescript
<ErrorBoundary>
    <React.Suspense fallback={<Spin />}>
        <PanelComponent panelId={panelId} />
    </React.Suspense>
</ErrorBoundary>
```

#### **5. 面板生命周期管理**

- 面板选中/激活状态
- 全屏进入/退出钩子
- 面板销毁时的资源清理
- 快捷键注册和注销

#### **6. 模块化和可扩展性**

Panel系统支持：
- 动态面板注册 (`PanelCatalogStore`)
- 面板类型枚举 (`PanelType`)
- 远程面板和本地面板的统一管理
- 面板工具栏的独立渲染

#### **7. 性能优化**

- 使用 `React.memo` 优化渲染
- 懒加载面板组件 (`React.lazy`)
- 订阅数据的按需更新
- 面板大小变化的高效处理

### 2.7 总结

使用Panel架构而不是直接渲染的主要原因是：

1. **功能复用**: 避免在每个面板组件中重复实现数据订阅、全屏、快捷键等通用功能
2. **统一管理**: 通过Panel容器统一管理面板的生命周期、状态和交互
3. **布局能力**: 提供强大的拖拽布局和响应式适配能力
4. **错误处理**: 统一的错误边界和加载状态管理
5. **扩展性**: 支持动态面板注册和模块化开发
6. **性能**: 通过优化策略提升整体性能

这种架构设计使得每个具体的面板组件只需要关注自己的业务逻辑，而将通用的面板功能交给Panel容器来处理，实现了关注点分离和代码复用。

---

## 3. VehicleViz 面板功能详解

### 3.1 主要功能

#### **3D车辆可视化**
- 显示自动驾驶车辆的位置、姿态和运动状态
- 实时渲染车辆在3D环境中的位置
- 支持多种视角切换（默认、近景、俯视、地图视图）

#### **感知数据可视化**
```typescript
Perception: {
    polygon: { /* 多边形检测框 */ },
    boundingbox: { /* 3D边界框 */ },
    pointCloud: { /* 点云数据 */ },
    curbPointCloud: { /* 路沿点云 */ },
    vehicle: { /* 车辆检测 */ },
    pedestrian: { /* 行人检测 */ },
    bicycle: { /* 自行车检测 */ },
    unknownMovable: { /* 未知移动物体 */ },
    unknownStationary: { /* 未知静止物体 */ },
    // ... 更多感知元素
}
```

#### **预测数据可视化**
```typescript
Prediction: {
    majorPredictionLine: { /* 主要预测轨迹 */ },
    minorPredictionLine: { /* 次要预测轨迹 */ },
    gaussianInfo: { /* 高斯分布信息 */ },
    obstaclePriority: { /* 障碍物优先级 */ },
    interactiveTag: { /* 交互标签 */ }
}
```

#### **规划数据可视化**
```typescript
Planning: {
    planningTrajectoryLine: { /* 规划轨迹线 */ },
    planningReferenceLine: { /* 参考线 */ },
    planningBoundaryLine: { /* 边界线 */ },
    planningCar: { /* 规划车辆 */ }
}
```

#### **地图数据可视化**
```typescript
Map: {
    lane: { /* 车道线 */ },
    crosswalk: { /* 人行横道 */ },
    signal: { /* 交通信号灯 */ },
    stopSign: { /* 停车标志 */ },
    yieldSign: { /* 让行标志 */ },
    junction: { /* 路口 */ },
    clearArea: { /* 清障区域 */ }
}
```

#### **定位数据可视化**
```typescript
Position: {
    localization: { /* 定位信息 */ },
    gps: { /* GPS数据 */ },
    shadow: { /* 阴影轨迹 */ }
}
```

#### **决策数据可视化**
```typescript
Decision: {
    mainDecision: { /* 主要决策 */ },
    obstacleDecision: { /* 障碍物决策 */ }
}
```

#### **路径规划可视化**
```typescript
Routing: {
    routingLine: { /* 路径规划线 */ }
}
```

### 3.2 交互功能

#### **视角控制**
- **D键**: 切换视角（默认/近景/俯视/地图）
- **Ctrl/Cmd + =**: 放大视图
- **Ctrl/Cmd + -**: 缩小视图
- 鼠标拖拽旋转视角

#### **图层管理**
- 通过LayerMenu可以控制各种可视化元素的显示/隐藏
- 支持点云通道切换
- 实时调整可视化参数

#### **性能监控**
- 显示FPS（帧率）
- 显示渲染三角形数量
- 性能优化和调试信息

#### **数据订阅**
- 实时订阅WebSocket数据流
- 支持多种数据通道切换
- 自动处理数据更新和渲染

### 3.3 技术特点

1. **基于Three.js**: 使用WebGL进行3D渲染
2. **实时数据流**: 通过WebSocket接收实时数据
3. **模块化设计**: 支持动态加载和配置
4. **高性能渲染**: 使用requestIdleCallback优化渲染性能
5. **响应式布局**: 支持面板拖拽和大小调整

### 3.4 使用场景

VehicleViz面板主要用于：
- **自动驾驶测试**: 实时监控车辆状态和周围环境
- **算法调试**: 可视化感知、预测、规划算法的输出结果
- **数据回放**: 回放和分析历史驾驶数据
- **系统监控**: 监控自动驾驶系统的运行状态
- **教学演示**: 展示自动驾驶技术的工作原理

这是Apollo Dreamview Plus中最重要的可视化工具，为开发者和测试人员提供了直观的3D环境来理解和调试自动驾驶系统。

---

## 4. 俯视渲染实现原理

### 4.1 视角切换机制

俯视渲染通过`View`类实现，当用户点击"Overhead"选项时：

```typescript
// ViewMenu组件中的切换逻辑
const changeView = (viewType: string) => {
    setView(viewType);
    setCurrentView(viewType);
    carviz.view.changeViewType(viewType); // 调用View类的changeViewType方法
};
```

### 4.2 相机参数配置

俯视视角有专门的相机参数配置：

```typescript
// cameraParams配置
export const cameraParams = {
    Overhead: {
        fov: 60,        // 视野角度60度
        near: 1,        // 近裁剪面距离1米
        far: 100,       // 远裁剪面距离100米
    },
    // ... 其他视角配置
};
```

### 4.3 相机位置计算

俯视渲染的核心在于相机位置和朝向的计算：

```typescript
public setView() {
    if (!this.adc) return;
    
    const target = this.adc?.adc; // 获取车辆位置
    const { x = 0, y = 0, z = 0 } = target?.position || {};
    const rotationY = target?.rotation.y || 0;
    
    // 计算相机偏移量
    const offsetX = this.overheadViewDistance * Math.cos(rotationY) * Math.cos(this.viewAngle);
    const offsetY = this.overheadViewDistance * Math.sin(rotationY) * Math.cos(this.viewAngle);
    const offsetZ = this.overheadViewDistance * Math.sin(this.viewAngle);
    
    switch (this.viewType) {
        case 'Overhead':
            // 俯视视角的相机设置
            this.camera.position.set(x, y, z + offsetZ);           // 相机位置：车辆正上方
            this.camera.up.set(0, 1, 0);                          // 相机上方向：Y轴正方向
            this.camera.lookAt(x, y + offsetY / 8, z);            // 相机朝向：稍微向前看
            this.controls.enabled = false;                        // 禁用手动控制
            break;
    }
}
```

### 4.4 俯视渲染的关键参数

#### **距离参数**
```typescript
private overheadViewDistance = 90; // 俯视距离90米
private viewAngle = 0.8;           // 视角角度约45.8度
```

#### **相机位置计算**
- **X坐标**: `x` (与车辆X坐标相同)
- **Y坐标**: `y` (与车辆Y坐标相同)  
- **Z坐标**: `z + offsetZ` (车辆Z坐标 + 90米高度)

#### **相机朝向**
- **目标点**: `(x, y + offsetY / 8, z)` (车辆位置稍微向前偏移)
- **上方向**: `(0, 1, 0)` (Y轴正方向，确保是俯视)

### 4.5 俯视渲染的视觉效果

#### **高度优势**
- 相机距离地面90米高度，提供鸟瞰视角
- 可以清楚看到车辆周围的大范围环境

#### **视角特点**
- **FOV 60度**: 提供适中的视野范围
- **近远裁剪面**: 1-100米，适合观察中近距离场景
- **固定朝向**: 相机始终向下看，提供稳定的俯视效果

#### **渲染范围**
- 可以同时看到车辆前方、后方、左右两侧的环境
- 适合观察车道线、交通标志、其他车辆等

### 4.6 与其他视角的对比

```typescript
// 默认视角 - 第三人称跟随
case 'Default':
    this.camera.position.set(x - offsetX, y - offsetY, z + offsetZ);
    this.camera.up.set(0, 0, 1);
    this.camera.lookAt(x + offsetX, y + offsetY, 0);

// 俯视视角 - 正上方俯视
case 'Overhead':
    this.camera.position.set(x, y, z + offsetZ);
    this.camera.up.set(0, 1, 0);
    this.camera.lookAt(x, y + offsetY / 8, z);
```

### 4.7 俯视渲染的优势

1. **全局视野**: 可以同时观察车辆周围360度环境
2. **路径规划**: 清楚看到规划轨迹和参考线
3. **障碍物检测**: 便于观察感知系统检测到的各种障碍物
4. **地图信息**: 可以同时看到车道线、交通标志等地图元素
5. **调试分析**: 适合算法调试和数据分析

### 4.8 技术实现细节

- **基于Three.js**: 使用WebGL进行3D渲染
- **实时更新**: 相机位置跟随车辆位置实时更新
- **性能优化**: 通过`requestIdleCallback`优化渲染性能
- **状态管理**: 视角状态持久化到localStorage

俯视渲染通过将相机放置在车辆正上方90米高度，并设置合适的朝向和视野参数，实现了类似无人机俯拍的视觉效果，为自动驾驶系统的监控和调试提供了重要的可视化工具。

---

## 5. Offline Mode 完整设计方案

### 5.1 整体架构设计

#### **5.1.1 新增HMI模式**
首先需要在HMI系统中添加offline mode：

```typescript
// modules/common_msgs/dreamview_msgs/hmi_status.proto
enum HMIModeOperation {
  // ... 现有模式
  OFFLINE_ANALYSIS = 8;  // 离线分析模式
}
```

#### **5.1.2 前端架构设计**

```
OfflineAnalysis (统一Panel)
├── DatabaseConnection (数据库连接)
├── DataQuery (数据查询)
├── TimelinePlayer (时间轴播放)
├── ObstacleViewer (障碍物查看)
└── DataStatistics (数据分析)
```

### 5.2 后端API设计

#### **5.2.1 数据库连接API**

```typescript
// 后端API接口设计
interface DatabaseConfig {
  type: 'mysql' | 'postgresql' | 'mongodb';
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
}

interface DatabaseAPI {
  // 连接数据库
  connectDatabase(config: DatabaseConfig): Promise<boolean>;
  
  // 获取可用数据库列表
  getAvailableDatabases(): Promise<DatabaseConfig[]>;
  
  // 获取数据表列表
  getTables(): Promise<string[]>;
  
  // 查询障碍物数据
  queryObstacles(params: QueryParams): Promise<ObstacleData[]>;
  
  // 获取时间范围
  getTimeRange(tableName: string): Promise<{start: number, end: number}>;
}
```

#### **5.2.2 数据查询参数**

```typescript
interface QueryParams {
  tableName: string;
  startTime: number;
  endTime: number;
  vehicleId?: string;
  obstacleTypes?: string[];
  limit?: number;
  offset?: number;
}

interface ObstacleData {
  id: string;
  timestamp: number;
  vehicleId: string;
  obstacleType: string;
  position: {
    x: number;
    y: number;
    z: number;
  };
  velocity: {
    x: number;
    y: number;
    z: number;
  };
  heading: number;
  length: number;
  width: number;
  height: number;
  confidence: number;
  polygon: Point2D[];
}
```

### 5.3 前端组件设计

#### **5.3.1 统一的OfflineAnalysis Panel**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineAnalysis/index.tsx
import React, { useState, useEffect, useMemo } from 'react';
import { Tabs, Card, Row, Col, message } from '@dreamview/dreamview-ui';
import { Panel } from '../base/Panel';
import DatabaseConnection from './components/DatabaseConnection';
import DataQuery from './components/DataQuery';
import TimelinePlayer from './components/TimelinePlayer';
import ObstacleViewer from './components/ObstacleViewer';
import DataStatistics from './components/DataStatistics';
import { useOfflineAnalysisStore } from '../../store/OfflineAnalysisStore';

function OfflineAnalysisInner() {
  const { state, actions } = useOfflineAnalysisStore();
  const [activeTab, setActiveTab] = useState('connection');

  const tabItems = [
    {
      key: 'connection',
      label: '数据库连接',
      children: (
        <DatabaseConnection
          onConnect={actions.connectDatabase}
          onDisconnect={actions.disconnectDatabase}
          connected={!!state.database}
          loading={state.loading}
        />
      ),
    },
    {
      key: 'query',
      label: '数据查询',
      children: (
        <DataQuery
          database={state.database}
          onQuery={actions.queryData}
          loading={state.loading}
          disabled={!state.database}
        />
      ),
    },
    {
      key: 'player',
      label: '时间轴播放',
      children: (
        <TimelinePlayer
          data={state.obstacleData}
          currentTime={state.currentTime}
          onTimeChange={actions.setCurrentTime}
          onPlay={actions.togglePlay}
          onSpeedChange={actions.setPlaySpeed}
          playing={state.playing}
          disabled={state.obstacleData.length === 0}
        />
      ),
    },
    {
      key: 'viewer',
      label: '障碍物查看',
      children: (
        <ObstacleViewer
          data={state.obstacleData}
          currentTime={state.currentTime}
          onObstacleSelect={actions.selectObstacle}
          selectedObstacle={state.selectedObstacle}
        />
      ),
    },
    {
      key: 'statistics',
      label: '数据分析',
      children: (
        <DataStatistics
          data={state.obstacleData}
          currentTime={state.currentTime}
          selectedObstacle={state.selectedObstacle}
        />
      ),
    },
  ];

  return (
    <div className="offline-analysis-panel">
      <Card title="离线数据分析" className="offline-analysis-card">
        <Tabs
          activeKey={activeTab}
          onChange={setActiveTab}
          items={tabItems}
          size="small"
        />
      </Card>
    </div>
  );
}

function OfflineAnalysis(props: any) {
  const Component = useMemo(
    () => Panel({
      PanelComponent: OfflineAnalysisInner,
      panelId: props.panelId,
      subscribeInfo: [], // 不需要实时数据订阅
    }),
    []
  );

  return <Component {...props} />;
}

export default React.memo(OfflineAnalysis);
```

#### **5.3.2 子组件结构**

```
OfflineAnalysis/
├── index.tsx                    # 主Panel组件
├── components/
│   ├── DatabaseConnection.tsx   # 数据库连接组件
│   ├── DataQuery.tsx           # 数据查询组件
│   ├── TimelinePlayer.tsx      # 时间轴播放组件
│   ├── ObstacleViewer.tsx      # 障碍物查看组件
│   └── DataStatistics.tsx      # 数据分析组件
└── styles/
    └── index.less              # 样式文件
```

#### **5.3.3 数据库连接组件示例**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineAnalysis/components/DatabaseConnection.tsx
import React, { useState } from 'react';
import { Form, Select, Input, Button, message, Space } from '@dreamview/dreamview-ui';

interface DatabaseConnectionProps {
  onConnect: (config: DatabaseConfig) => void;
  onDisconnect: () => void;
  connected: boolean;
  loading: boolean;
}

function DatabaseConnection({ onConnect, onDisconnect, connected, loading }: DatabaseConnectionProps) {
  const [formData, setFormData] = useState<DatabaseConfig>({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    database: '',
    username: '',
    password: ''
  });

  const handleConnect = () => {
    if (!formData.host || !formData.database || !formData.username) {
      message.warning('请填写完整的连接信息');
      return;
    }
    onConnect(formData);
  };

  return (
    <div className="database-connection">
      <Form layout="vertical" size="small">
        <Row gutter={16}>
          <Col span={8}>
            <Form.Item label="数据库类型">
              <Select
                value={formData.type}
                onChange={(value) => setFormData({...formData, type: value})}
                disabled={connected}
              >
                <Select.Option value="mysql">MySQL</Select.Option>
                <Select.Option value="postgresql">PostgreSQL</Select.Option>
                <Select.Option value="mongodb">MongoDB</Select.Option>
              </Select>
            </Form.Item>
          </Col>
          <Col span={8}>
            <Form.Item label="主机地址">
              <Input
                value={formData.host}
                onChange={(e) => setFormData({...formData, host: e.target.value})}
                disabled={connected}
              />
            </Form.Item>
          </Col>
          <Col span={8}>
            <Form.Item label="端口">
              <Input
                type="number"
                value={formData.port}
                onChange={(e) => setFormData({...formData, port: parseInt(e.target.value)})}
                disabled={connected}
              />
            </Form.Item>
          </Col>
        </Row>
        
        <Row gutter={16}>
          <Col span={8}>
            <Form.Item label="数据库名">
              <Input
                value={formData.database}
                onChange={(e) => setFormData({...formData, database: e.target.value})}
                disabled={connected}
              />
            </Form.Item>
          </Col>
          <Col span={8}>
            <Form.Item label="用户名">
              <Input
                value={formData.username}
                onChange={(e) => setFormData({...formData, username: e.target.value})}
                disabled={connected}
              />
            </Form.Item>
          </Col>
          <Col span={8}>
            <Form.Item label="密码">
              <Input.Password
                value={formData.password}
                onChange={(e) => setFormData({...formData, password: e.target.value})}
                disabled={connected}
              />
            </Form.Item>
          </Col>
        </Row>
        
        <Form.Item>
          <Space>
            <Button
              type="primary"
              onClick={handleConnect}
              loading={loading}
              disabled={connected}
            >
              {connected ? '已连接' : '连接数据库'}
            </Button>
            {connected && (
              <Button onClick={onDisconnect}>
                断开连接
              </Button>
            )}
          </Space>
        </Form.Item>
      </Form>
    </div>
  );
}

export default DatabaseConnection;
```

### 5.4 状态管理设计

#### **5.4.1 OfflineAnalysis Store**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/OfflineAnalysisStore/index.tsx
import { createContext, useContext, useReducer, ReactNode } from 'react';

interface OfflineAnalysisState {
  database: DatabaseConfig | null;
  obstacleData: ObstacleData[];
  currentTime: number;
  selectedObstacle: ObstacleData | null;
  playing: boolean;
  playSpeed: number;
  loading: boolean;
  error: string | null;
}

type OfflineAnalysisAction = 
  | { type: 'CONNECT_DATABASE'; payload: DatabaseConfig }
  | { type: 'DISCONNECT_DATABASE' }
  | { type: 'SET_OBSTACLE_DATA'; payload: ObstacleData[] }
  | { type: 'SET_CURRENT_TIME'; payload: number }
  | { type: 'SELECT_OBSTACLE'; payload: ObstacleData | null }
  | { type: 'TOGGLE_PLAY' }
  | { type: 'SET_PLAY_SPEED'; payload: number }
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'SET_ERROR'; payload: string | null };

const initialState: OfflineAnalysisState = {
  database: null,
  obstacleData: [],
  currentTime: 0,
  selectedObstacle: null,
  playing: false,
  playSpeed: 1,
  loading: false,
  error: null
};

function offlineAnalysisReducer(state: OfflineAnalysisState, action: OfflineAnalysisAction): OfflineAnalysisState {
  switch (action.type) {
    case 'CONNECT_DATABASE':
      return { ...state, database: action.payload, error: null };
    case 'DISCONNECT_DATABASE':
      return { ...state, database: null, obstacleData: [], currentTime: 0, selectedObstacle: null };
    case 'SET_OBSTACLE_DATA':
      return { ...state, obstacleData: action.payload };
    case 'SET_CURRENT_TIME':
      return { ...state, currentTime: action.payload };
    case 'SELECT_OBSTACLE':
      return { ...state, selectedObstacle: action.payload };
    case 'TOGGLE_PLAY':
      return { ...state, playing: !state.playing };
    case 'SET_PLAY_SPEED':
      return { ...state, playSpeed: action.payload };
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    case 'SET_ERROR':
      return { ...state, error: action.payload };
    default:
      return state;
  }
}

const OfflineAnalysisContext = createContext<{
  state: OfflineAnalysisState;
  actions: {
    connectDatabase: (config: DatabaseConfig) => void;
    disconnectDatabase: () => void;
    queryData: (params: QueryParams) => void;
    setCurrentTime: (time: number) => void;
    selectObstacle: (obstacle: ObstacleData | null) => void;
    togglePlay: () => void;
    setPlaySpeed: (speed: number) => void;
  };
} | null>(null);

export function OfflineAnalysisProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(offlineAnalysisReducer, initialState);
  
  const actions = {
    connectDatabase: (config: DatabaseConfig) => {
      dispatch({ type: 'CONNECT_DATABASE', payload: config });
    },
    disconnectDatabase: () => {
      dispatch({ type: 'DISCONNECT_DATABASE' });
    },
    queryData: (params: QueryParams) => {
      dispatch({ type: 'SET_LOADING', payload: true });
      // API调用逻辑
      fetch('/api/offline/query', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(params)
      })
      .then(response => response.json())
      .then(data => {
        dispatch({ type: 'SET_OBSTACLE_DATA', payload: data });
        dispatch({ type: 'SET_LOADING', payload: false });
      })
      .catch(error => {
        dispatch({ type: 'SET_ERROR', payload: error.message });
        dispatch({ type: 'SET_LOADING', payload: false });
      });
    },
    setCurrentTime: (time: number) => {
      dispatch({ type: 'SET_CURRENT_TIME', payload: time });
    },
    selectObstacle: (obstacle: ObstacleData | null) => {
      dispatch({ type: 'SELECT_OBSTACLE', payload: obstacle });
    },
    togglePlay: () => {
      dispatch({ type: 'TOGGLE_PLAY' });
    },
    setPlaySpeed: (speed: number) => {
      dispatch({ type: 'SET_PLAY_SPEED', payload: speed });
    },
  };
  
  return (
    <OfflineAnalysisContext.Provider value={{ state, actions }}>
      {children}
    </OfflineAnalysisContext.Provider>
  );
}

export function useOfflineAnalysisStore() {
  const context = useContext(OfflineAnalysisContext);
  if (!context) {
    throw new Error('useOfflineAnalysisStore must be used within OfflineAnalysisProvider');
  }
  return context;
}
```

### 5.5 需要修改的文件

#### **5.5.1 添加新的Panel类型**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/type/Panel.ts
export enum PanelType {
    Console = 'console',
    ModuleDelay = 'moduleDelay',
    VehicleViz = 'vehicleViz',
    CameraView = 'cameraView',
    PointCloud = 'pointCloud',
    DashBoard = 'dashBoard',
    PncMonitor = 'pncMonitor',
    Components = 'components',
    MapCollect = 'MapCollect',
    Charts = 'charts',
    TerminalWin = 'terminalWin',
    ObstaclePanel = 'obstaclePanel',
    // 新增
    OfflineAnalysis = 'offlineAnalysis',  // 离线分析模式
}
```

#### **5.5.2 注册新的Panel**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/index.tsx
const localPanels: IPanelMetaInfo[] = [
    // ... 现有panels
    {
        title: t('offlineAnalysisTitle'),
        type: PanelType.OfflineAnalysis,
        thumbnail: offlineAnalysisIllustratorImg,
        description: t('offlineAnalysisDescription'),
        module: () => import('@dreamview/dreamview-core/src/components/panels/OfflineAnalysis'),
    },
];
```

#### **5.5.3 添加HMI模式**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actionTypes.ts
export enum HMIModeOperation {
    // ... 现有操作
    OFFLINE_ANALYSIS = 'OFFLINE_ANALYSIS',
}
```

#### **5.5.4 App.tsx修改**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-core/src/App.tsx

export function App() {
    const Providers = [
        // ... 现有Providers
        <OfflineAnalysisProvider key='OfflineAnalysisProvider' />, // 新增
    ];

    return (
        <ThemeProvider>
            <DndProvider backend={HTML5Backend}>
                <GlobalStyles />
                <CombineContext providers={Providers}>
                    <InitAppData />
                    <PageLayout />
                </CombineContext>
            </DndProvider>
        </ThemeProvider>
    );
}
```

#### **5.5.5 国际化文件**

```typescript
// modules/dreamview_plus/frontend/packages/dreamview-lang/zh/panels.ts
export default {
    // ... 现有翻译
    offlineAnalysisTitle: '离线分析',
    offlineAnalysisDescription: '从数据库获取历史数据并分析检测到的物体',
};

// modules/dreamview_plus/frontend/packages/dreamview-lang/en/panels.ts
export default {
    // ... 现有翻译
    offlineAnalysisTitle: 'Offline Analysis',
    offlineAnalysisDescription: 'Retrieve historical data from database and analyze detected objects',
};
```

### 5.6 后端API实现

#### **5.6.1 数据库连接服务**

```cpp
// modules/dreamview_plus/backend/hmi/offline_analysis_service.h
class OfflineAnalysisService {
public:
  bool ConnectDatabase(const DatabaseConfig& config);
  std::vector<std::string> GetTables();
  std::vector<ObstacleData> QueryObstacles(const QueryParams& params);
  TimeRange GetTimeRange(const std::string& tableName);
  
private:
  std::unique_ptr<DatabaseConnection> db_connection_;
};
```

### 5.7 数据流设计

```
用户操作 → 前端组件 → OfflineAnalysisStore → API调用 → 后端服务 → 数据库
    ↓
数据返回 → 状态更新 → 组件重新渲染 → 可视化显示
```

### 5.8 架构优势

#### **为什么不需要修改PanelLayout和PageLayout？**

1. **松耦合设计**: 现有架构采用了松耦合的设计，新Panel可以无缝集成
2. **动态渲染**: Mosaic组件支持动态渲染，通过PanelCatalogStore管理Panel
3. **状态隔离**: 每个Panel有独立的状态管理，不会影响整体布局
4. **可扩展性**: 架构设计时就考虑了扩展性，新功能可以轻松添加

#### **集成流程**

```
1. 创建OfflineAnalysis组件
   ↓
2. 注册到PanelCatalogStore
   ↓  
3. 添加到localPanels数组
   ↓
4. 用户通过Menu选择Panel
   ↓
5. Mosaic自动渲染Panel
   ↓
6. PanelLayout自动管理布局
```

### 5.9 总结

**新增的Panel组件**：
- 1个主Panel：`OfflineAnalysis`
- 5个子组件：`DatabaseConnection`, `DataQuery`, `TimelinePlayer`, `ObstacleViewer`, `DataStatistics`

**需要修改的文件**：
- ✅ `panels/index.tsx` - 注册新Panel
- ✅ `panels/type/Panel.ts` - 添加Panel类型
- ✅ `App.tsx` - 添加状态管理Provider
- ✅ 国际化文件 - 添加翻译
- ❌ `PanelLayout` - **不需要修改**
- ❌ `PageLayout` - **不需要修改**

这种设计充分利用了现有架构的扩展性，新功能可以无缝集成而不影响现有代码。通过统一的标签页界面，用户可以方便地进行数据库连接、数据查询、时间轴播放、障碍物查看和数据分析等操作。

---

## 总结

本文档详细分析了Apollo Dreamview Plus的前端架构，包括：

1. **Panel和PageLayout的关系**：三层嵌套的容器架构，实现了关注点分离
2. **Panel架构深入解析**：自定义的业务组件，提供统一的面板功能抽象
3. **VehicleViz面板功能**：核心的3D可视化工具，支持多种数据类型的可视化
4. **俯视渲染实现**：基于Three.js的相机控制，实现鸟瞰视角效果
5. **Offline Mode设计**：完整的离线数据分析方案，支持数据库连接和历史数据回放

整个架构设计体现了现代前端开发的最佳实践，具有良好的可扩展性、可维护性和用户体验。
