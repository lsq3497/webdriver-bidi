# WebDriver BiDi

WebDriver BiDi is a bidirectional protocol for browser automation,
building on and extending [WebDriver](https://w3c.github.io/webdriver/).

WebDriver BiDi is a living standard that continuously gets new features added. For more info, consult these resources:

- An [explainer](./explainer.md) with more background and goals
- A [roadmap](./roadmap.md) based on real-world end-to-end user scenarios
- Detailed [proposals](./proposals/) for the initial protocol
- A [spec](https://w3c.github.io/webdriver-bidi/) under active development
- [Browser-compat-data](https://github.com/mdn/browser-compat-data/tree/main/webdriver/bidi) with the current implementation status of the protocol

## Status

[![test](https://github.com/w3c/webdriver-bidi/actions/workflows/test.yml/badge.svg)](https://github.com/w3c/webdriver-bidi/actions/workflows/test.yml)

## How to build the specification locally

We use [bikeshed](https://tabatkins.github.io/bikeshed/) to generate the specification.

Make sure you have the [right version of python](https://tabatkins.github.io/bikeshed/#install-py3) installed.

Now you can run in your terminal:

```bash
./scripts/build.sh
```

This script installs `bikeshed` (if not installed yet) and generates an
`index.html` file for the specification.

Later on, you can use the `--upgrade` argument to force installing a newer version.

## How to generate CDDL locally

Make sure you have [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
and [rust](https://www.rust-lang.org/tools/install) installed.

Now you can run in your terminal:

```bash
./scripts/test.sh
```

This script installs required `npm` and `cargo` packages (if not installed yet)
and generates the CDDL files for the remote end (`remote.cddl`) and the client
(`local.cddl`).

Later on, you can use the `--upgrade` argument to force installing newer versions.

---

# WebDriver BiDi 协议结构分析与改进方向

## 一、协议结构分析

### 1.1 协议架构概述

WebDriver BiDi 协议基于以下核心架构设计：

#### 传输层
- **传输机制**: WebSocket（全双工通信）
- **消息格式**: 基于 JSON-RPC 2.0，但移除了 `jsonrpc` 属性，并采用自定义错误格式
- **协议描述**: 使用 OpenRPC 规范进行 API 描述（见 `proposals/openrpc.json`）

#### 消息结构
根据 `index.bs` 中的 CDDL 定义，协议消息结构如下：

**命令（Command）结构**:
```cddl
Command = {
  id: js-uint,
  CommandData,
  Extensible,
}
```

**响应（Response）结构**:
```cddl
CommandResponse = {
  type: "success",
  id: js-uint,
  result: ResultData,
  Extensible
}

ErrorResponse = {
  type: "error",
  id: js-uint / null,
  error: ErrorCode,
  message: text,
  ? stacktrace: text,
  Extensible
}
```

**事件（Event）结构**:
```cddl
Event = {
  type: "event",
  EventData,
  Extensible
}
```

### 1.2 模块化设计

协议采用模块化架构，当前定义的模块包括：

1. **Browser** - 浏览器级别操作
2. **BrowsingContext** - 浏览上下文管理
3. **Emulation** - 设备模拟
4. **Input** - 输入操作
5. **Network** - 网络拦截与监控
6. **Script** - 脚本执行
7. **Session** - 会话管理
8. **Storage** - 存储操作
9. **WebExtension** - Web 扩展支持

每个模块包含：
- **命令（Commands）**: 客户端到服务器的操作请求
- **事件（Events）**: 服务器到客户端的通知
- **结果类型（Result Types）**: 命令的返回数据结构

### 1.3 事件订阅机制

协议实现了灵活的事件订阅系统（见 `index.bs` 第 2081-2299 行）：

**订阅结构**:
- `subscription id`: UUID 字符串，唯一标识订阅
- `event names`: 事件名称集合
- `top-level traversable ids`: 目标浏览上下文 ID 集合
- `user context ids`: 用户上下文 ID 集合

**订阅管理**:
- `session.subscribe`: 启用事件订阅，支持全局订阅或针对特定上下文订阅
- `session.unsubscribe`: 取消事件订阅，支持按属性或按订阅 ID 取消
- 支持引用计数机制，允许多个客户端独立订阅/取消订阅同一事件

### 1.4 扩展性设计

协议通过以下机制支持扩展：

1. **Extensible 字段**: 所有消息结构都包含 `Extensible` 字段，允许添加自定义属性
2. **扩展模块**: 实现可以定义扩展模块，模块名必须包含单个冒号（`:`）
3. **外部规范扩展**: 其他规范可以定义自己的模块，但不能使用冒号字符

### 1.5 协议文件组织

- **`index.bs`**: 主规范文件，使用 Bikeshed 格式编写，包含完整的协议定义
- **`proposals/openrpc.json`**: OpenRPC 格式的 API 描述，用于生成客户端绑定
- **`proposals/core.md`**: 核心功能提案文档
- **CDDL 定义**: 在 `index.bs` 中使用 CDDL 定义数据结构，可通过脚本生成 `remote.cddl` 和 `local.cddl`

---

## 二、改进方向可行性分析

### 2.1 增强事件与消息语义

#### 可行性评估: ⭐⭐⭐⭐⭐ (高度可行)

**当前状态分析**:
- 协议已具备完善的事件订阅机制（`session.subscribe`/`session.unsubscribe`）
- 支持基于上下文的事件过滤（`top-level traversable ids`）
- 事件系统采用模块化设计，易于扩展新事件类型

**实施建议**:

1. **事件类型扩展**:
   - 在 `index.bs` 的相应模块中添加新事件定义
   - 例如，在 `browsingContext` 模块中添加 `browsingContext.domContentLoaded` 和 `browsingContext.load` 事件
   - 在 `script` 模块中添加 `script.runtimeError` 事件，包含错误堆栈信息

2. **动态事件过滤**:
   - 扩展 `session.SubscribeParameters` 结构，添加过滤条件字段：
   ```cddl
   session.SubscribeParameters = {
     events: [+text],
     ? contexts: [+browsingContext.BrowsingContext],
     ? userContexts: [+browsingContext.UserContext],
     ? filters: EventFilters,  // 新增过滤条件
   }
   
   EventFilters = {
     ? urlPattern: text,        // URL 模式匹配
     ? eventType: text,          // 事件类型过滤
     ? elementSelector: text,    // 元素选择器过滤
   }
   ```

3. **JavaScript 错误捕获**:
   - 在 `script` 模块中添加 `script.runtimeError` 事件：
   ```cddl
   script.RuntimeError = {
     method: "script.runtimeError",
     params: {
       realm: script.Realm,
       error: script.RemoteValue,
       stackTrace: [+script.StackTraceFrame],
       timestamp: js-uint,
     }
   }
   ```

**修改文件**:
- `index.bs`: 在相应模块的事件定义部分添加新事件
- `proposals/openrpc.json`: 更新 OpenRPC 定义以包含新事件

---

### 2.2 协议命令的扩展与增强

#### 可行性评估: ⭐⭐⭐⭐ (可行)

**当前状态分析**:
- 命令系统采用模块化设计，易于添加新命令
- 命令支持参数化配置（`params` 字段）
- 命令可以异步执行，支持并发

**实施建议**:

1. **复合命令定义**:
   - 创建新的 `composite` 模块，定义复合命令：
   ```cddl
   composite.FillFormAndSubmit = {
     method: "composite.fillFormAndSubmit",
     params: {
       context: browsingContext.BrowsingContext,
       formSelector: text,
       fields: [+composite.FormField],
       ? delay: js-uint,        // 延迟时间
       ? retryCount: js-uint,   // 重试次数
     }
   }
   
   composite.FormField = {
     selector: text,
     value: text,
     ? action: "type" / "select" / "check",
   }
   ```

2. **命令参数增强**:
   - 在现有命令的 `params` 中添加可选配置字段：
   ```cddl
   browsingContext.NavigateParameters = {
     context: browsingContext.BrowsingContext,
     url: text,
     ? wait: "none" / "interactive" / "complete",  // 等待策略
     ? timeout: js-uint,                            // 超时时间
   }
   ```

3. **批量命令支持**:
   - 扩展命令结构，支持批量执行：
   ```cddl
   session.BatchCommand = {
     method: "session.batch",
     params: {
       commands: [+Command],      // 命令数组
       ? stopOnError: bool,       // 遇到错误是否停止
       ? parallel: bool,          // 是否并行执行
     }
   }
   ```

**修改文件**:
- `index.bs`: 添加 `composite` 模块定义，或扩展现有模块
- `proposals/openrpc.json`: 更新命令定义

**注意事项**:
- 复合命令的实现需要在 remote end 端执行多个步骤
- 需要考虑错误处理和回滚机制
- 需要确保与现有命令的兼容性

---

### 2.3 权限与安全控制

#### 可行性评估: ⭐⭐⭐ (中等可行)

**当前状态分析**:
- 协议目前缺乏显式的权限控制机制
- 会话管理（`session.new`）支持 capabilities 配置，可以扩展

**实施建议**:

1. **权限管理扩展**:
   - 在 `session.new` 命令中添加权限配置：
   ```cddl
   session.NewParameters = {
     capabilities: {...},
     ? permissions: PermissionsConfig,  // 新增权限配置
   }
   
   PermissionsConfig = {
     ? allowedDomains: [+text],          // 域名白名单
     ? blockedDomains: [+text],         // 域名黑名单
     ? allowedFeatures: [+text],        // 允许的功能列表
     ? blockedFeatures: [+text],        // 禁止的功能列表
   }
   ```

2. **安全沙盒机制**:
   - 在浏览上下文创建时应用安全策略：
   ```cddl
   browsingContext.CreateParameters = {
     type: browsingContext.BrowsingContextType,
     ? sandbox: SandboxConfig,
   }
   
   SandboxConfig = {
     ? allowLocalStorage: bool,
     ? allowCookies: bool,
     ? allowFileAccess: bool,
     ? allowClipboard: bool,
   }
   ```

3. **权限检查命令**:
   - 添加权限查询命令：
   ```cddl
   session.CheckPermission = {
     method: "session.checkPermission",
     params: {
       feature: text,
       ? context: browsingContext.BrowsingContext,
     }
   }
   ```

**修改文件**:
- `index.bs`: 扩展 `session.new` 和 `browsingContext.create` 的参数定义
- 需要在 remote end 实现权限检查逻辑

**挑战与考虑**:
- 权限控制的实现需要浏览器端的深度集成
- 需要定义清晰的权限模型和策略
- 可能影响现有功能的兼容性
- 需要与浏览器安全模型协调

---

### 2.4 数据交换与消息格式扩展

#### 可行性评估: ⭐⭐⭐⭐⭐ (高度可行)

**当前状态分析**:
- 协议已支持 `Extensible` 字段，允许添加自定义属性
- JSON 格式天然支持复杂数据结构
- 消息格式基于 JSON-RPC，易于扩展

**实施建议**:

1. **自定义数据结构支持**:
   - 利用现有的 `Extensible` 机制，允许客户端定义扩展字段
   - 在命令和事件的 `params` 中添加 `extensions` 字段：
   ```cddl
   ExtensibleParams = {
     *text => any,
     ? extensions: {*text => any},  // 显式扩展字段
   }
   ```

2. **高效数据传输优化**:
   - 添加数据压缩支持：
   ```cddl
   session.ConfigureTransport = {
     method: "session.configureTransport",
     params: {
       ? compression: "none" / "gzip" / "brotli",
       ? chunkSize: js-uint,           // 分块大小
       ? batchSize: js-uint,           // 批量传输大小
     }
   }
   ```

3. **大数据传输命令**:
   - 为大数据量操作添加流式传输：
   ```cddl
   browsingContext.CaptureScreenshotStream = {
     method: "browsingContext.captureScreenshotStream",
     params: {
       context: browsingContext.BrowsingContext,
       ? format: "base64" / "binary",
       ? chunked: bool,                // 是否分块传输
     }
   }
   ```

**修改文件**:
- `index.bs`: 扩展消息结构定义，添加传输配置命令
- `proposals/openrpc.json`: 更新数据结构定义

**优势**:
- 利用现有扩展机制，实现成本低
- 向后兼容，不影响现有功能
- 可以逐步引入，无需一次性重构

---

### 2.5 AI 和自动化优化支持

#### 可行性评估: ⭐⭐ (较低可行性)

**当前状态分析**:
- 协议专注于浏览器自动化控制，不涉及 AI 功能
- 缺乏执行历史记录和性能数据收集机制

**实施建议**:

1. **性能监控扩展**:
   - 添加性能数据收集事件：
   ```cddl
   performance.Metrics = {
     method: "performance.metrics",
     params: {
       context: browsingContext.BrowsingContext,
       metrics: {
         loadTime: js-uint,
         domContentLoadedTime: js-uint,
         firstPaint: js-uint,
         memoryUsage: js-uint,
         networkRequests: js-uint,
       },
       timestamp: js-uint,
     }
   }
   ```

2. **执行历史记录**:
   - 添加命令执行历史查询：
   ```cddl
   session.GetExecutionHistory = {
     method: "session.getExecutionHistory",
     params: {
       ? limit: js-uint,
       ? filter: ExecutionFilter,
     }
   }
   ```

3. **优化建议命令**:
   - 添加性能分析命令（但 AI 分析需要在客户端实现）：
   ```cddl
   session.AnalyzePerformance = {
     method: "session.analyzePerformance",
     params: {
       context: browsingContext.BrowsingContext,
       ? metrics: [+text],
     }
   }
   ```

**修改文件**:
- `index.bs`: 添加 `performance` 模块
- 客户端需要实现 AI 分析逻辑

**挑战与考虑**:
- AI 功能更适合在客户端库层面实现，而非协议层面
- 协议可以提供数据收集机制，但分析逻辑应在客户端
- 需要平衡协议复杂度和实际价值
- 性能监控可能影响执行效率

**建议**:
- 优先实现性能监控数据收集
- AI 优化建议作为客户端库的高级功能
- 协议层面提供必要的数据接口即可

---

### 2.6 性能与效率提升

#### 可行性评估: ⭐⭐⭐⭐ (可行)

**当前状态分析**:
- 协议支持异步命令执行
- WebSocket 传输支持全双工通信
- 缺乏性能监控机制

**实施建议**:

1. **性能监控模块**:
   - 创建 `performance` 模块，添加性能数据事件：
   ```cddl
   performance.PageLoadMetrics = {
     method: "performance.pageLoadMetrics",
     params: {
       context: browsingContext.BrowsingContext,
       navigationId: text,
       metrics: {
         domContentLoaded: js-uint,
         loadComplete: js-uint,
         firstContentfulPaint: js-uint,
         largestContentfulPaint: js-uint,
       },
     }
   }
   
   performance.ResourceUsage = {
     method: "performance.resourceUsage",
     params: {
       context: browsingContext.BrowsingContext,
       cpuTime: js-uint,
       memoryUsage: js-uint,
       networkBytes: js-uint,
       timestamp: js-uint,
     }
   }
   ```

2. **性能配置命令**:
   - 添加性能相关配置：
   ```cddl
   session.SetPerformanceMonitoring = {
     method: "session.setPerformanceMonitoring",
     params: {
       enabled: bool,
       ? metrics: [+text],              // 监控指标列表
       ? samplingInterval: js-uint,      // 采样间隔
     }
   }
   ```

3. **资源使用报告**:
   - 添加资源使用查询命令：
   ```cddl
   session.GetResourceUsage = {
     method: "session.getResourceUsage",
     params: {
       ? context: browsingContext.BrowsingContext,
       ? timeRange: TimeRange,
     }
   }
   
   TimeRange = {
     start: js-uint,
     end: js-uint,
   }
   ```

**修改文件**:
- `index.bs`: 添加 `performance` 模块定义
- `proposals/openrpc.json`: 更新性能相关命令和事件

**优势**:
- 性能监控是自动化测试的常见需求
- 实现相对直接，主要是数据收集和报告
- 可以逐步添加监控指标

**注意事项**:
- 性能监控本身可能带来性能开销
- 需要定义清晰的指标和测量方法
- 需要考虑不同浏览器的实现差异

---

## 三、改进方向总结与优先级建议

### 优先级排序

1. **高优先级（立即实施）**:
   - ✅ **增强事件与消息语义**: 高度可行，价值明确，实施成本低
   - ✅ **性能与效率提升**: 自动化测试的核心需求，实施可行

2. **中优先级（短期规划）**:
   - ⚠️ **协议命令的扩展与增强**: 需要仔细设计，但价值较高
   - ⚠️ **数据交换与消息格式扩展**: 利用现有机制，实施成本低

3. **低优先级（长期考虑）**:
   - ⚠️ **权限与安全控制**: 需要浏览器端深度支持，实施复杂
   - ❌ **AI 和自动化优化支持**: 更适合客户端实现，协议层面价值有限

### 实施建议

1. **渐进式实施**: 优先实施高优先级改进，逐步引入新功能
2. **向后兼容**: 所有改进都应保持与现有协议的兼容性
3. **模块化扩展**: 利用现有的模块化架构，在新模块中实现新功能
4. **标准化流程**: 遵循 W3C 标准化流程，确保改进的规范性和互操作性

### 具体实施步骤示例（以事件增强为例）

1. **设计阶段**:
   - 在 `index.bs` 中定义新事件类型（如 `browsingContext.domContentLoaded`）
   - 设计事件参数结构（CDDL 定义）
   - 定义事件触发条件（remote end event trigger）

2. **实现阶段**:
   - 更新 `proposals/openrpc.json` 添加事件定义
   - 实现 remote end subscribe steps（如需要）
   - 更新文档和示例

3. **测试阶段**:
   - 编写测试用例
   - 验证浏览器实现
   - 更新兼容性数据

4. **标准化阶段**:
   - 提交提案到 W3C
   - 收集实现反馈
   - 完善规范

---

## 四、结论

WebDriver BiDi 协议采用了良好的模块化设计和扩展机制，为后续改进提供了坚实的基础。通过分析，我们认为：

1. **事件增强**和**性能监控**是最优先的改进方向，实施可行且价值明确
2. **命令扩展**和**数据格式优化**可以逐步引入，提升协议能力
3. **权限控制**需要谨慎设计，考虑浏览器实现复杂度
4. **AI 优化**更适合在客户端库层面实现，协议层面提供数据接口即可

所有改进都应遵循协议的设计原则，保持向后兼容，并通过 W3C 标准化流程确保互操作性。