[根目录](../CLAUDE.md) > **frontend**

# Frontend 模块文档

## 模块职责

Frontend 模块是 res-downloader 的用户界面层，基于 Vue 3 + TypeScript 构建现代化桌面应用界面。主要职责包括：

- 用户界面展示和交互
- 下载任务管理和监控
- 应用配置和设置界面
- 多语言支持和主题切换
- 与后端 Go 服务的通信
- 响应式布局和用户体验

## 入口与启动

### 主要入口文件
- **`src/main.ts`** - 应用程序入口，初始化 Vue 实例
- **`src/App.vue`** - 根组件，提供全局配置和主题
- **`index.html`** - HTML 模板文件

### 启动流程
1. 加载 CSS 样式文件
2. 创建 Vue 应用实例
3. 注册 Pinia 状态管理
4. 注册 Vue Router 路由
5. 注册 Vue i18n 国际化
6. 挂载应用到 DOM

## 对外接口

### API 通信层
- **`src/api/app.ts`** - 后端 API 调用封装
- **`src/api/request.ts`** - HTTP 请求工具
- 使用 Wails 生成的绑定：`@/wailsjs/go/core/Bind`

### 路由结构
```typescript
// 主布局组件
/components/layout/Index.vue
  ├── /index - 主页面 (下载管理)
  └── /setting - 设置页面
```

## 关键依赖与配置

### 前端技术栈
- **Vue 3.2.37** - 渐进式 JavaScript 框架
- **TypeScript 4.6.4** - 类型安全的 JavaScript
- **Naive UI 2.38.2** - Vue 3 组件库
- **Pinia 2.1.7** - Vue 状态管理
- **Vue Router 4.3.3** - 客户端路由
- **Vue i18n 11.1.3** - 国际化支持

### 构建工具
- **Vite 3.0.7** - 现代前端构建工具
- **Vue TSC 1.8.27** - TypeScript 编译器
- **PostCSS + Tailwind CSS** - CSS 处理和样式框架
- **Sass 1.77.6** - CSS 预处理器

### 开发工具
- **unplugin-auto-import** - 自动导入 API
- **unplugin-vue-components** - 组件自动导入
- **@vitejs/plugin-vue** - Vue 单文件组件支持

## 数据模型与状态管理

### 全局状态 (Pinia Stores)
- **`stores/index.ts`** - 主要状态管理
  - 应用信息 (`appInfo`)
  - 全局配置 (`globalConfig`)
  - 环境信息 (`envInfo`)
  - 代理状态 (`isProxy`)
- **`stores/event.ts`** - 事件处理和消息系统

### 配置数据结构
```typescript
interface Config {
  Theme: "lightTheme" | "darkTheme"
  Locale: "zh" | "en"
  Host: string
  Port: string
  Quality: number
  SaveDirectory: string
  // ... 更多配置项
}
```

## 组件架构

### 布局组件
- **`layout/Index.vue`** - 主布局容器
- **`layout/Sider.vue`** - 侧边栏导航

### 功能组件
- **`Action.vue`** - 操作按钮组件
- **`ActionDesc.vue`** - 操作描述组件
- **`ImportJson.vue`** - JSON 导入组件
- **`Password.vue`** - 密码输入组件
- **`Preview.vue`** - 文件预览组件
- **`Screen.vue`** - 屏幕截图组件
- **`ShowLoading.vue`** - 加载状态组件

### 工具组件
- **`NaiveProvider.vue`** - Naive UI 全局配置
- **`Footer.vue`** - 页脚组件

## 页面结构

### 主页面 (`views/index.vue`)
主要功能区域：
- 下载任务列表
- 添加新任务
- 批量操作
- 进度监控

### 设置页面 (`views/setting.vue`)
配置选项：
- 主题和语言设置
- 下载参数配置
- 代理设置
- 高级选项

## 国际化支持

### 语言文件
- **`src/locales/zh.json`** - 中文语言包
- **`src/locales/en.json`** - 英文语言包

### i18n 配置 (`src/i18n.ts`)
- 支持中英文切换
- 动态语言加载
- 与全局状态同步

## 样式与主题

### 主题系统
- 支持明亮主题和暗黑主题
- 动态切换 CSS 类
- Naive UI 主题配置

### 样式文件
- **`src/assets/css/base.css`** - 基础样式重置
- **`src/assets/css/main.css`** - 主要样式定义
- **`postcss.config.js`** - PostCSS 配置

### UI 组件库
- 基于 Naive UI 组件系统
- 响应式设计支持
- 自定义主题颜色

## 特殊功能

### 视频播放支持
- **flv.js 1.6.2** - FLV 视频格式支持
- **video.js 8.22.0** - 通用视频播放器
- 支持视频预览和播放

### 文件解密
- **`src/assets/js/decrypt.js`** - 文件解密逻辑
- 前端文件内容处理
- 安全的加密数据传输

### 图标系统
- **@vicons/ionicons5** - Ionicons 图标库
- **@iconify/vue** - 图标组件化支持

## 开发配置

### TypeScript 配置
- 严格类型检查
- 路径别名配置 (`@/` 指向 `src/`)
- 组件类型自动生成

### 自动导入配置
- Vue API 自动导入
- Naive UI 组件自动导入
- 自定义组件类型提示

### Vite 配置优化
- 开发服务器热重载
- 生产环境构建优化
- 资源处理和压缩

## 测试与质量

### 当前状态
- ❌ 缺少单元测试 (Vue Test Utils)
- ❌ 缺少组件测试
- ❌ 缺少端到端测试
- ❌ 缺少类型检查覆盖率

### 代码质量
- ✅ TypeScript 严格模式
- ✅ ESLint 代码规范检查
- ✅ Prettier 代码格式化
- ✅ Vite 构建优化

## 性能优化

### 组件优化
- 使用 Vue 3 Composition API
- 组件懒加载和代码分割
- 响应式数据优化

### 构建优化
- Vite 快速热重载
- 资源压缩和缓存
- Tree shaking 减少包体积

### 运行时优化
- 虚拟滚动处理大列表
- 防抖和节流优化交互
- 内存泄漏防护

## 常见问题 (FAQ)

1. **Q: 如何添加新的国际化语言？**
   A: 在 `src/locales/` 目录添加新的语言文件，更新 `i18n.ts` 配置。

2. **Q: 如何自定义主题颜色？**
   A: 修改 Naive UI 主题配置，更新 CSS 变量定义。

3. **Q: 如何与后端通信？**
   A: 使用 Wails 生成的绑定接口，通过 `stores/index.ts` 统一管理。

## 相关文件清单

### 入口文件
- `src/main.ts` - 应用入口
- `src/App.vue` - 根组件
- `index.html` - HTML 模板

### 配置文件
- `package.json` - 项目依赖
- `vite.config.ts` - Vite 配置
- `tsconfig.json` - TypeScript 配置
- `postcss.config.js` - PostCSS 配置

### 核心目录
- `src/api/` - API 通信层
- `src/components/` - 可复用组件
- `src/views/` - 页面组件
- `src/stores/` - 状态管理
- `src/router/` - 路由配置
- `src/assets/` - 静态资源

### 样式文件
- `src/assets/css/` - 样式文件
- `src/assets/image/` - 图片资源

## 变更记录 (Changelog)

### 2025-11-20 - 前端模块初始化文档
- 创建 frontend 模块详细文档
- 分析 Vue 3 + TypeScript 技术栈
- 识别组件架构和状态管理模式

---

> 本文档由 AI 助手自动生成，基于前端代码结构分析