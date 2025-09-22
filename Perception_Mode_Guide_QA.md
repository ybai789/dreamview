# Perception Mode 引导机制问答总结

## 问题：Perception mode 是如何在 WelcomeGuide 中跳转到其对应的 panel layout 的？

### 答案：完整的跳转流程

#### **1. 用户交互流程**

```
用户点击 "Enter This Mode" 按钮
    ↓
clickEnterThisMode() 函数执行
    ↓
dispatch(changeMode(mainApi, CURRENT_MODE.PERCEPTION))
    ↓
订阅 HMI_STATUS 等待模式切换确认
    ↓
clickEnterThisModeToGuide(CURRENT_MODE.PERCEPTION)
    ↓
显示对应的 Panel Layout
```

#### **2. 核心代码流程**

**A. WelcomeGuide 中的模式切换**
```typescript
// 文件：src/components/WelcomeGuide/index.tsx
const clickEnterThisMode = () => {
    // 1. 发送模式切换请求到后端
    dispatch(changeMode(mainApi, activeWelcomeGuideItemKey as CURRENT_MODE));
    
    // 2. 保存本地状态
    setLocalFirstUseGuideMode(activeWelcomeGuideItemKey);
    
    // 3. 订阅 HMI_STATUS 等待确认
    const subscription = streamApi?.subscribeToData(StreamDataNames.HMI_STATUS).subscribe((data: any) => {
        if (data?.currentMode === activeWelcomeGuideItemKey) {
            // 4. 模式切换确认后执行跳转
            clickEnterThisModeToGuide(activeWelcomeGuideItemKey);
            subscription.unsubscribe();
        }
    });
};
```

**B. PageLayout 中的跳转处理**
```typescript
// 文件：src/components/PageLayout/index.tsx
const clickEnterThisModeToGuide = useCallback(
    (useGuideMode: CURRENT_MODE) => {
        if (useGuideMode === CURRENT_MODE.PERCEPTION) {
            setEnterGuideState({ currentMode: CURRENT_MODE.PERCEPTION });
        }
        // ... 其他模式处理
    },
    [isMainConnected],
);
```

**C. 条件渲染逻辑**
```typescript
// 文件：src/components/PageLayout/index.tsx
return (
    <div className={c['dv-root']}>
        {/* 显示 WelcomeGuide */}
        {!localFirstUseGuideMode && (
            <WelcomeGuide
                clickEnterThisModeToGuide={clickEnterThisModeToGuide}
                setLocalFirstUseGuideMode={setLocalFirstUseGuideMode}
            />
        )}
        
        {/* 显示 Perception 引导 */}
        {enterGuideState.currentMode === CURRENT_MODE.PERCEPTION && (
            <PerceptionGuide controlPerceptionGuideLastStep={controlPerceptionGuideLastStep} />
        )}
        
        {/* 显示主界面和 PanelLayout */}
        {localFirstUseGuideMode && (
            <div className={c['dv-layout-content']}>
                <div className={c['dv-layout-window']}>
                    <div className={c['dv-layout-panellayout']}>
                        <PanelLayout /> {/* 这里渲染面板布局 */}
                    </div>
                </div>
            </div>
        )}
    </div>
);
```

#### **3. PanelLayout 的布局确定机制**

**A. 布局获取逻辑**
```typescript
// 文件：src/store/PanelLayoutStore/index.tsx
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
    
    return currentLayout;
}
```

**B. PanelLayout 渲染逻辑**
```typescript
// 文件：src/components/PanelLayout/index.tsx
function PanelLayout() {
    const layout = useGetCurrentLayout();
    
    return (
        <div className={classes['layout-root']}>
            {layout ? <DreamviewWindowMemo /> : <PanelEmpty />}
        </div>
    );
}
```

#### **4. 没有对应 Layout 时的处理机制**

**A. 显示 PanelEmpty 组件**
当 `useGetCurrentLayout()` 返回 `null` 时，会显示 `PanelEmpty` 组件：

```typescript
// 文件：src/components/PanelLayout/PanelEmpty/index.tsx
function PanelEmpty() {
    const { allPanel } = usePanelCatalogContext();
    
    const onAddPanel = (type: string) => {
        // 添加面板到布局
        dispatch(update({ mode: hmi.currentMode, layout: genereatePanelId(type) }));
    };
    
    return (
        <div className={classes['panel-empty-container']}>
            <div className={classes['panel-empty-title']}>
                {t('panelEmptyTitle')} {/* "请选择一个面板添加到布局中" */}
            </div>
            <CustomScroll className={classes['panel-list-container']}>
                <ul className={classes['panel-list']}>
                    {allPanel.map((item) => (
                        <li key={item.type}>
                            <div className={classes['panel-item-img']} />
                            <p>{item.title}</p>
                            <p>{item.description}</p>
                            <div onClick={() => onAddPanel(item.type)}>
                                <AddIcon />
                                <span>{t('addPanel')}</span>
                            </div>
                        </li>
                    ))}
                </ul>
            </CustomScroll>
        </div>
    );
}
```

**B. 面板添加机制**
用户可以从 `PanelEmpty` 中选择面板添加到布局中：

1. **显示所有可用面板**：从 `PanelCatalogStore` 获取所有注册的面板
2. **点击添加**：调用 `onAddPanel(type)` 函数
3. **更新布局**：通过 `dispatch(update())` 更新当前模式的布局
4. **重新渲染**：`PanelLayout` 重新渲染，显示新添加的面板

---

## 问题：如果没有对应的 panel layout 是如何跳转到其对应的 panel 的？

### 答案：渐进式面板添加机制

#### **1. 布局优先级**

1. **用户自定义布局**：`store.layout[mode].layout`
2. **默认布局**：`getDefaultLayout(mode)`
3. **上一次布局**：`prevLayout.current`
4. **空状态**：显示 `PanelEmpty`

#### **2. 面板添加流程**

```
没有布局 → 显示 PanelEmpty → 用户选择面板 → 添加到布局 → 重新渲染
```

#### **3. 关键特点**

**A. 异步确认机制**
- 不等待 HMI Action Response
- 通过订阅 HMI_STATUS 确认模式切换
- 确保模式切换成功后才进行 UI 更新

**B. 布局优先级**
1. **用户自定义布局**：`store.layout[mode].layout`
2. **默认布局**：`getDefaultLayout(mode)`
3. **上一次布局**：`prevLayout.current`
4. **空状态**：显示 `PanelEmpty`

**C. 渐进式体验**
- 先显示引导界面（可选）
- 再显示面板布局
- 支持用户自定义面板组合

---

## 问题：显示 PerceptionGuide (可选) 如何可选？

### 答案：PerceptionGuide 的"可选"显示机制

#### **1. 控制机制**

PerceptionGuide 的显示是通过 `enterGuideState.currentMode` 状态控制的：

```typescript
// 文件：src/components/PageLayout/index.tsx
const [enterGuideState, setEnterGuideState] = useState({ currentMode: '' });

// 条件渲染
{enterGuideState.currentMode === CURRENT_MODE.PERCEPTION && (
    <PerceptionGuide controlPerceptionGuideLastStep={controlPerceptionGuideLastStep} />
)}
```

#### **2. 触发条件**

PerceptionGuide 的显示需要满足以下条件：

**A. 模式切换触发**
```typescript
const clickEnterThisModeToGuide = useCallback(
    (useGuideMode: CURRENT_MODE) => {
        if (useGuideMode === CURRENT_MODE.PERCEPTION) {
            setEnterGuideState({ currentMode: CURRENT_MODE.PERCEPTION });
        }
        // ... 其他模式
    },
    [isMainConnected],
);
```

**B. 状态更新流程**
```
用户点击 "Enter This Mode" (Perception)
    ↓
clickEnterThisMode() 执行
    ↓
dispatch(changeMode(mainApi, PERCEPTION))
    ↓
订阅 HMI_STATUS 等待确认
    ↓
clickEnterThisModeToGuide(PERCEPTION)
    ↓
setEnterGuideState({ currentMode: PERCEPTION })
    ↓
PerceptionGuide 显示
```

#### **3. PerceptionGuide 的本质**

PerceptionGuide 实际上是一个**用户引导组件**，使用 `react-joyride` 库实现：

```typescript
// 文件：src/components/Guide/PerceptionGuide/index.tsx
return (
    <Joyride
        run={run}                    // 控制是否运行引导
        steps={steps(isLogin, stepIndexState, t)}  // 引导步骤
        continuous                   // 连续模式
        showSkipButton              // 显示跳过按钮
        callback={operationGuide}   // 操作回调
        styles={styles}             // 样式
        disableOverlayClose         // 禁用点击遮罩关闭
        scrollOffset={100}          // 滚动偏移
        disableScrolling={disableScrollState}  // 禁用滚动
    />
);
```

#### **4. "可选"的具体含义**

**A. 自动显示**
- 当用户选择 Perception mode 时，PerceptionGuide **自动显示**
- 不需要用户额外操作

**B. 可跳过**
- 用户可以通过 "Skip" 按钮跳过引导
- 用户可以通过 "Next" 按钮逐步完成引导

**C. 可关闭**
- 引导完成后自动关闭
- 用户可以通过操作提前结束引导

#### **5. 引导步骤**

PerceptionGuide 包含多个引导步骤：

```typescript
// 根据用户登录状态显示不同步骤
const steps = (isLogin: boolean, stepIndexState: number, t: TFunction) => {
    // 已登录用户：5个步骤
    // 未登录用户：4个步骤
    return filteredSteps.map((step, index) => ({
        target: step.target,
        content: step.content,
        placement: step.placement,
        // ... 其他配置
    }));
};
```

#### **6. 状态管理**

**A. 引导状态**
```typescript
const [run, setRun] = useState(false);                    // 是否运行引导
const [stepIndexState, setStepIndexState] = useState(0);  // 当前步骤索引
const [disableScrollState, setDisableScrollState] = useState(true); // 禁用滚动状态
```

**B. 样式控制**
```typescript
const [perceptionGuideLastStep, setPerceptionGuideLastState] = useState({});

// 在引导过程中动态调整样式
controlPerceptionGuideLastStep({
    position: 'absolute',
    left: 10,
    right: 20,
    top: 15,
    bottom: 20,
});
```

#### **7. 生命周期**

**A. 初始化**
```typescript
useEffect(() => {
    dispatch(UpdateMenuAction(ENUM_MENU_KEY.MODE_SETTING));
    setRun(true);  // 自动开始引导
}, []);
```

**B. 步骤切换**
```typescript
const operationGuide = (operationGuideProps: any) => {
    const { action, index, lifecycle, status } = operationGuideProps;
    
    // 处理步骤切换逻辑
    if (action === 'next' && index === 3 && lifecycle === 'complete') {
        controlPerceptionGuideLastStep({
            position: 'absolute',
            left: 10,
            right: 20,
            top: 15,
            bottom: 20,
        });
    }
    
    // 引导完成
    if (action === 'next' && index === 4 && lifecycle === 'complete') {
        controlPerceptionGuideLastStep({});
    }
};
```

#### **8. 为什么是"可选"的**

**A. 用户体验考虑**
- **新用户**：需要引导了解界面功能
- **老用户**：可能不需要重复引导
- **跳过机制**：允许用户快速进入工作状态

**B. 技术实现**
- **状态控制**：通过 `enterGuideState.currentMode` 控制显示
- **条件渲染**：只在特定模式下显示
- **生命周期管理**：自动开始和结束

**C. 灵活性**
- **可配置**：可以修改引导步骤内容
- **可扩展**：可以为其他模式添加类似引导
- **可控制**：可以通过状态管理控制显示时机

---

## 问题：如何修改代码跳过这个引导直接进入 perception panel？

### 答案：多种跳过引导的方法

#### **方法1：注释掉 PerceptionGuide 渲染（推荐）**

**最简单直接的方法**是注释掉 PerceptionGuide 的渲染代码：

```typescript
// 文件：src/components/PageLayout/index.tsx
{/* 跳过 PerceptionGuide 直接进入 panel */}
{/* {enterGuideState.currentMode === CURRENT_MODE.PERCEPTION && (
    <PerceptionGuide controlPerceptionGuideLastStep={controlPerceptionGuideLastStep} />
)} */}
```

#### **方法2：修改 PerceptionGuide 组件**

让 PerceptionGuide 立即完成：

```typescript
// 文件：src/components/Guide/PerceptionGuide/index.tsx
useEffect(() => {
    dispatch(UpdateMenuAction(ENUM_MENU_KEY.MODE_SETTING));
    // 跳过引导，直接完成
    setRun(false);
}, []);
```

#### **方法3：使用配置控制**

**A. 创建配置文件**
```typescript
// 文件：src/config/guideConfig.ts
export const GUIDE_CONFIG = {
    // 是否显示 Perception 引导
    showPerceptionGuide: false,
    
    // 是否显示其他引导
    showDefaultGuide: true,
    showPNCGuide: true,
    showVehicleGuide: true,
    
    // 是否显示所有引导
    showAllGuides: false,
};

// 检查是否应该显示特定引导
export function shouldShowGuide(mode: string): boolean {
    if (GUIDE_CONFIG.showAllGuides) {
        return true;
    }
    
    switch (mode) {
        case 'Perception':
            return GUIDE_CONFIG.showPerceptionGuide;
        case 'Default':
            return GUIDE_CONFIG.showDefaultGuide;
        case 'PNC':
            return GUIDE_CONFIG.showPNCGuide;
        case 'Vehicle Test':
            return GUIDE_CONFIG.showVehicleGuide;
        default:
            return false;
    }
}
```

**B. 修改 PageLayout 使用配置**
```typescript
// 文件：src/components/PageLayout/index.tsx
import { shouldShowGuide } from '../../config/guideConfig';

// 条件渲染
{enterGuideState.currentMode === CURRENT_MODE.PERCEPTION && shouldShowGuide('Perception') && (
    <PerceptionGuide controlPerceptionGuideLastStep={controlPerceptionGuideLastStep} />
)}
```

#### **方法4：修改 clickEnterThisModeToGuide 函数**

直接设置 localFirstUseGuideMode 为 true：

```typescript
// 文件：src/components/PageLayout/index.tsx
const clickEnterThisModeToGuide = useCallback(
    (useGuideMode: CURRENT_MODE) => {
        if (useGuideMode === CURRENT_MODE.PERCEPTION) {
            // 跳过 PerceptionGuide，直接进入 panel
            setEnterGuideState({ currentMode: '' });
            setLocalFirstUseGuideMode(CURRENT_MODE.PERCEPTION);
        }
        // ... 其他模式处理
    },
    [isMainConnected],
);
```

### **修改效果：**

1. **用户选择 Perception mode** → 直接进入 PanelLayout
2. **不再显示引导步骤** → 立即显示面板界面
3. **保持其他功能正常** → 其他模式的引导不受影响

### **验证方法：**

1. 启动应用
2. 选择 "Perception Mode"
3. 点击 "Enter This Mode"
4. 应该直接显示面板布局，不再有引导步骤

---

## 总结

### **完整的 Perception Mode 跳转流程**

```
┌─────────────────────────────────────────────────────────────┐
│                    Perception Mode 跳转流程                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 用户点击 "Enter This Mode"                              │
│     ↓                                                        │
│  2. clickEnterThisMode() 执行                               │
│     - dispatch(changeMode(mainApi, PERCEPTION))             │
│     - setLocalFirstUseGuideMode(PERCEPTION)                 │
│     - 订阅 HMI_STATUS 等待确认                              │
│     ↓                                                        │
│  3. 后端处理模式切换                                         │
│     - 更新 HMI 状态                                          │
│     - 广播 HMI_STATUS 更新                                  │
│     ↓                                                        │
│  4. 前端接收 HMI_STATUS 更新                                │
│     - 检查 data.currentMode === PERCEPTION                  │
│     - 调用 clickEnterThisModeToGuide(PERCEPTION)            │
│     ↓                                                        │
│  5. PageLayout 状态更新                                     │
│     - setEnterGuideState({ currentMode: PERCEPTION })       │
│     - 隐藏 WelcomeGuide                                     │
│     - 显示 PerceptionGuide (可选)                           │
│     - 显示 PanelLayout                                      │
│     ↓                                                        │
│  6. PanelLayout 确定布局                                    │
│     - useGetCurrentLayout() 获取布局                        │
│     - 检查 store.layout[PERCEPTION]?.layout                │
│     - 如果没有，尝试 getDefaultLayout(PERCEPTION)           │
│     - 如果还没有，返回 prevLayout.current                   │
│     ↓                                                        │
│  7. 渲染面板                                                │
│     - 如果有布局：渲染 Mosaic 组件                          │
│     - 如果没有布局：渲染 PanelEmpty 组件                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### **关键特点**

1. **异步确认机制**：通过订阅 HMI_STATUS 确认模式切换
2. **布局优先级**：用户自定义 → 默认布局 → 上一次布局 → 空状态
3. **渐进式体验**：引导界面 → 面板布局 → 用户自定义
4. **灵活控制**：可以通过多种方式跳过引导

这种设计确保了 Perception mode 能够平滑地从 WelcomeGuide 跳转到对应的面板布局，同时提供了灵活的布局管理和用户引导功能。
