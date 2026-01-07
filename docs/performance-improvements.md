# 性能与效率提升改进 - 更新记录

## 概述

本文档记录了基于 README.md 中分析完成的"性能与效率提升"改进的实施详情。

**改进日期**: 2026-01-06  
**改进版本**: 1.0  
**状态**: 已完成

---

## 改进目标

根据 README.md 中的分析，本次改进旨在：

1. **性能监控模块**: 创建 `performance` 模块，添加性能数据事件，包括页面加载指标和资源使用情况
2. **性能配置命令**: 在 `session` 模块中添加性能监控配置命令，允许客户端启用/禁用性能监控并配置监控参数
3. **资源使用报告**: 添加资源使用查询命令，允许客户端获取历史资源使用数据

---

## 实施内容

### 1. 创建 performance 模块

#### 1.1 模块定义

在 `index.bs` 中创建了新的 `performance` 模块，用于性能监控相关的事件。

**位置**: `index.bs` 第 13426-13650 行

**模块结构**:
- 仅包含事件定义，不包含命令
- 定义了两个性能事件：`performance.pageLoadMetrics` 和 `performance.resourceUsage`

#### 1.2 performance.pageLoadMetrics 事件

**CDDL 定义**:
```cddl
performance.PageLoadMetrics = (
  method: "performance.pageLoadMetrics",
  params: performance.PageLoadMetricsParameters
)

performance.PageLoadMetricsParameters = {
  context: browsingContext.BrowsingContext,
  navigationId: text,
  metrics: performance.PageLoadMetricsData,
  timestamp: js-uint,
}

performance.PageLoadMetricsData = {
  domContentLoaded: js-uint,
  loadComplete: js-uint,
  firstContentfulPaint: js-uint,
  largestContentfulPaint: js-uint,
}
```

**事件参数说明**:
- **context**: 浏览上下文 ID
- **navigationId**: 关联的导航 ID
- **metrics**: 页面加载指标数据
  - **domContentLoaded**: 从导航开始到 DOMContentLoaded 事件的时间（毫秒）
  - **loadComplete**: 从导航开始到 load 事件的时间（毫秒）
  - **firstContentfulPaint**: 从导航开始到首次内容绘制的时间（毫秒）
  - **largestContentfulPaint**: 从导航开始到最大内容绘制的时间（毫秒）
- **timestamp**: 指标收集的时间戳（Unix 时间戳，毫秒）

**事件触发条件**:
- 当页面加载完成且性能监控已启用时触发
- 仅在已订阅该事件的会话中发送

**使用示例**:
```json
// 订阅 performance.pageLoadMetrics 事件
{
  "id": 1,
  "method": "session.subscribe",
  "params": {
    "events": ["performance.pageLoadMetrics"]
  }
}

// 接收到的事件示例
{
  "type": "event",
  "method": "performance.pageLoadMetrics",
  "params": {
    "context": "context-123",
    "navigationId": "nav-456",
    "metrics": {
      "domContentLoaded": 1250,
      "loadComplete": 2100,
      "firstContentfulPaint": 800,
      "largestContentfulPaint": 1800
    },
    "timestamp": 1704547200000
  }
}
```

#### 1.3 performance.resourceUsage 事件

**CDDL 定义**:
```cddl
performance.ResourceUsage = (
  method: "performance.resourceUsage",
  params: performance.ResourceUsageParameters
)

performance.ResourceUsageParameters = {
  context: browsingContext.BrowsingContext,
  cpuTime: js-uint,
  memoryUsage: js-uint,
  networkBytes: js-uint,
  timestamp: js-uint,
}
```

**事件参数说明**:
- **context**: 浏览上下文 ID
- **cpuTime**: CPU 使用时间（毫秒）
- **memoryUsage**: 内存使用量（字节）
- **networkBytes**: 网络传输字节数
- **timestamp**: 数据收集的时间戳（Unix 时间戳，毫秒）

**事件触发条件**:
- 当性能监控已启用时，按照配置的采样间隔定期触发
- 仅在已订阅该事件的会话中发送

**使用示例**:
```json
// 订阅 performance.resourceUsage 事件
{
  "id": 2,
  "method": "session.subscribe",
  "params": {
    "events": ["performance.resourceUsage"]
  }
}

// 接收到的事件示例
{
  "type": "event",
  "method": "performance.resourceUsage",
  "params": {
    "context": "context-123",
    "cpuTime": 150,
    "memoryUsage": 52428800,
    "networkBytes": 1048576,
    "timestamp": 1704547200000
  }
}
```

---

### 2. session.setPerformanceMonitoring 命令

#### 2.1 命令定义

在 `session` 模块中添加了 `session.setPerformanceMonitoring` 命令，用于启用或禁用性能监控。

**位置**: `index.bs` 第 1638-1644 行（命令列表）和第 2318-2365 行（命令实现）

**CDDL 定义**:
```cddl
session.SetPerformanceMonitoring = (
  method: "session.setPerformanceMonitoring",
  params: session.SetPerformanceMonitoringParameters
)

session.SetPerformanceMonitoringParameters = {
  enabled: bool,
  ? metrics: [+text],
  ? samplingInterval: js-uint,
}
```

**参数说明**:
- **enabled**: 是否启用性能监控（必需）
- **metrics**: 要监控的指标列表（可选）。如果未指定，则监控所有可用指标
  - 支持的指标：`"pageLoad"`, `"resourceUsage"`, `"memory"`, `"cpu"`, `"network"`
- **samplingInterval**: 采样间隔（可选，单位：毫秒）。默认值为 1000ms

**返回类型**:
```cddl
session.SetPerformanceMonitoringResult = EmptyResult
```

#### 2.2 命令算法

命令执行步骤：
1. 获取 `enabled`、`metrics` 和 `samplingInterval` 参数
2. 设置会话的性能监控启用标志
3. 如果启用，设置监控指标和采样间隔，并启动性能监控
4. 如果禁用，停止性能监控
5. 返回成功响应

#### 2.3 使用示例

```json
// 启用性能监控，监控所有指标，使用默认采样间隔（1000ms）
{
  "id": 3,
  "method": "session.setPerformanceMonitoring",
  "params": {
    "enabled": true
  }
}

// 启用性能监控，仅监控页面加载和资源使用，自定义采样间隔为 2000ms
{
  "id": 4,
  "method": "session.setPerformanceMonitoring",
  "params": {
    "enabled": true,
    "metrics": ["pageLoad", "resourceUsage"],
    "samplingInterval": 2000
  }
}

// 禁用性能监控
{
  "id": 5,
  "method": "session.setPerformanceMonitoring",
  "params": {
    "enabled": false
  }
}
```

---

### 3. session.getResourceUsage 命令

#### 3.1 命令定义

在 `session` 模块中添加了 `session.getResourceUsage` 命令，用于查询资源使用数据。

**位置**: `index.bs` 第 1638-1644 行（命令列表）和第 2367-2405 行（命令实现）

**CDDL 定义**:
```cddl
session.GetResourceUsage = (
  method: "session.getResourceUsage",
  params: session.GetResourceUsageParameters
)

session.GetResourceUsageParameters = {
  ? context: browsingContext.BrowsingContext,
  ? timeRange: session.TimeRange,
}

session.TimeRange = {
  start: js-uint,
  end: js-uint,
}
```

**参数说明**:
- **context**: 浏览上下文 ID（可选）。如果未指定，返回所有上下文的数据
- **timeRange**: 时间范围（可选）。如果未指定，返回所有可用数据
  - **start**: 开始时间戳（Unix 时间戳，毫秒）
  - **end**: 结束时间戳（Unix 时间戳，毫秒）

**返回类型**:
```cddl
session.GetResourceUsageResult = {
  resourceUsage: [+session.ResourceUsageEntry],
}

session.ResourceUsageEntry = {
  context: browsingContext.BrowsingContext,
  timestamp: js-uint,
  cpuTime: js-uint,
  memoryUsage: js-uint,
  networkBytes: js-uint,
}
```

#### 3.2 命令算法

命令执行步骤：
1. 获取 `context` 和 `timeRange` 参数
2. 如果指定了 `context`，获取该上下文的资源使用数据
3. 如果未指定 `context`，获取所有顶级可遍历上下文的资源使用数据
4. 根据 `timeRange` 过滤数据（如果提供）
5. 返回资源使用数据列表

#### 3.3 使用示例

```json
// 获取所有上下文的资源使用数据
{
  "id": 6,
  "method": "session.getResourceUsage",
  "params": {}
}

// 获取特定上下文的资源使用数据
{
  "id": 7,
  "method": "session.getResourceUsage",
  "params": {
    "context": "context-123"
  }
}

// 获取特定时间范围的资源使用数据
{
  "id": 8,
  "method": "session.getResourceUsage",
  "params": {
    "timeRange": {
      "start": 1704547000000,
      "end": 1704547200000
    }
  }
}

// 响应示例
{
  "type": "success",
  "id": 8,
  "result": {
    "resourceUsage": [
      {
        "context": "context-123",
        "timestamp": 1704547100000,
        "cpuTime": 150,
        "memoryUsage": 52428800,
        "networkBytes": 1048576
      },
      {
        "context": "context-123",
        "timestamp": 1704547150000,
        "cpuTime": 180,
        "memoryUsage": 53687091,
        "networkBytes": 2097152
      }
    ]
  }
}
```

---

## 修改的文件清单

### 1. index.bs

**修改位置**:
- 第 1638-1644 行: 在 `SessionCommand` 中添加 `session.SetPerformanceMonitoring` 和 `session.GetResourceUsage`
- 第 1650-1656 行: 在 `SessionResult` 中添加对应的结果类型
- 第 1901-1950 行: 添加性能监控相关的类型定义
- 第 2318-2405 行: 添加性能监控命令的实现算法
- 第 523-529 行: 在 `EventData` 中添加 `PerformanceEvent`
- 第 13426-13650 行: 创建 `performance` 模块，包含事件定义和触发算法

**修改类型**: 
- 新增模块定义
- 新增命令类型定义
- 新增事件类型定义
- 新增命令实现算法
- 新增事件触发算法

### 2. proposals/openrpc.json

**修改位置**:
- 第 80-109 行: 添加 `session.setPerformanceMonitoring` 命令定义
- 第 110-130 行: 添加 `session.getResourceUsage` 命令定义
- 第 1620-1651 行: 添加 `performance.pageLoadMetrics` 事件定义
- 第 1653-1690 行: 添加 `performance.resourceUsage` 事件定义
- 第 1906-1920 行: 添加 `TimeRange` 类型定义
- 第 1921-1946 行: 添加 `ResourceUsageEntry` 类型定义
- 第 1948-1958 行: 添加 `GetResourceUsageResult` 类型定义
- 第 1960-1981 行: 添加 `PageLoadMetricsData` 类型定义

**修改类型**: 
- 新增命令定义（methods 数组）
- 新增事件定义（methods 数组）
- 新增类型定义（components.schemas）

---

## 实施注意事项

### 1. 向后兼容性

- ✅ `performance` 模块是新模块，不影响现有功能
- ✅ `session.setPerformanceMonitoring` 和 `session.getResourceUsage` 是新命令，不影响现有命令
- ✅ 性能事件仅在订阅时发送，不影响未订阅的会话
- ✅ 所有新增字段都使用可选标记（`?`），确保向后兼容

### 2. 实现考虑

#### 性能监控

- **性能开销**: 性能监控本身可能带来性能开销，需要平衡监控频率和开销
- **采样间隔**: 默认采样间隔为 1000ms，可根据需要调整
- **指标收集**: 需要浏览器端支持性能 API（如 Performance API、Resource Timing API）

#### 资源使用数据

- **数据存储**: 需要在浏览器端存储历史资源使用数据
- **数据清理**: 需要考虑数据清理策略，避免内存泄漏
- **时间范围**: 支持按时间范围查询，但需要限制查询范围以避免性能问题

### 3. 浏览器实现要求

- 需要浏览器端支持 Performance API
- 需要能够收集 CPU 时间、内存使用和网络字节数
- 需要支持页面加载指标（DOMContentLoaded、load、FCP、LCP）
- 需要实现性能监控的启动和停止机制

---

## 测试建议

### 1. session.setPerformanceMonitoring 命令测试

```javascript
// 测试用例 1: 启用性能监控
await session.setPerformanceMonitoring({ enabled: true });
// 应该成功返回

// 测试用例 2: 启用性能监控并指定指标
await session.setPerformanceMonitoring({
  enabled: true,
  metrics: ["pageLoad", "resourceUsage"],
  samplingInterval: 2000
});
// 应该成功返回，并且只监控指定的指标

// 测试用例 3: 禁用性能监控
await session.setPerformanceMonitoring({ enabled: false });
// 应该成功返回，并且停止发送性能事件
```

### 2. session.getResourceUsage 命令测试

```javascript
// 测试用例 1: 获取所有上下文的资源使用数据
const result = await session.getResourceUsage({});
// 应该返回所有上下文的资源使用数据

// 测试用例 2: 获取特定上下文的资源使用数据
const result = await session.getResourceUsage({
  context: "context-123"
});
// 应该返回指定上下文的资源使用数据

// 测试用例 3: 获取特定时间范围的数据
const result = await session.getResourceUsage({
  timeRange: {
    start: Date.now() - 60000,
    end: Date.now()
  }
});
// 应该返回指定时间范围内的数据
```

### 3. performance.pageLoadMetrics 事件测试

```javascript
// 测试用例 1: 订阅页面加载指标事件
await session.subscribe({ events: ["performance.pageLoadMetrics"] });
await session.setPerformanceMonitoring({ enabled: true });
await browsingContext.navigate({ url: "https://example.com" });
// 应该接收到 performance.pageLoadMetrics 事件

// 测试用例 2: 验证指标数据
// 接收到的事件应该包含 domContentLoaded、loadComplete、firstContentfulPaint、largestContentfulPaint
```

### 4. performance.resourceUsage 事件测试

```javascript
// 测试用例 1: 订阅资源使用事件
await session.subscribe({ events: ["performance.resourceUsage"] });
await session.setPerformanceMonitoring({
  enabled: true,
  samplingInterval: 1000
});
// 应该每隔 1 秒接收到 performance.resourceUsage 事件

// 测试用例 2: 验证资源使用数据
// 接收到的事件应该包含 cpuTime、memoryUsage、networkBytes
```

---

## 后续工作

### 短期（1-2 周）

1. ✅ 完成 `index.bs` 中的模块和命令定义
2. ✅ 更新 `proposals/openrpc.json` 以包含新命令和事件
3. ⏳ 编写测试用例
4. ⏳ 更新文档和示例

### 中期（1-2 月）

1. ⏳ 浏览器实现验证
2. ⏳ 性能测试和优化
3. ⏳ 收集实现反馈
4. ⏳ 完善性能监控逻辑

### 长期（3-6 月）

1. ⏳ W3C 标准化流程
2. ⏳ 多浏览器实现支持
3. ⏳ 更新兼容性数据
4. ⏳ 社区反馈和迭代

---

## 相关资源

- [README.md](../README.md) - 协议结构分析和改进方向
- [index.bs](../index.bs) - 主规范文件
- [proposals/openrpc.json](../proposals/openrpc.json) - OpenRPC API 描述

---

## 变更历史

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|----------|------|
| 2026-01-06 | 1.0 | 初始版本：创建 performance 模块，添加性能监控命令和事件 | - |

---

## 反馈与问题

如有问题或建议，请通过以下方式反馈：

- GitHub Issues: [w3c/webdriver-bidi](https://github.com/w3c/webdriver-bidi)
- W3C 邮件列表: public-browser-tools-testing@w3.org

