# 增强事件与消息语义改进 - 更新记录

## 概述

本文档记录了基于 README.md 中分析完成的"增强事件与消息语义"改进的实施详情。

**改进日期**: 2026-01-06  
**改进版本**: 1.0  
**状态**: 实施中

---

## 改进目标

根据 README.md 中的分析，本次改进旨在：

1. **扩展事件类型**: 添加更多细粒度的事件类型，特别是 JavaScript 运行时错误事件
2. **增强事件过滤**: 支持动态事件过滤，允许客户端根据 URL、事件类型、元素选择器等条件过滤事件
3. **提升调试能力**: 通过详细的错误堆栈信息，帮助开发者更快定位问题

---

## 实施内容

### 1. 添加 script.runtimeError 事件

#### 1.1 事件定义

在 `index.bs` 的 `script` 模块中添加了新事件 `script.runtimeError`，用于捕获 JavaScript 运行时错误。

**位置**: `index.bs` 第 10552-10556 行（ScriptEvent 定义）和第 13323-13400 行（事件详细定义）

**CDDL 定义**:
```cddl
ScriptEvent = (
  script.Message //
  script.RealmCreated //
  script.RealmDestroyed //
  script.RuntimeError  // 新增
)

script.RuntimeError = (
  method: "script.runtimeError",
  params: script.RuntimeErrorParameters
)

script.RuntimeErrorParameters = {
  realm: script.Realm,
  error: script.RemoteValue,
  stackTrace: [+script.StackTraceFrame],
  timestamp: js-uint,
  source: script.Source,
}

script.StackTraceFrame = {
  functionName: text,
  ? lineNumber: js-uint,
  ? columnNumber: js-uint,
  ? url: text,
}
```

#### 1.2 事件参数说明

- **realm**: 发生错误的脚本领域 ID
- **error**: 序列化的错误对象（RemoteValue 格式）
- **stackTrace**: 堆栈跟踪信息数组，包含函数名、行号、列号和源 URL
- **timestamp**: 错误发生的时间戳（Unix 时间戳，毫秒）
- **source**: 错误发生的源信息（包含 realm 和可选的 browsingContext）

#### 1.3 事件触发条件

事件在以下情况下触发：
- 脚本执行过程中发生未捕获的异常
- 错误发生在已订阅该事件的会话的浏览上下文中

#### 1.4 使用示例

```json
// 订阅 script.runtimeError 事件
{
  "id": 1,
  "method": "session.subscribe",
  "params": {
    "events": ["script.runtimeError"]
  }
}

// 接收到的事件示例
{
  "type": "event",
  "method": "script.runtimeError",
  "params": {
    "realm": "realm-123",
    "error": {
      "type": "error",
      "value": {
        "name": "TypeError",
        "message": "Cannot read property 'x' of undefined"
      }
    },
    "stackTrace": [
      {
        "functionName": "processData",
        "lineNumber": 42,
        "columnNumber": 15,
        "url": "https://example.com/app.js"
      },
      {
        "functionName": "onClick",
        "lineNumber": 10,
        "columnNumber": 5,
        "url": "https://example.com/app.js"
      }
    ],
    "timestamp": 1704547200000,
    "source": {
      "realm": "realm-123",
      "context": "context-456"
    }
  }
}
```

---

### 2. 扩展事件订阅过滤功能

#### 2.1 订阅参数扩展

在 `session.SubscribeParameters` 类型中添加了可选的 `filters` 字段，支持细粒度的事件过滤。

**位置**: `index.bs` 第 1872-1895 行

**CDDL 定义**:
```cddl
session.SubscribeParameters = {
  events: [+text],
  ? contexts: [+browsingContext.BrowsingContext],
  ? userContexts: [+browser.UserContext],
  ? filters: session.EventFilters,  // 新增
}

session.EventFilters = {
  ? urlPattern: text,        // URL 模式匹配（支持通配符）
  ? eventType: text,         // 事件类型过滤
  ? elementSelector: text,   // 元素选择器过滤（CSS 选择器）
}
```

#### 2.2 过滤条件说明

- **urlPattern**: URL 模式匹配，支持通配符（如 `https://example.com/*`）
- **eventType**: 特定的事件类型过滤（如 `"click"`, `"error"`）
- **elementSelector**: CSS 选择器，用于过滤特定元素的事件

#### 2.3 使用示例

```json
// 订阅特定 URL 模式的事件
{
  "id": 2,
  "method": "session.subscribe",
  "params": {
    "events": ["browsingContext.load"],
    "filters": {
      "urlPattern": "https://example.com/*"
    }
  }
}

// 订阅特定元素的事件
{
  "id": 3,
  "method": "session.subscribe",
  "params": {
    "events": ["input.click"],
    "filters": {
      "elementSelector": "#submit-button"
    }
  }
}

// 组合多个过滤条件
{
  "id": 4,
  "method": "session.subscribe",
  "params": {
    "events": ["script.runtimeError"],
    "filters": {
      "urlPattern": "https://example.com/app/*",
      "eventType": "error"
    }
  }
}
```

---

## 修改的文件清单

### 1. index.bs

**修改位置**:
- 第 10552-10556 行: 在 `ScriptEvent` 中添加 `script.RuntimeError`
- 第 1872-1895 行: 扩展 `session.SubscribeParameters` 添加 `filters` 字段
- 第 13323-13400 行: 添加 `script.runtimeError` 事件的完整定义和触发算法

**修改类型**: 
- 新增事件类型定义
- 扩展现有类型定义
- 新增事件触发算法

### 2. proposals/openrpc.json

**状态**: 待更新

**建议更新内容**:
- 在 `methods` 数组中添加 `script.runtimeError` 事件定义
- 更新 `session.subscribe` 方法的参数定义，添加 `filters` 字段
- 在 `components/schemas` 中添加 `EventFilters` 和 `StackTraceFrame` 类型定义

---

## 实施注意事项

### 1. 向后兼容性

- ✅ `script.runtimeError` 是新事件，不影响现有功能
- ✅ `filters` 字段是可选的，现有客户端代码无需修改
- ✅ 所有新增字段都使用可选标记（`?`），确保向后兼容

### 2. 实现考虑

#### script.runtimeError 事件

- **错误捕获**: 需要在浏览器端实现全局错误处理器，捕获未捕获的异常
- **堆栈解析**: 需要解析 JavaScript 错误对象的 `stack` 属性，提取堆栈帧信息
- **性能影响**: 错误捕获和序列化可能带来轻微性能开销，但影响可控

#### 事件过滤

- **过滤逻辑**: 过滤逻辑需要在事件触发时进行，可能需要额外的计算开销
- **URL 模式匹配**: 需要实现 URL 模式匹配算法（支持通配符）
- **元素选择器**: 需要支持 CSS 选择器匹配，可能需要 DOM 查询

### 3. 浏览器实现要求

- 需要浏览器端支持全局错误监听
- 需要能够访问错误对象的堆栈信息
- 需要支持 URL 模式匹配和 CSS 选择器查询

---

## 测试建议

### 1. script.runtimeError 事件测试

```javascript
// 测试用例 1: 基本错误捕获
await session.subscribe({ events: ["script.runtimeError"] });
await script.evaluate({ script: "throw new Error('Test error')" });
// 应该接收到 script.runtimeError 事件

// 测试用例 2: 堆栈跟踪
await script.evaluate({
  script: `
    function a() { throw new Error('Error in a'); }
    function b() { a(); }
    b();
  `
});
// 应该接收到包含完整堆栈跟踪的事件

// 测试用例 3: 多个 realm 的错误
// 在主页面和 iframe 中分别触发错误
// 应该分别接收到对应 realm 的错误事件
```

### 2. 事件过滤测试

```javascript
// 测试用例 1: URL 模式过滤
await session.subscribe({
  events: ["browsingContext.load"],
  filters: { urlPattern: "https://example.com/*" }
});
await browsingContext.navigate({ url: "https://example.com/page" });
// 应该接收到事件

await browsingContext.navigate({ url: "https://other.com/page" });
// 不应该接收到事件

// 测试用例 2: 元素选择器过滤
await session.subscribe({
  events: ["input.click"],
  filters: { elementSelector: "#test-button" }
});
// 点击 #test-button 应该接收到事件
// 点击其他元素不应该接收到事件
```

---

## 后续工作

### 短期（1-2 周）

1. ✅ 完成 `index.bs` 中的事件定义
2. ✅ 更新 `proposals/openrpc.json` 以包含新事件和参数
3. ⏳ 编写测试用例
4. ⏳ 更新文档和示例

### 中期（1-2 月）

1. ⏳ 浏览器实现验证
2. ⏳ 性能测试和优化
3. ⏳ 收集实现反馈
4. ⏳ 完善错误处理逻辑

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
| 2026-01-06 | 1.0 | 初始版本：添加 script.runtimeError 事件和事件过滤功能 | - |

---

## 反馈与问题

如有问题或建议，请通过以下方式反馈：

- GitHub Issues: [w3c/webdriver-bidi](https://github.com/w3c/webdriver-bidi)
- W3C 邮件列表: public-browser-tools-testing@w3.org

