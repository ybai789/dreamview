我来帮您解析dreamview-log目录下的文件结构和内容。

Ran tool
Read file: frontend/packages/dreamview-log/package.json
Read file: frontend/packages/dreamview-log/README.md
Ran tool
Ran tool
Read file: frontend/packages/dreamview-log/lib/log.ts
Read file: frontend/packages/dreamview-log/lib/log.ts
Read file: frontend/packages/dreamview-log/lib/log.ts
Read file: frontend/packages/dreamview-log/lib/utils/constant.ts
Read file: frontend/packages/dreamview-log/lib/utils/visualizeFlatted.ts
Ran tool
Read file: frontend/packages/dreamview-log/__tests__/log.test.js
Read file: frontend/packages/dreamview-log/jest.config.js
Read file: frontend/packages/dreamview-log/tsconfig.json
## dreamview-log 模块解析

基于对 `dreamview-log` 目录下文件的详细分析，这是一个用于创建和管理日志的 TypeScript 库模块。以下是详细解析：

### �� 目录结构
```
dreamview-log/
├── package.json          # 包配置文件
├── README.md            # 模块说明文档
├── tsconfig.json        # TypeScript 配置
├── jest.config.js       # Jest 测试配置
├── .eslintrc.js         # ESLint 配置
├── lib/                 # 核心代码目录
│   ├── log.ts          # 主要的 Logger 类实现
│   └── utils/          # 工具函数目录
│       ├── constant.ts # 全局常量和工具函数
│       └── visualizeFlatted.ts # 数据可视化工具
└── __tests__/          # 测试文件目录
    └── log.test.js     # Logger 类的单元测试
```

### 🎯 核心功能

#### 1. **Logger 类** (`lib/log.ts`)
- **单例模式**: 每个名称对应一个唯一的 Logger 实例
- **日志级别**: 支持 DEBUG、INFO、WARN、ERROR 四个级别
- **双重输出**: 既输出到控制台，也支持输出到 HTML 元素
- **性能优化**: 使用缓冲区批量处理日志，避免频繁 DOM 操作

#### 2. **主要特性**
- **时间戳**: 自动添加 ISO 格式的时间戳
- **实例名称**: 支持为每个 Logger 实例命名
- **HTML 输出**: 可以将日志输出到指定的 HTML 元素
- **颜色区分**: 不同日志级别使用不同颜色显示
- **缓冲区管理**: 自动限制日志数量（最多500条）

#### 3. **全局工具函数**
- `setLogLevel(name?, level)`: 设置指定或所有 Logger 的日志级别
- `showLogNames()`: 显示所有已注册的 Logger 实例名称

### 🔧 技术实现

#### 依赖库
- **loglevel**: 基础日志库
- **rxjs**: 响应式编程，用于消息处理
- **flatted**: 处理循环引用的数据序列化
- **lodash**: 工具函数库

#### 核心类设计
```typescript
export default class Logger {
    private static instances: Map<string, Logger>;  // 单例存储
    private logger: loglevel.Logger;               // 底层日志实例
    private logElement: HTMLElement | null;        // HTML 输出元素
    private logBuffer: HTMLElement[];              // 日志缓冲区
    
    // 主要方法
    static getInstance(name: string): Logger;      // 获取实例
    debug/info/warn/error(...message): void;       // 日志方法
    setLevel(level): void;                        // 设置日志级别
    setLogElement(element): void;                 // 设置 HTML 输出
}
```

### 🛠️ 工具函数

#### `constant.ts`
- **Global**: 跨环境全局对象获取（浏览器/Worker/Node.js）
- **isDarkMode**: 检测系统是否为深色模式

#### `visualizeFlatted.ts`
- **visualizeFlatted**: 处理包含循环引用的对象序列化
- 使用 flatted 库解决 JSON.stringify 无法处理循环引用的问题

### 📝 使用示例

```typescript
// 创建 Logger 实例
const logger = Logger.getInstance('myLogger');
const logger = Logger.getInstance(__filename);

// 输出不同级别的日志
logger.debug('This is a debug message');
logger.info('This is an info message');
logger.warn('This is a warning message');
logger.error('This is an error message');

// 设置 HTML 输出
logger.setLogElement(document.getElementById('log-container'));

// 全局设置日志级别
setLogLevel('myLogger', 'info');
setLogLevel(); // 设置所有 Logger 为 debug 级别

// 查看所有 Logger 实例
showLogNames();
```

### 🧪 测试覆盖
- 单例模式测试
- 各日志级别方法测试
- 实例创建和复用测试

### �� 特色功能
1. **智能缓冲区**: 使用 `requestAnimationFrame` 批量处理 DOM 操作
2. **深色模式适配**: 根据系统主题自动调整显示样式
3. **循环引用处理**: 安全处理包含循环引用的复杂对象
4. **跨环境兼容**: 支持浏览器、Web Worker、Node.js 环境

这个模块为 DreamView 项目提供了一个功能完整、性能优化的日志系统，特别适合在 Web 应用中展示和管理日志信息。

------

我来帮您解析dreamview-web目录下的文件结构和内容。

Ran tool
Read file: frontend/packages/dreamview-web/package.json
Read file: frontend/packages/dreamview-web/README.md
Ran tool
Read file: frontend/packages/dreamview-web/src/index.tsx
Read file: frontend/packages/dreamview-web/src/Root.tsx
Ran tool
Read file: frontend/packages/dreamview-web/public/index.html
Read file: frontend/packages/dreamview-web/public/manifest.json
Ran tool
Read file: frontend/packages/dreamview-web/config/path.js
Read file: frontend/packages/dreamview-web/config/webpackConfig.js
Read file: frontend/packages/dreamview-web/config/webpackConfig.js
Read file: frontend/packages/dreamview-web/tsconfig.json
Read file: frontend/packages/dreamview-web/.eslintrc.js
## dreamview-web 模块解析

基于对 `dreamview-web` 目录下文件的详细分析，这是一个基于 React 的 Web 应用入口模块，主要负责 DreamView 的 Web 平台初始化和配置。以下是详细解析：

### �� 目录结构
```
dreamview-web/
├── package.json          # 包配置文件
├── README.md            # 模块说明文档
├── tsconfig.json        # TypeScript 配置
├── .eslintrc.js         # ESLint 配置
├── src/                 # 源代码目录
│   ├── index.tsx       # 应用入口文件
│   └── Root.tsx        # 根组件
├── public/             # 静态资源目录
│   ├── index.html      # HTML 模板
│   └── manifest.json   # PWA 清单文件
└── config/             # 构建配置目录
    ├── path.js         # 路径配置
    └── webpackConfig.js # Webpack 配置
```

### �� 核心功能

#### 1. **应用入口** (`src/index.tsx`)
- **应用初始化**: 提供 `dreamviewWebInit` 异步函数作为应用入口
- **DOM 挂载**: 使用 React 18 的 `createRoot` API 挂载应用
- **国际化**: 集成 `@dreamview/dreamview-lang` 进行国际化支持
- **性能监控**: 集成 `@dreamview/dreamview-analysis` 的性能监控插件
- **浏览器兼容性**: 原本有 Chrome 浏览器限制，目前被注释掉

#### 2. **根组件** (`src/Root.tsx`)
- **核心应用**: 渲染 `@dreamview/dreamview-core` 的 `App` 组件
- **Service Worker**: 原本支持 PWA 功能，目前被注释掉
- **日志记录**: 使用 `@dreamview/log` 记录应用状态

### 🔧 技术栈

#### 主要依赖
- **React 18**: 使用最新的 React 版本和并发特性
- **TypeScript**: 完整的类型支持
- **Webpack 5**: 现代化的构建工具
- **Module Federation**: 微前端架构支持

#### 开发工具
- **ESLint**: 代码质量检查
- **Prettier**: 代码格式化
- **Jest**: 单元测试
- **React Refresh**: 热重载支持

### 🌐 Web 平台特性

#### 1. **PWA 支持**
```json
// manifest.json
{
    "short_name": "Dreamview",
    "name": "Dreamview",
    "description": "Apollo's HMI module",
    "display": "standalone",
    "theme_color": "#000000",
    "background_color": "#ffffff"
}
```

#### 2. **HTML 模板**
```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="zh-CN">
    <head>
        <title>dreamview core</title>
        <!-- 全局变量配置 -->
        <script>
            var CONSOLE_START_TIME = (new Date).getTime();
            var G_APP_NAME = 'home';
            var G_API_PREFIX = '/api/apollo';
        </script>
    </head>
    <body>
        <div id="root"></div>
    </body>
</html>
```

### ⚙️ 构建配置

#### Webpack 配置特性
1. **模块联邦**: 支持微前端架构
2. **热重载**: 开发环境支持 React 组件热重载
3. **代码分割**: 生产环境自动代码分割和缓存
4. **代理配置**: 开发环境 API 代理到本地后端
5. **资源复制**: 自动复制静态资源

#### 共享模块配置
```javascript
// Module Federation 共享模块
shared: {
    react: { singleton: true, eager: true },
    'react-dom': { singleton: true, eager: true },
    '@dreamview/dreamview-lang': { singleton: true, eager: true },
    '@dreamview/dreamview-theme': { singleton: true, eager: true }
}
```

### 🚀 应用启动流程

1. **DOM 检查**: 验证 `#root` 元素存在
2. **浏览器检查**: 原本检查 Chrome 浏览器（已注释）
3. **日志初始化**: 记录应用启动日志
4. **组件加载**: 动态导入 `Root` 组件
5. **国际化初始化**: 调用 `initI18n()`
6. **应用渲染**: 渲染根组件和性能监控插件

### 🔌 插件集成

#### 性能监控
```typescript
import { PerformanceMonitorPluginAdaptor } from '@dreamview/dreamview-analysis';

// 在应用根组件中集成性能监控
<>
    <Root />
    <PerformanceMonitorPluginAdaptor />
</>
```

#### 日志系统
```typescript
import Logger from '@dreamview/log';

const logger = Logger.getInstance(__filename);
logger.info('dreamview-web init');
```

### 🛠️ 开发特性

#### 开发服务器配置
- **端口检测**: 自动检测可用端口
- **热重载**: 支持组件热重载
- **代理设置**: API 请求代理到后端服务
- **自动打开**: 启动时自动打开浏览器

#### 生产构建优化
- **代码分割**: 按需加载优化
- **缓存优化**: 文件名包含内容哈希
- **资源压缩**: CSS 提取和压缩
- **Service Worker**: PWA 支持（目前注释）

### 📊 模块关系

```
dreamview-web
├── @dreamview/dreamview-core    # 核心应用组件
├── @dreamview/log              # 日志系统
├── @dreamview/dreamview-lang   # 国际化
├── @dreamview/dreamview-theme  # 主题系统
└── @dreamview/dreamview-analysis # 性能监控
```

### 🎨 特色功能

1. **微前端架构**: 通过 Module Federation 支持模块化开发
2. **国际化支持**: 内置多语言支持
3. **性能监控**: 集成性能分析工具
4. **PWA 支持**: 支持渐进式 Web 应用
5. **开发体验**: 完整的开发工具链和热重载

这个模块作为 DreamView 的 Web 平台入口，提供了现代化的 React 应用架构，支持微前端、国际化、性能监控等企业级特性，为 Apollo 自动驾驶系统的可视化界面提供了强大的 Web 平台支持。

我来帮您解析dreamview-ui目录下的文件结构和内容。

Ran tool
Read file: frontend/packages/dreamview-ui/package.json
Ran tool
Read file: frontend/packages/dreamview-ui/src/index.ts
Ran tool
Read file: frontend/packages/dreamview-ui/src/components/index.ts
Ran tool
Read file: frontend/packages/dreamview-ui/src/components/Button/index.tsx
Ran tool
Read file: frontend/packages/dreamview-ui/src/IconPark/index.tsx
Ran tool
Read file: frontend/packages/dreamview-ui/src/IconPark/Light/index.ts
Ran tool
Ran tool
Read file: frontend/packages/dreamview-ui/src/tools/prefixCls/prefixCls.ts
Ran tool
Read file: frontend/packages/dreamview-ui/template/index.tsx
Ran tool
Read file: frontend/packages/dreamview-ui/scripts/gen_component.sh
Read file: frontend/packages/dreamview-ui/scripts/gen_icon_with_svg.js
Ran tool
Read file: frontend/packages/dreamview-ui/.storybook/main.js
Ran tool
Read file: frontend/packages/dreamview-ui/config/webpack.config.js
Ran tool
## dreamview-ui 模块解析

基于对 `dreamview-ui` 目录下文件的详细分析，这是一个基于 React 和 Ant Design 的 UI 组件库，专门为 DreamView 项目定制。以下是详细解析：

### 🏗️ 目录结构
```
dreamview-ui/
├── package.json              # 包配置文件
├── tsconfig.json            # TypeScript 配置
├── .babelrc.json           # Babel 配置
├── .eslintrc.js            # ESLint 配置
├── jest.config.js          # Jest 测试配置
├── src/                    # 源代码目录
│   ├── index.ts           # 主入口文件
│   ├── components/        # UI 组件目录
│   ├── IconPark/         # 图标系统
│   ├── tools/            # 工具函数
│   ├── svgs/             # SVG 图标资源
│   ├── raw_svgs/         # 原始 SVG 文件
│   └── mdxs/             # Storybook 文档
├── template/              # 组件模板
├── scripts/               # 构建脚本
├── config/                # 构建配置
├── .storybook/           # Storybook 配置
└── dist/                 # 构建输出目录
```

### �� 核心功能

#### 1. **UI 组件库** (`src/components/`)
提供丰富的 React 组件，包括：
- **基础组件**: Button、Input、Modal、Tabs
- **表单组件**: Form、Select、Checkbox、Radio、Switch
- **展示组件**: List、Card、Progress、Spin、Tag
- **交互组件**: Popover、Collapse、Tree、Steps
- **特殊组件**: ColorPicker、ImagePrak、Wave 效果

#### 2. **图标系统** (`src/IconPark/`)
- **主题适配**: 支持 Light 和 Dark 两种主题
- **SVG 图标**: 大量自定义 SVG 图标
- **动态切换**: 根据主题自动切换图标样式
- **图标生成**: 自动化图标生成脚本

### 🔧 技术栈

#### 主要依赖
- **React 18**: 最新版本的 React
- **Ant Design 5.5.1**: 企业级 UI 组件库
- **TypeScript**: 完整的类型支持
- **Less**: CSS 预处理器
- **Storybook**: 组件文档和开发环境

#### 开发工具
- **Webpack 5**: 现代化构建工具
- **Babel**: JavaScript 编译器
- **ESLint**: 代码质量检查
- **Jest**: 单元测试框架

### 🎨 组件特性

#### 1. **Button 组件示例**
```typescript
// 支持多种类型和状态
export interface BaseButtonProps {
    type?: 'default' | 'primary' | 'dashed' | 'link' | 'text';
    shape?: 'default' | 'circle' | 'round';
    size?: 'small' | 'middle' | 'large';
    loading?: boolean | { delay?: number };
    danger?: boolean;
    ghost?: boolean;
    block?: boolean;
}
```

#### 2. **主题系统集成**
```typescript
// IconPark 组件支持主题切换
export default function IconPark(props: IconComponentProps) {
    const theme = useThemeContext()?.theme;
    const Comp = theme === 'drak' ? DrakSvg : LightSvg;
    return <Icon component={Comp[name]} {...others}/>;
}
```

### 🛠️ 构建系统

#### Webpack 配置特性
1. **UMD 输出**: 支持多种模块系统
2. **CSS 提取**: 自动提取和压缩样式
3. **SVG 处理**: 支持 SVG 作为 React 组件
4. **TypeScript**: 完整的 TS 支持
5. **Less 支持**: 预处理器支持

#### 构建脚本
```bash
# 生成新组件
yarn gen:component <component_name>

# 生成图标
yarn gen:icons:svg    # 生成 SVG 图标
yarn gen:icons:mdx    # 生成 MDX 文档
yarn gen:icons        # 生成所有图标

# 构建文档
yarn build:doc        # 构建 Storybook
yarn serve           # 启动 Storybook 开发服务器
```

### 📚 文档系统

#### Storybook 集成
- **组件文档**: 自动生成组件文档
- **交互测试**: 支持组件交互测试
- **主题预览**: 支持不同主题预览
- **代码示例**: 提供完整的使用示例

#### 配置特性
```javascript
// Storybook 配置支持
const config = {
    stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
    addons: ['@storybook/addon-links', '@storybook/addon-essentials'],
    framework: { name: '@storybook/react-webpack5' },
    webpackFinal: async (config) => {
        // 自定义 Webpack 配置
        config.module.rules.push({
            test: /\.less$/,
            use: ['style-loader', 'css-loader', 'less-loader']
        });
        return config;
    }
};
```

### �� 工具函数

#### 1. **前缀管理** (`src/tools/prefixCls/`)
```typescript
const GLOBAL_PREFIX_CLS = 'dreamview';

export const getPrefixCls = (suffixCls?: string, customizePrefixCls?: string): string => {
    if (customizePrefixCls) return customizePrefixCls;
    return suffixCls ? `${GLOBAL_PREFIX_CLS}-${suffixCls}` : GLOBAL_PREFIX_CLS;
};
```

#### 2. **图标生成工具** (`scripts/gen_icon_with_svg.js`)
- **SVG 优化**: 使用 SVGO 优化 SVG 文件
- **自动导入**: 自动生成图标导入语句
- **主题支持**: 支持 Light/Dark 主题图标
- **批量处理**: 批量处理多个 SVG 文件

### �� 设计系统

#### 1. **图标库**
- **数量丰富**: 包含 100+ 个自定义图标
- **主题适配**: 每个图标都有 Light/Dark 版本
- **语义化命名**: 图标名称具有明确的语义
- **矢量格式**: 支持任意缩放不失真

#### 2. **组件设计原则**
- **一致性**: 统一的视觉风格和交互模式
- **可访问性**: 支持键盘导航和屏幕阅读器
- **响应式**: 适配不同屏幕尺寸
- **主题化**: 支持主题切换

### 📦 模块导出

#### 主入口文件
```typescript
// src/index.ts
export { MyComponent } from './components/MyComponent';
export * from './components';
export { default as IconPark } from './IconPark';
```

#### 组件导出
```typescript
// src/components/index.ts
export { Input, TextArea } from './Input';
export { Button } from './Button';
export { Modal } from './Modal';
export { Select } from './Select';
// ... 更多组件
```

### 🔌 集成特性

#### 1. **Ant Design 集成**
- **基础组件**: 基于 Ant Design 组件扩展
- **主题系统**: 与 Ant Design 主题系统兼容
- **样式定制**: 支持 Less 变量定制

#### 2. **DreamView 生态集成**
- **主题系统**: 与 `@dreamview/dreamview-theme` 集成
- **日志系统**: 使用 `@dreamview/log` 记录组件状态
- **国际化**: 支持多语言显示

### �� 开发体验

#### 1. **快速开发**
- **组件模板**: 提供标准组件模板
- **自动生成**: 脚本自动生成组件结构
- **热重载**: 开发环境支持热重载

#### 2. **质量保证**
- **TypeScript**: 完整的类型检查
- **ESLint**: 代码质量检查
- **Jest**: 单元测试覆盖
- **Storybook**: 组件文档和测试

### 🎯 使用场景

1. **DreamView 界面**: 为 Apollo 自动驾驶系统提供统一的 UI 组件
2. **可视化面板**: 支持各种数据可视化组件
3. **交互界面**: 提供丰富的用户交互组件
4. **主题定制**: 支持深色/浅色主题切换

这个 UI 组件库为 DreamView 项目提供了完整的设计系统，包括组件库、图标系统、主题支持和开发工具，确保整个应用具有一致的用户体验和高效的开发效率。

我来帮您解析dreamview-theme目录下的文件结构和内容。

Ran tool
Read file: frontend/packages/dreamview-theme/package.json
Read file: frontend/packages/dreamview-theme/README.md
Read file: frontend/packages/dreamview-theme/main.ts
Read file: frontend/packages/dreamview-theme/createStyles.tsx
Ran tool
Ran tool
Read file: frontend/packages/dreamview-theme/tokens/baseToken/index.ts
Ran tool
Read file: frontend/packages/dreamview-theme/tokens/util/index.ts
Ran tool
Read file: frontend/packages/dreamview-theme/light/base.ts
Read file: frontend/packages/dreamview-theme/light/components.ts
Ran tool
Read file: frontend/packages/dreamview-theme/drak/base.ts
Read file: frontend/packages/dreamview-theme/light/button.ts
Read file: frontend/packages/dreamview-theme/light/lightButton.ts
Read file: frontend/packages/dreamview-theme/light/modal.ts
Read file: frontend/packages/dreamview-theme/light/carViz.ts
## dreamview-theme 模块解析

基于对 `dreamview-theme` 目录下文件的详细分析，这是一个基于 React 和 Emotion 的主题系统，为 DreamView 项目提供统一的设计令牌和主题管理。以下是详细解析：

### 🏗️ 目录结构
```
dreamview-theme/
├── package.json              # 包配置文件
├── README.md                # 详细使用文档
├── main.ts                  # 主入口文件
├── createStyles.tsx         # 主题提供者和样式创建
├── theme.stories.tsx        # Storybook 主题故事
├── tokens/                  # 设计令牌目录
│   ├── baseToken/          # 基础令牌
│   │   └── index.ts       # 令牌生成和类型定义
│   └── util/              # 工具函数
│       └── index.ts       # 样式工具函数
├── light/                  # 浅色主题
│   ├── base.ts            # 基础颜色和字体
│   ├── components.ts      # 组件样式集合
│   └── [component].ts     # 各组件样式定义
└── drak/                  # 深色主题
    ├── base.ts            # 基础颜色和字体
    ├── components.ts      # 组件样式集合
    └── [component].ts     # 各组件样式定义
```

### �� 核心功能

#### 1. **主题系统架构**
- **双主题支持**: Light（浅色）和 Dark（深色）主题
- **设计令牌**: 统一的颜色、字体、间距等设计变量
- **组件样式**: 预定义的组件样式集合
- **动态切换**: 支持运行时主题切换

#### 2. **技术栈**
- **React 18**: 最新版本的 React
- **Emotion**: CSS-in-JS 解决方案
- **tss-react**: TypeScript 友好的样式系统
- **Ant Design Colors**: 颜色系统集成

### 🔧 主题提供者系统

#### 1. **Provider 组件**
```typescript
// 基础主题提供者
export function Provider(props: React.PropsWithChildren<ProviderProps>) {
    const { theme = 'light' } = props;
    const values = useMemo(() => ({
        theme,
        tokens: { light, drak }[theme],
    }), [theme]);
    return <context.Provider value={values}>{props.children}</context.Provider>;
}

// 扩展主题提供者
export function ExpandProvider<T>(props: ExpandProviderProps<T>) {
    // 支持自定义主题扩展
}
```

#### 2. **样式创建系统**
```typescript
// 使用 makeStyles 创建样式
export const makeStyles = initUseStyle();

// 使用示例
const useStyle = makeStyles((theme) => ({
    root: {
        backgroundColor: theme.tokens.backgroundColor.main,
        color: theme.tokens.font.color.main,
    },
}));
```

### �� 设计令牌系统

#### 1. **颜色系统**
```typescript
// 品牌色系
brand1: '#044CB9',    // 主品牌色
brand2: '#055FE7',    // 品牌色变体
brand3: '#347EED',    // 品牌色亮色
brand4: '#CFE5FC',    // 品牌色浅色
brand5: '#E6EFFC',    // 品牌色最浅色

// 语义化颜色
error1: '#CC2B36',    // 错误色
warn1: '#CC5A04',     // 警告色
success1: '#009072',  // 成功色
yellow1: '#C79E07',   // 黄色
```

#### 2. **字体系统**
```typescript
// 字体大小
size: {
    sm: '12px',
    regular: '14px',
    large: '16px',
    huge: '18px',
},

// 字体权重
weight: {
    light: 300,
    regular: 400,
    medium: 500,
    semibold: 700,
},

// 行高
lineHeight: {
    dense: 1.4,      // 密集
    regular: 1.5714, // 常规
    sparse: 1.8,     // 稀疏
}
```

#### 3. **间距系统**
```typescript
padding: {
    speace0: '0',
    speace: '8px',
    speace2: '16px',
    speace3: '24px',
},
```

### 🌓 主题对比

#### 浅色主题 (Light)
- **背景色**: 白色系 (`#FFFFFF`, `#F5F7FA`)
- **文字色**: 深色系 (`#232A33`, `#6E7277`)
- **分割线**: 浅灰色 (`#DBDDE0`, `#EEEEEE`)
- **品牌色**: 蓝色系 (`#044CB9`, `#055FE7`)

#### 深色主题 (Dark)
- **背景色**: 深色系 (`#1A1D24`, `#343C4D`)
- **文字色**: 浅色系 (`#FFFFFF`, `#A6B5CC`)
- **分割线**: 深灰色 (`#383C4D`, `#252833`)
- **品牌色**: 亮蓝色 (`#1252C0`, `#1971E6`)

### �� 组件样式系统

#### 1. **预定义组件样式**
```typescript
// 组件样式集合
export const components = {
    button: {},
    select,
    modal,
    input,
    lightButton,
    carViz,
    // ... 更多组件
};
```

#### 2. **组件样式示例**
```typescript
// Modal 组件样式
export default {
    contentColor: colors.fontColor5,
    headColor: colors.fontColor5,
    closeIconColor: colors.fontColor3,
    backgroundColor: colors.background2,
    divider: colors.divider2,
    closeBtnColor: colors.fontColor5,
    closeBtnHoverColor: colors.brand3,
};

// CarViz 组件样式
export default {
    bgColor: '#F5F7FA',
    textColor: '#232A33',
    gridColor: 'black',
    colorMapping: {
        YELLOW: '#daa520',
        RED: 'red',
        GREEN: '#006400',
        // ... 更多颜色映射
    },
};
```

### 🛠️ 工具函数

#### 1. **布局工具**
```typescript
// Flex 布局
flex(direction, justifyContent, alignItems) {
    return {
        display: 'flex',
        flexDirection: direction,
        justifyContent,
        alignItems,
    };
},

// 居中布局
flexCenterCenter: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
},
```

#### 2. **文本工具**
```typescript
// 文本省略
textEllipsis: {
    overflow: 'hidden',
    textOverflow: 'ellipsis',
    whiteSpace: 'nowrap',
},

// 多行文本省略
textEllipsis2: {
    width: '100%',
    overflow: 'hidden',
    textOverflow: 'ellipsis',
    display: '-webkit-box',
    '-WebkitLineClamp': '2',
    '-WebkitBoxOrient': 'vertical',
},
```

#### 3. **滚动工具**
```typescript
// 滚动条控制
scrollX: {
    'overflow-x': 'hidden',
    '&:hover': { 'overflow-x': 'auto' },
},
scrollY: {
    'overflow-y': 'hidden',
    '&:hover': { 'overflow-y': 'auto' },
},
```

### �� 使用示例

#### 1. **基础使用**
```typescript
import { Provider as ThemeProvider, makeStyles } from '@dreamview/dreamview-theme';

const useStyle = makeStyles((theme) => ({
    root: {
        backgroundColor: theme.tokens.backgroundColor.main,
        color: theme.tokens.font.color.main,
    },
}));

function App() {
    return (
        <ThemeProvider theme="light">
            <ChildComponents />
        </ThemeProvider>
    );
}
```

#### 2. **自定义主题**
```typescript
const customTheme = {
    tokens: {
        backgroundColor: {
            main: 'blue'
        }
    }
};

<ThemeProvider config={customTheme}>
    <ChildComponents />
</ThemeProvider>
```

#### 3. **组件样式使用**
```typescript
const useStyle = makeStyles((theme) => ({
    root: theme.components.button.primary,
}));
```

### 🔌 集成特性

#### 1. **React 集成**
- **Context API**: 使用 React Context 提供主题
- **Hooks**: 提供 `useThemeContext` Hook
- **TypeScript**: 完整的类型支持

#### 2. **Emotion 集成**
- **CSS-in-JS**: 支持动态样式生成
- **主题注入**: 自动注入主题变量
- **性能优化**: 样式缓存和优化

#### 3. **Ant Design 集成**
- **颜色系统**: 与 Ant Design 颜色系统兼容
- **组件样式**: 支持 Ant Design 组件样式定制

### 🎯 设计原则

#### 1. **一致性**
- 统一的设计令牌系统
- 一致的组件样式定义
- 标准化的颜色和字体规范

#### 2. **可维护性**
- 模块化的主题结构
- 类型安全的主题定义
- 清晰的命名规范

#### 3. **可扩展性**
- 支持自定义主题扩展
- 灵活的组件样式定制
- 动态主题切换能力

### �� 开发体验

#### 1. **类型安全**
- 完整的 TypeScript 支持
- 自动类型推导
- 编译时错误检查

#### 2. **开发工具**
- Storybook 集成
- 主题预览功能
- 样式调试工具

#### 3. **性能优化**
- 样式缓存机制
- 按需加载主题
- 最小化重渲染

这个主题系统为 DreamView 项目提供了完整的设计系统解决方案，确保整个应用具有一致的用户体验和高效的开发效率，同时支持灵活的主题定制和动态切换功能。

我来帮您解析dreamview-lang目录下的文件结构和内容。

Ran tool
Read file: frontend/packages/dreamview-lang/package.json
Read file: frontend/packages/dreamview-lang/README.md
Read file: frontend/packages/dreamview-lang/index.ts
Ran tool
Read file: frontend/packages/dreamview-lang/en/index.ts
Read file: frontend/packages/dreamview-lang/en/hello.ts
Read file: frontend/packages/dreamview-lang/en/panels.ts
Read file: frontend/packages/dreamview-lang/en/menu.ts
Read file: frontend/packages/dreamview-lang/en/guide.ts
Ran tool
Read file: frontend/packages/dreamview-lang/zh/index.ts
Read file: frontend/packages/dreamview-lang/zh/hello.ts
Read file: frontend/packages/dreamview-lang/zh/panels.ts
Ran tool
Ran tool
Read file: frontend/packages/dreamview-lang/language-i18next/en/language.ts
Read file: frontend/packages/dreamview-lang/language.stories.tsx
Read file: frontend/packages/dreamview-lang/en/resourceManager.ts
Read file: frontend/packages/dreamview-lang/en/routeEditing.ts
## dreamview-lang 模块解析

基于对 `dreamview-lang` 目录下文件的详细分析，这是一个基于 i18next 的国际化模块，为 DreamView 项目提供多语言支持。以下是详细解析：

### 🏗️ 目录结构
```
dreamview-lang/
├── package.json              # 包配置文件
├── README.md                # 模块说明文档
├── index.ts                 # 主入口文件
├── language.stories.tsx     # Storybook 语言故事
├── .eslintrc.js            # ESLint 配置
├── en/                     # 英文语言包
│   ├── index.ts           # 英文导出文件
│   ├── hello.ts           # 基础问候语
│   ├── panels.ts          # 面板相关文本
│   ├── menu.ts            # 菜单相关文本
│   ├── guide.ts           # 引导教程文本
│   ├── resourceManager.ts # 资源管理器文本
│   ├── routeEditing.ts    # 路径编辑文本
│   └── [other].ts         # 其他功能模块文本
├── zh/                     # 中文语言包
│   ├── index.ts           # 中文导出文件
│   ├── hello.ts           # 基础问候语
│   ├── panels.ts          # 面板相关文本
│   ├── menu.ts            # 菜单相关文本
│   ├── guide.ts           # 引导教程文本
│   ├── resourceManager.ts # 资源管理器文本
│   ├── routeEditing.ts    # 路径编辑文本
│   └── [other].ts         # 其他功能模块文本
└── language-i18next/       # i18next 配置
    ├── en/                # 英文配置
    └── zh/                # 中文配置
```

### 🌐 核心功能

#### 1. **国际化系统架构**
- **双语言支持**: 英文 (en) 和中文 (zh)
- **i18next 集成**: 基于 i18next 和 react-i18next
- **语言检测**: 自动检测浏览器语言
- **本地存储**: 记住用户语言选择

#### 2. **技术栈**
- **i18next**: 国际化框架
- **react-i18next**: React 集成
- **i18next-browser-languagedetector**: 语言检测器
- **TypeScript**: 完整的类型支持

### �� 初始化系统

#### 1. **主入口文件** (`index.ts`)
```typescript
export async function initI18n(resources = translations): Promise<void> {
    await i18next
        .use(LanguageDetector)
        .use(initReactI18next)
        .init({
            resources,
            lng: localStorage.getItem('i18nextLng') || 'en',
            interpolation: {
                escapeValue: false,
            },
        });
}
```

#### 2. **语言类型定义**
```typescript
export type Language = keyof typeof translations;
export const translations = { en, zh };
```

### 📚 语言包结构

#### 1. **基础语言包** (`hello.ts`)
```typescript
// 英文
export const hello = {
    hello: 'hello',
};

// 中文
export const hello = {
    hello: '你好',
};
```

#### 2. **面板语言包** (`panels.ts`)
包含所有面板相关的文本：

**英文示例**:
```typescript
export const panels = {
    // 面板标题
    consoleTitle: 'Console',
    vehicleVizTitle: 'Vehicle Visualization',
    cameraViewTitle: 'Camera View',
    pointCloudTitle: 'Point Cloud',
    
    // 面板描述
    consoleDescription: 'System console panel',
    vehicleVizDescription: 'Main panel to show self-driving vehicle position...',
    
    // 操作文本
    play: 'Play/Run',
    stop: 'Stop',
    zoomIn: 'Zoom In',
    zoomOut: 'Zoom Out',
};
```

**中文示例**:
```typescript
export const panels = {
    // 面板标题
    consoleTitle: '控制台',
    vehicleVizTitle: '车辆可视化',
    cameraViewTitle: '相机视图',
    pointCloudTitle: '点云',
    
    // 面板描述
    consoleDescription: '系统控制台面板',
    vehicleVizDescription: '显示自动驾驶车辆位置、地图数据...',
    
    // 操作文本
    play: '播放/运行',
    stop: '停止',
    zoomIn: '放大',
    zoomOut: '缩小',
};
```

#### 3. **菜单语言包** (`menu.ts`)
包含菜单和设置相关的文本：

**英文示例**:
```typescript
export const personal = {
    setting: 'Settings',
    cloud: 'Cloud Profile',
    guide: 'Use guide',
    document: 'Product Documentation',
    community: 'Apollo Developer Community',
    language: 'Language',
    confirm: 'confirm',
    close: 'Close',
};
```

**中文示例**:
```typescript
export const personal = {
    setting: '设置',
    cloud: '云端配置',
    guide: '使用指南',
    document: '产品文档',
    community: 'Apollo 开发者社区',
    language: '语言',
    confirm: '确认',
    close: '关闭',
};
```

#### 4. **引导教程语言包** (`guide.ts`)
包含用户引导和教程文本：

**英文示例**:
```typescript
export const guide = {
    welcome: 'Welcome',
    modeSelectDesc: 'We provide you with the following visualization template...',
    defaultMode: 'Default Mode',
    perceptionMode: 'Perception Mode',
    enterThisMode: 'Enter this mode',
    skip: 'Skip',
    back: 'Back',
    next: 'Next',
};
```

**中文示例**:
```typescript
export const guide = {
    welcome: '欢迎',
    modeSelectDesc: '我们为您提供以下可视化模板...',
    defaultMode: '默认模式',
    perceptionMode: '感知模式',
    enterThisMode: '进入此模式',
    skip: '跳过',
    back: '返回',
    next: '下一步',
};
```

#### 5. **资源管理器语言包** (`resourceManager.ts`)
包含资源管理相关的文本：

```typescript
export const profileManager = {
    title: 'Resource Manager',
    state: 'State',
    records: 'Records',
    scenarios: 'Scenarios',
    HDMap: 'HDMap',
    vehicle: 'Vehicle',
    V2X: 'V2X',
    dynamical: 'Dynamical Model',
};

export const profileManagerOperate = {
    cancel: 'cancel download',
    download: 'download',
    update: 'update',
    delete: 'delete',
    upload: 'upload',
    reset: 'reset',
};
```

#### 6. **路径编辑语言包** (`routeEditing.ts`)
包含路径编辑功能相关的文本：

```typescript
export const routeEditing = {
    routeEditingBtnOther: 'Routing Editing is suitable for Scenario and SimControl Operation',
    routeCreateCommonBtn: 'Create Common Routing',
    cancel: 'Cancel',
    saveEditing: 'Save Editing',
    create: 'Create',
    backToLastPoint: 'Back to last point',
    backToStartPoint: 'Back to the start point',
    removeLastPoint: 'Remove last point',
    removeAllPoints: 'Remove All points',
};
```

### �� 功能模块覆盖

#### 1. **核心功能模块**
- **面板系统**: 所有可视化面板的文本
- **菜单系统**: 主菜单和设置菜单
- **引导系统**: 用户引导和教程
- **资源管理**: 资源下载和管理
- **路径编辑**: 路径规划和编辑
- **车辆可视化**: 车辆状态显示
- **图表编辑**: 数据图表功能

#### 2. **专业术语**
- **自动驾驶术语**: Planning、Control、Perception 等
- **传感器术语**: LiDAR、Camera、Radar 等
- **地图术语**: HDMap、Waypoint、Routing 等
- **系统术语**: Module Delay、Console、Dashboard 等

### 🔌 集成特性

#### 1. **React 集成**
```typescript
import { useTranslation } from '@dreamview/dreamview-lang';

function MyComponent() {
    const { t } = useTranslation();
    return <div>{t('panels.consoleTitle')}</div>;
}
```

#### 2. **语言切换**
```typescript
// 切换语言
i18next.changeLanguage('zh'); // 切换到中文
i18next.changeLanguage('en'); // 切换到英文
```

#### 3. **本地存储**
- 自动保存用户语言选择到 localStorage
- 页面刷新后保持语言设置

### �� Storybook 集成

#### 语言演示组件
```typescript
export function CommonLanguage() {
    const { t: tCommon } = useTranslation('language');
    
    const onChange = (val: string) => {
        if (val === 'zh') {
            i18next.changeLanguage('zh');
        } else {
            i18next.changeLanguage('en');
        }
    };
    
    return (
        <div>
            <Select defaultValue='en' options={options} onChange={onChange} />
            <div>{tCommon('title')}</div>
        </div>
    );
}
```

### �� 设计特点

#### 1. **模块化组织**
- 按功能模块组织语言包
- 清晰的命名规范
- 易于维护和扩展

#### 2. **类型安全**
- 完整的 TypeScript 支持
- 编译时类型检查
- 自动补全支持

#### 3. **一致性**
- 统一的翻译风格
- 专业术语标准化
- 用户体验一致性

### 🛠️ 开发体验

#### 1. **易于使用**
```typescript
// 简单使用
const { t } = useTranslation();
t('panels.consoleTitle')

// 命名空间使用
const { t } = useTranslation('panels');
t('consoleTitle')
```

#### 2. **易于扩展**
- 添加新语言只需创建对应目录
- 添加新功能只需创建对应文件
- 自动类型推导

#### 3. **调试支持**
- 开发环境显示翻译键
- 缺失翻译警告
- 语言切换实时预览

### 🌍 国际化最佳实践

#### 1. **文本组织**
- 按功能模块分组
- 使用语义化键名
- 避免硬编码文本

#### 2. **翻译质量**
- 专业术语准确翻译
- 保持语言风格一致
- 考虑文化差异

#### 3. **性能优化**
- 按需加载语言包
- 缓存翻译结果
- 最小化包体积

这个国际化模块为 DreamView 项目提供了完整的多语言支持，确保全球用户都能获得良好的本地化体验，同时为开发者提供了便捷的国际化开发工具。