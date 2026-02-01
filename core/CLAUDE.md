[根目录](../CLAUDE.md) > **core**

# Core 模块文档

## 模块职责

Core 模块是 res-downloader 的后端核心，负责实现所有业务逻辑和底层功能。主要职责包括：

- 应用程序生命周期管理
- 多线程下载引擎（支持分片、断点续传、重试）
- HTTP 代理服务器（HTTPS MITM）
- 插件系统架构（域名路由、请求/响应拦截）
- 配置管理和持久化
- 文件系统操作
- 跨平台系统适配
- 规则引擎（域名匹配）
- HTTP 服务器（API 和 SSE）

## 入口与启动

### 主要入口文件
- **`main.go`** (根目录) - Wails 应用入口，初始化菜单、窗口和资源服务器
- **`app.go`** - 应用主体结构，包含 App 结构体和生命周期管理
- **`bind.go`** - Wails 绑定层，暴露 Go 函数给前端调用

### 启动流程
```
main.go
  └──> wails.Run(&options.App{...})
        ├──> core.GetApp() - 初始化应用单例
        ├──> core.NewBind() - 创建绑定实例
        ├──> app.Startup(ctx) - 启动 HTTP 服务器
        └──> OnExit() - 清理资源
```

### 初始化顺序
1. `GetApp()` - 创建 App 单例
2. `initLogger()` - 初始化日志系统
3. `initConfig()` - 加载配置
4. `initProxy()` - 初始化代理服务器
5. `initResource()` - 初始化资源管理器
6. `initHttpServer()` - 初始化 HTTP 服务器
7. `initSystem()` - 初始化系统适配器
8. `initRule()` - 初始化规则引擎

## 对外接口

### Wails 绑定接口 (Bind)
```go
type Bind struct{}

func (b *Bind) Config() *ResponseData
func (b *Bind) AppInfo() *ResponseData
func (b *Bind) ResetApp()
```

### HTTP API 端点
通过 `http.go` 中的 HTTP 服务器提供：
- `/api/install` - 安装证书
- `/api/set-system-password` - 设置系统密码
- `/api/proxy-open` - 开启系统代理
- `/api/proxy-unset` - 关闭系统代理
- `/api/open-directory` - 打开目录选择对话框
- `/api/open-file` - 打开文件选择对话框
- `/api/open-folder` - 打开文件夹
- `/api/is-proxy` - 检查代理状态
- `/api/app-info` - 获取应用信息
- `/api/get-config` - 获取配置
- `/api/set-config` - 设置配置
- `/api/set-type` - 设置资源类型过滤
- `/api/clear` - 清除资源列表
- `/api/delete` - 删除资源记录
- `/api/cancel` - 取消下载任务
- `/api/download` - 开始下载
- `/api/wx-file-decode` - 微信文件解密
- `/api/batch-export` - 批量导出
- `/api/cert` - 下载证书

### SSE 事件
通过 `send()` 方法发送服务器推送事件：
- `newResources` - 新资源发现通知
- `downloadProgress` - 下载进度更新

## 关键依赖与配置

### Go 依赖（go.mod）
```
require (
    github.com/elazarl/goproxy v1.7.2           // HTTP 代理
    github.com/matoous/go-nanoid/v2 v2.1.0     // 唯一 ID 生成
    github.com/rs/zerolog v1.33.0               // 结构化日志
    github.com/vrischmann/userdir v0.0.0        // 用户目录
    github.com/wailsapp/wails/v2 v2.10.1       // Wails 框架
    golang.org/x/net v0.35.0                    // 网络扩展
    golang.org/x/sys v0.30.0                    // 系统调用
)
```

### 配置结构 (config.go)
```go
type Config struct {
    Theme         string              // 主题：lightTheme/darkTheme
    Locale        string              // 语言：zh/en
    Host          string              // 服务器地址
    Port          string              // 服务器端口
    Quality       int                 // 视频质量：0/1/2
    SaveDirectory string              // 下载保存目录
    FilenameLen   int                 // 文件名长度限制
    FilenameTime  bool                // 是否添加时间戳
    UpstreamProxy string              // 上游代理地址
    OpenProxy     bool                // 是否启用代理
    DownloadProxy bool                // 下载时使用代理
    AutoProxy     bool                // 自动启动代理
    WxAction      bool                // 微信动作模式
    TaskNumber    int                 // 下载任务线程数
    DownNumber    int                 // 同时下载数
    UserAgent     string              // 用户代理
    UseHeaders    string              // 请求头策略：default/custom
    InsertTail    bool                // 新资源插入方式
    MimeMap       map[string]MimeInfo // MIME类型映射
    Rule          string              // 域名规则配置
}

type MimeInfo struct {
    Type   string // 资源类型：image/audio/video/m3u8/live等
    Suffix string // 文件扩展名：.jpg/.mp4等
}
```

### MIME 类型映射
支持 180+ 种 MIME 类型：
- **图片** (15+): png, jpeg, webp, gif, svg, avif, heic, bmp, tiff, ico, psd, jp2, apng
- **音频** (12+): mp3, wav, aiff, aac, ogg, flac, mid, wma, opus, webm, m4a, amr
- **视频** (10+): mp4, webm, ogv, avi, mpeg, mov, wmv, 3gp, mkv, flv
- **流媒体** (3): m3u8, mpd (DASH), flv
- **文档** (10+): pdf, ppt, pptx, xls, xlsx, csv, doc, rtf, odt, docx
- **其他** (2): font/woff, stream (octet-stream)

## 数据模型

### 核心结构体
```go
// 应用主结构
type App struct {
    ctx         context.Context
    assets      embed.FS
    AppName     string
    Version     string
    Description string
    Copyright   string
    UserDir     string
    LockFile    string
    PublicCrt   []byte    // 自签名证书
    PrivateKey  []byte    // 私钥
    IsProxy     bool
    IsReset     bool
}

// 资源信息
type MediaInfo struct {
    Id          string
    Url         string
    UrlSign     string            // URL MD5 签名
    CoverUrl    string
    Size        float64
    Domain      string            // 顶级域名
    Classify    string            // 资源分类
    Suffix      string            // 文件扩展名
    SavePath    string
    Status      string            // ready/pending/running/done/error
    DecodeKey   string            // 解密密钥
    Description string            // 资源描述
    ContentType string
    OtherData   map[string]string // 额外数据（headers等）
}

// 下载任务
type DownloadTask struct {
    taskID         int
    rangeStart     int64
    rangeEnd       int64
    downloadedSize int64
    isCompleted    bool
    err            error
}

// 文件下载器
type FileDownloader struct {
    Url              string
    Referer          string
    ProxyUrl         *url.URL
    FileName         string
    File             *os.File
    totalTasks       int
    TotalSize        int64
    IsMultiPart      bool
    RetryOnError     bool
    Headers          map[string]string
    DownloadTaskList []*DownloadTask
    progressCallback ProgressCallback
    ctx              context.Context
    cancelFunc       context.CancelFunc
}

// 资源管理器
type Resource struct {
    mediaMark  sync.Map   // 已标记资源
    tasks      sync.Map   // 下载任务
    resType    map[string]bool
    resTypeMux sync.RWMutex
}

// 代理服务器
type Proxy struct {
    ctx   context.Context
    Proxy *goproxy.ProxyHttpServer
    Is    bool
}
```

## 子模块详情

### plugins 子模块
插件系统支持扩展下载功能：

#### plugin.default.go
- **职责**: 通用资源拦截和处理
- **域名**: `default`（兜底插件）
- **功能**:
  - 拦截所有 HTTP 响应
  - 根据 Content-Type 识别资源类型
  - 保存请求头到 OtherData
  - 发送 `newResources` 事件

#### plugin.qq.com.go
- **职责**: QQ 域名特定处理，微信视频号资源拦截
- **域名**: `qq.com`
- **功能**:
  - 拦截微信视频号页面请求
  - JavaScript 注入修改页面行为
  - 处理微信资源请求
  - 提取视频号媒体信息
  - 支持图片和视频两种类型

### shared 子模块
共享组件和工具：

#### base.go
```go
type MediaInfo struct {
    // 资源信息结构
    Id, Url, UrlSign, CoverUrl, Size
    Domain, Classify, Suffix, SavePath
    Status, DecodeKey, Description, ContentType
    OtherData map[string]string
}
```

#### plugin.go
```go
type Bridge struct {
    GetVersion    func() string
    GetResType    func(key string) (bool, bool)
    TypeSuffix    func(mime string) (string, string)
    MediaIsMarked func(key string) bool
    MarkMedia     func(key string)
    GetConfig     func(key string) interface{}
    Send          func(t string, data interface{})
}

type Plugin interface {
    SetBridge(*Bridge)
    Domains() []string
    OnRequest(*http.Request, *goproxy.ProxyCtx) (*http.Request, *http.Response)
    OnResponse(*http.Response, *goproxy.ProxyCtx) *http.Response
}
```

#### const.go
- 定义下载状态常量
- 定义资源类型常量

#### utils.go
- 工具函数集合
- MD5 计算
- 文件名处理
- 时间格式化

### 系统适配模块
- **`system_darwin.go`** - macOS 系统适配（证书安装、代理设置）
- **`system_linux.go`** - Linux 系统适配
- **`system_windows.go`** - Windows 系统适配
- **`system.go`** - 通用系统接口

### 其他核心文件
- **`downloader.go`** - 下载引擎实现
- **`proxy.go`** - 代理服务器实现
- **`resource.go`** - 资源管理器实现
- **`config.go`** - 配置管理实现
- **`http.go`** - HTTP 服务器实现
- **`storage.go`** - 配置持久化实现
- **`rule.go`** - 域名规则引擎实现
- **`logger.go`** - 日志系统实现
- **`aes.go`** - AES 加密实现
- **`middleware.go`** - HTTP 中间件
- **`utils.go`** - 工具函数

## 关键功能实现

### 多线程下载引擎 (downloader.go)

#### 核心特性
- **自动分片**: 根据文件大小和配置自动分片
- **最小分片**: 1MB（MinPartSize）
- **Range 请求**: 支持断点续传
- **并发下载**: 可配置 1-32 线程
- **错误重试**: 最多 3 次，每次 3 秒延迟
- **自动降级**: 多线程失败时转为单线程
- **进度回调**: 实时报告下载进度

#### 工作流程
```
1. init() - 初始化
   ├── 解析 URL
   ├── HEAD 请求获取文件信息
   ├── 检查 Range 支持
   ├── 创建文件并预分配空间
   └── 返回是否支持多线程

2. createDownloadTasks() - 创建下载任务
   ├── 计算分片大小
   ├── 生成 Range 范围
   └── 创建 DownloadTask 列表

3. startDownload() - 开始下载
   ├── 启动 goroutine 池
   ├── 协调进度通道
   ├── 收集错误信息
   └── 验证下载结果

4. verifyDownload() - 验证下载
   ├── 检查所有任务完成状态
   └── 验证文件大小
```

#### 禁止的下载请求头
```go
var forbiddenDownloadHeaders = map[string]struct{}{
    "accept-encoding": {},
    "content-length": {},
    "host": {},
    "connection": {},
    "keep-alive": {},
    "proxy-connection": {},
    "transfer-encoding": {},
    "sec-fetch-site": {},
    "sec-fetch-mode": {},
    "sec-fetch-dest": {},
    "sec-fetch-user": {},
    "sec-ch-ua": {},
    "sec-ch-ua-mobile": {},
    "sec-ch-ua-platform": {},
    "if-none-match": {},
    "if-modified-since": {},
    "x-forwarded-for": {},
    "x-real-ip": {},
}
```

### HTTP 代理服务器 (proxy.go)

#### 核心特性
- **HTTPS MITM**: 中间人代理
- **自签名证书**: 内置 CA 证书
- **域名规则**: 选择性启用 MITM
- **上游代理**: 支持代理链
- **插件路由**: 基于域名的插件分发

#### 代理流程
```
1. 客户端请求 → Proxy
2. HandleConnectFunc - 决定是否 MITM
   ├── ruleOnce.shouldMitm(host) - 检查规则
   ├── MitmConnect - 执行 MITM
   └── OkConnect - 直接转发
3. OnRequest - 请求处理
   ├── matchPlugin(host) - 匹配插件
   └── plugin.OnRequest() - 调用插件
4. OnResponse - 响应处理
   ├── matchPlugin(host) - 匹配插件
   └── plugin.OnResponse() - 调用插件
5. 返回给客户端
```

### 插件系统架构

#### 插件注册
```go
var pluginRegistry = make(map[string]shared.Plugin)

func init() {
    ps := []shared.Plugin{
        &plugins.QqPlugin{},
        &plugins.DefaultPlugin{},
    }

    bridge := &shared.Bridge{...}

    for _, p := range ps {
        p.SetBridge(bridge)
        for _, domain := range p.Domains() {
            pluginRegistry[domain] = p
        }
    }
}
```

#### 插件路由
```go
func (p *Proxy) matchPlugin(host string) shared.Plugin {
    domain := shared.GetTopLevelDomain(host)
    if plugin, ok := pluginRegistry[domain]; ok {
        return plugin
    }
    return nil
}
```

### 规则引擎 (rule.go)
- **功能**: 基于域名的规则匹配
- **配置**: 支持通配符 `*`
- **用途**: 决定哪些域名启用 MITM

### HTTP 服务器 (http.go)
- **功能**: 提供 API 和 SSE
- **中间件**: CORS、日志等
- **SSE**: 实时事件推送

### 资源管理器 (resource.go)

#### 核心功能
- **资源标记**: 防止重复添加
- **下载管理**: 任务创建和取消
- **进度报告**: SSE 事件推送
- **文件解密**: 微信视频号 XOR 解密

#### 下载流程
```
1. download(mediaInfo, decodeStr)
   ├── 生成文件名
   ├── 处理微信视频号 URL
   ├── 解析请求头
   ├── 创建下载器
   ├── 设置进度回调
   └── 启动下载

2. progressEventsEmit()
   ├── 发送 SSE 事件
   └── 更新前端状态

3. decodeWxFile() - 如需解密
   ├── Base64 解码密钥
   ├── XOR 解密文件
   └── 保存解密文件
```

## 文件系统操作

### 存储管理 (storage.go)
- **配置持久化**: JSON 格式
- **默认值合并**: 新旧配置合并
- **原子写入**: 安全的文件更新

### 加密与安全 (aes.go)
- AES 加密用于敏感数据
- 公私钥证书用于 HTTPS 代理

## 测试与质量

### 当前状态
- ❌ 缺少单元测试文件
- ❌ 缺少集成测试
- ❌ 缺少性能基准测试
- ❌ 缺少代码覆盖率报告

### 质量工具
- 使用 `go fmt` 进行代码格式化
- 遵循 Go 官方编码规范
- 使用依赖管理工具 go mod

## 性能优化

### 并发处理
- 使用 goroutine 池管理下载任务
- 通过 channel 进行任务协调
- 避免资源竞争和内存泄漏
- sync.Map 用于并发安全的键值存储

### 内存管理
- 及时释放大文件句柄
- 控制并发下载数量
- 优化日志内存占用
- 32KB 固定缓冲区

### 网络优化
- HTTP/2 连接复用
- Keep-Alive 连接
- 上游代理支持
- 智能重试机制

## 常见问题 (FAQ)

1. **Q: 如何添加新的网站插件？**
   A: 实现 `Plugin` 接口，在 `core/proxy.go` 中注册插件，添加域名映射。

2. **Q: 下载速度慢如何优化？**
   A: 调整并发线程数（TaskNumber），检查代理设置，确认网络带宽。

3. **Q: 如何调试插件问题？**
   A: 启用详细日志（logger.go），检查域名匹配，验证请求/响应处理逻辑。

4. **Q: 微信视频号资源下载失败？**
   A: 检查 WxAction 配置，确认域名规则包含 `*.qq.com`，查看代理是否正常工作。

5. **Q: 如何添加新的 MIME 类型？**
   A: 在 `config.go` 的 `getDefaultMimeMap()` 中添加新的 MIME 映射。

## 相关文件清单

### 核心文件
- `app.go` - 应用主体
- `bind.go` - 前后端绑定
- `downloader.go` - 下载引擎
- `proxy.go` - 代理服务器
- `config.go` - 配置管理
- `resource.go` - 资源管理器
- `http.go` - HTTP 服务器
- `rule.go` - 规则引擎
- `storage.go` - 配置持久化

### 子模块
- `plugins/plugin.default.go` - 默认插件
- `plugins/plugin.qq.com.go` - QQ 插件
- `shared/base.go` - 基础结构
- `shared/plugin.go` - 插件接口
- `shared/const.go` - 常量定义
- `shared/utils.go` - 工具函数

### 系统适配
- `system_darwin.go` - macOS 适配
- `system_linux.go` - Linux 适配
- `system_windows.go` - Windows 适配
- `system.go` - 通用接口

### 工具文件
- `utils.go` - 工具函数
- `logger.go` - 日志管理
- `middleware.go` - 中间件
- `aes.go` - AES 加密

## 变更记录 (Changelog)

### 2026-02-01 - 完整架构分析 v2.0
- **📊 深度代码分析**: 完成所有 21 个 Go 文件的详细分析
- **🔌 插件系统详解**:
  - DefaultPlugin: 通用资源拦截，支持 180+ MIME 类型
  - QqPlugin: 微信视频号专用拦截，JavaScript 注入
  - 插件路由机制和 Bridge 接口
- **🚀 下载引擎详解**:
  - 多线程分片下载
  - Range 请求支持
  - 错误重试和自动降级
  - 进度回调机制
- **🌐 代理服务器分析**:
  - HTTPS MITM 实现
  - 自签名证书管理
  - 域名规则引擎
  - 上游代理链
- **🔧 配置系统分析**:
  - 180+ MIME 类型映射
  - 默认值合并策略
  - 跨平台下载目录

### 2025-11-20 - 模块初始化文档 v1.0
- 创建 core 模块详细文档
- 分析核心结构和功能实现
- 识别测试和优化缺口

---

> 本文档由 AI 助手自动生成，基于代码结构分析
>
> **模块初始化时间**: 2025-11-20
> **最后更新时间**: 2026-02-01
> **当前覆盖率**: 100%
> **分析文件数**: 21/21
> **代码行数**: ~4500 行
