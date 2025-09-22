# Offline Mode 完整实现文档

## 概述

本文档整理了 Apollo Dreamview Plus 中 Offline Mode 的完整实现，包括已存在的代码和需要修改的文件。Offline Mode 仿照 Perception Mode 的结构，包含交互式面板和障碍物可视化面板。

## 已存在的代码文件

### 1. 类型定义和枚举

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/HmiStore/actionTypes.ts`
```typescript
export enum CURRENT_MODE {
    NONE = 'none',
    DEFAULT = 'Default',
    PERCEPTION = 'Perception',
    PNC = 'Pnc',
    VEHICLE_TEST = 'Vehicle Test',
    MAP_COLLECT = 'Map Collect',
    MAP_EDITOR = 'Map Editor',
    CAMERA_CALIBRATION = 'Camera Calibration',
    LiDAR_CALIBRATION = 'Lidar Calibration',
    DYNAMICS_CALIBRATION = 'Dynamics Calibration',
    CANBUS_DEBUG = 'Canbus Debug',
    OFFLINE_ANALYSIS = 'Offline Analysis', // ✅ 已存在
}
```

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/type/Panel.ts`
```typescript
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
    OfflineAnalysis = 'offlineAnalysis', // ✅ 已存在
    OfflineObstacleViz = 'offlineObstacleViz', // ✅ 已存在
}
```

### 2. 面板注册

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/index.tsx`
```typescript
const localPanels: IPanelMetaInfo[] = [
    // ... 其他面板 ...
    {
        title: t('offlineAnalysisTitle'),
        type: PanelType.OfflineAnalysis,
        thumbnail: consoleIllustratorImg,
        description: t('offlineAnalysisDescription'),
        module: () => import('@dreamview/dreamview-core/src/components/panels/OfflineAnalysis'),
    },
    {
        title: t('offlineObstacleVizTitle'),
        type: PanelType.OfflineObstacleViz,
        thumbnail: vehicleVizIllustratorImg,
        description: t('offlineObstacleVizDescription'),
        module: () => import('@dreamview/dreamview-core/src/components/panels/OfflineObstacleViz'),
    },
];
```

### 3. 状态管理

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/OfflineAnalysisStore/index.tsx`
```typescript
// 类型定义
export interface DatabaseConfig {
    type: 'postgresql' | 'mysql' | 'mongodb';
    host: string;
    port: number;
    database: string;
    username: string;
    password: string;
    connected?: boolean;
}

export interface ObstacleData {
    id: string;
    type: 'vehicle' | 'pedestrian' | 'cyclist' | 'unknown';
    positionX: number;
    positionY: number;
    positionZ: number;
    confidence: number;
    timestamp: number;
    trajectory?: Array<{ x: number; y: number; z: number }>;
    prediction?: Array<{ x: number; y: number; z: number }>;
}

// Store Provider 和 Hook
export function OfflineAnalysisProvider({ children }: { children: ReactNode }) {
    // ... 实现
}

export function useOfflineAnalysisStore() {
    // ... 实现
}
```

### 4. 面板组件

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineAnalysis/index.tsx`
```typescript
// 包含 5 个功能组件：
// - DatabaseConnection: 数据库连接
// - DataQuery: 数据查询
// - TimelinePlayer: 时间轴播放
// - ObstacleViewer: 障碍物查看
// - DataStatistics: 数据分析

function OfflineAnalysisInner() {
    const [activeTab, setActiveTab] = useState('connection');
    // ... 实现
}

function OfflineAnalysis(props: any) {
    // ... 实现
}
```

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineObstacleViz/index.tsx`
```typescript
// 3D 可视化组件
function ObstacleVisualization() {
    // Canvas 2D 渲染
    // 支持交互选择、轨迹显示、预测显示
}

function OfflineObstacleVizInner() {
    // ... 实现
}

function OfflineObstacleViz(props: any) {
    // ... 实现
}
```

### 5. 样式文件

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineAnalysis/style.less`
```less
.offline-analysis-panel {
    height: 100%;
    padding: 16px;
    background: #fafafa;
    // ... 样式定义
}
```

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/panels/OfflineObstacleViz/style.less`
```less
.offline-obstacle-viz-panel {
    height: 100%;
    padding: 16px;
    background: #fafafa;
    // ... 样式定义
}
```

### 6. 应用集成

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/App.tsx`
```typescript
export function App() {
    const Providers = [
        // ... 其他 Provider ...
        <OfflineAnalysisProvider key='OfflineAnalysisProvider' />, // ✅ 已存在
    ];
    // ... 实现
}
```

### 7. 欢迎引导

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/WelcomeGuide/index.tsx`
```typescript
const WelcomeGuideTabsItem = [
    // ... 其他模式 ...
    {
        key: CURRENT_MODE.OFFLINE_ANALYSIS,
        label: 'Offline Analysis Mode',
        children: (
            <WelcomeGuideTabsContent
                contentImageUrl={welcomeGuideTabsImageDefault}
                clickEnterThisMode={clickEnterThisMode}
                currentMode={CURRENT_MODE.OFFLINE_ANALYSIS}
            />
        ),
    },
];
```

### 8. 默认布局

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/config/defaultLayouts.ts`
```typescript
// Offline Analysis 模式的布局
const offlineAnalysisLayout: MosaicNode<string> = {
    direction: 'row',
    first: {
        direction: 'column',
        first: generatePanelId(PanelType.OfflineObstacleViz, 0),
        second: generatePanelId(PanelType.Console, 0),
        splitPercentage: 80,
    },
    second: {
        direction: 'column',
        first: generatePanelId(PanelType.OfflineAnalysis, 0),
        second: {
            direction: 'row',
            first: generatePanelId(PanelType.Charts, 0),
            second: generatePanelId(PanelType.ObstaclePanel, 0),
            splitPercentage: 50,
        },
        splitPercentage: 60,
    },
    splitPercentage: 60,
};

export const DEFAULT_LAYOUTS: Record<CURRENT_MODE, MosaicNode<string> | null> = {
    // ... 其他模式 ...
    [CURRENT_MODE.OFFLINE_ANALYSIS]: offlineAnalysisLayout, // ✅ 已存在
};
```

### 9. 布局管理

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/store/PanelLayoutStore/index.tsx`
```typescript
export function useGetCurrentLayout() {
    const [hmi] = usePickHmiStore();
    const [store] = usePanelLayoutStore();
    const prevLayout = useRef(null);
    const currentLayout = store.layout[hmi.currentMode]?.layout;
    
    if (!currentLayout) {
        // 如果没有当前布局，尝试获取默认布局
        const defaultLayout = getDefaultLayout(hmi.currentMode);
        if (defaultLayout) {
            return defaultLayout;
        }
        return prevLayout.current;
    }

    prevLayout.current = currentLayout;
    return currentLayout;
}
```

## 需要修改的文件

### 1. PageLayout 组件

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/components/PageLayout/index.tsx`

**需要添加的修改：**

```typescript
// 在 clickEnterThisModeToGuide 函数中添加 OFFLINE_ANALYSIS 模式处理
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
        // 添加这一行：
        if (useGuideMode === CURRENT_MODE.OFFLINE_ANALYSIS) {
            setEnterGuideState({ currentMode: CURRENT_MODE.OFFLINE_ANALYSIS });
        }
    },
    [isMainConnected],
);

// 在渲染部分添加（可选，如果需要引导的话）：
{enterGuideState.currentMode === CURRENT_MODE.OFFLINE_ANALYSIS && (
    <div>Offline Analysis Guide (可选)</div>
)}
```

### 2. 国际化文件

#### `modules/dreamview_plus/frontend/packages/dreamview-lang/zh/panels.ts`

**需要添加的翻译：**

```typescript
export const panels = {
    // ... 现有翻译 ...
    
    // 添加离线分析相关翻译
    offlineAnalysisTitle: '离线分析',
    offlineAnalysisDescription: '从数据库获取历史数据并分析检测到的物体',
    offlineObstacleVizTitle: '离线障碍物可视化', 
    offlineObstacleVizDescription: '可视化显示离线数据中的障碍物信息',
};
```

#### `modules/dreamview_plus/frontend/packages/dreamview-lang/en/panels.ts`

**需要添加的翻译：**

```typescript
export const panels = {
    // ... 现有翻译 ...
    
    // 添加离线分析相关翻译
    offlineAnalysisTitle: 'Offline Analysis',
    offlineAnalysisDescription: 'Retrieve and analyze detected objects from historical database data',
    offlineObstacleVizTitle: 'Offline Obstacle Visualization',
    offlineObstacleVizDescription: 'Visualize obstacle information from offline data',
};
```

### 3. 默认布局配置

#### `modules/dreamview_plus/frontend/packages/dreamview-core/src/config/defaultLayouts.ts`

**需要确保 PERCEPTION 模式有布局：**

```typescript
export const DEFAULT_LAYOUTS: Record<CURRENT_MODE, MosaicNode<string> | null> = {
    [CURRENT_MODE.NONE]: null,
    [CURRENT_MODE.DEFAULT]: null,
    [CURRENT_MODE.PERCEPTION]: perceptionLayout, // 添加这一行
    [CURRENT_MODE.PNC]: null,
    [CURRENT_MODE.VEHICLE_TEST]: null,
    [CURRENT_MODE.MAP_COLLECT]: null,
    [CURRENT_MODE.MAP_EDITOR]: null,
    [CURRENT_MODE.CAMERA_CALIBRATION]: null,
    [CURRENT_MODE.LiDAR_CALIBRATION]: null,
    [CURRENT_MODE.DYNAMICS_CALIBRATION]: null,
    [CURRENT_MODE.CANBUS_DEBUG]: null,
    [CURRENT_MODE.OFFLINE_ANALYSIS]: offlineAnalysisLayout,
};
```

## 功能特性

### 1. OfflineAnalysis 面板功能
- **数据库连接**：支持 PostgreSQL、MySQL、MongoDB
- **数据查询**：时间范围、障碍物类型、置信度过滤
- **时间轴播放**：支持播放/暂停、速度控制
- **障碍物查看**：3D 可视化预览
- **数据分析**：统计信息展示

### 2. OfflineObstacleViz 面板功能
- **2D Canvas 渲染**：障碍物位置可视化
- **交互选择**：点击选择障碍物
- **轨迹显示**：历史轨迹可视化
- **预测显示**：未来路径预测
- **类型过滤**：按障碍物类型筛选
- **视图切换**：俯视图/透视图

### 3. 布局结构
```
┌─────────────────────────────────────────────────────────────┐
│                 Offline Analysis Mode Layout                │
├─────────────────────────────────┬───────────────────────────┤
│                                 │                           │
│    OfflineObstacleViz           │    OfflineAnalysis        │
│   (离线障碍物 3D 可视化)          │   (数据查询和控制面板)      │
│   - 3D 障碍物渲染                │   - 数据库连接             │
│   - 轨迹显示                    │   - 查询参数设置           │
│   - 交互选择                    │   - 时间轴播放器           │
│   - 视图控制                    │   - 数据分析工具           │
│                                 │                           │
│                                 ├───────────────────────────┤
│                                 │   Charts    │ ObstaclePanel│
│                                 │ (数据统计)   │ (障碍物列表) │
│                                 │ - 数量统计   │ - 详细信息   │
│                                 │ - 类型分布   │ - 属性列表   │
│                                 │ - 时间序列   │ - 过滤搜索   │
│                                 │ - 置信度分析 │ - 选择联动   │
├─────────────────────────────────┼───────────────────────────┤
│         Console                 │                           │
│      (系统控制台)                │                           │
│   - 系统日志                    │                           │
│   - 错误提示                    │                           │
│   - 状态反馈                    │                           │
└─────────────────────────────────┴───────────────────────────┘
```

## 使用流程

### 1. 启动应用
1. 打开 Dreamview Plus 应用
2. 在 WelcomeGuide 中选择 "Offline Analysis Mode"
3. 点击 "Enter This Mode" 按钮

### 2. 数据准备
1. 在 OfflineAnalysis 面板中连接数据库
2. 设置查询参数（时间范围、障碍物类型等）
3. 执行数据查询

### 3. 数据分析
1. 在 OfflineObstacleViz 面板中查看 3D 可视化
2. 使用时间轴播放器回放数据
3. 在 Charts 面板中查看统计分析
4. 在 ObstaclePanel 中查看详细列表

### 4. 交互操作
1. 点击 3D 视图中的障碍物进行选择
2. 查看选中障碍物的详细信息
3. 调整播放速度和时间轴位置
4. 切换不同的视图模式

## 技术架构

### 1. 状态管理
- 使用 `OfflineAnalysisStore` 管理所有 offline 相关状态
- 基于 `useReducer` 和 `Context` 实现
- 支持数据库连接、数据查询、时间轴播放等状态

### 2. 组件结构
- 基于 `Panel` 基类实现
- 使用 `react-mosaic-component` 进行布局管理
- 支持拖拽和调整面板大小

### 3. 数据流
```
数据库 → OfflineAnalysis → Charts + ObstaclePanel → OfflineObstacleViz → Console
  ↓           ↓                    ↓                      ↓              ↓
连接管理    查询控制              统计分析              3D可视化        状态反馈
```

## 总结

Offline Mode 的实现已经基本完成，包含了：
- ✅ 完整的类型定义和枚举
- ✅ 面板组件和样式
- ✅ 状态管理 Store
- ✅ 欢迎引导集成
- ✅ 默认布局配置
- ✅ 应用级集成

只需要进行以下小的修改：
- 在 PageLayout 中添加 OFFLINE_ANALYSIS 模式处理
- 添加国际化翻译
- 确保 PERCEPTION 模式有默认布局

修改完成后，Offline Mode 就可以完全正常使用了！
