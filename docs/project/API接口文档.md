# Claude-Mem API 接口文档

> 版本：v9.1.1 | 更新日期：2026-02-08

---

## 1. 概述

| 属性 | 值 |
|------|-----|
| 基础 URL | `http://127.0.0.1:37777` |
| 协议 | HTTP/1.1 |
| 数据格式 | JSON |
| 认证 | 无（本地信任模型） |
| CORS | 允许所有源 |

---

## 2. 搜索类接口

### 2.1 搜索观察

```
GET /api/search
```

**描述**：搜索历史观察记录，支持语义搜索和过滤。

**查询参数**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `query` | string | 否 | | 搜索关键词（语义搜索） |
| `project` | string | 否 | | 项目名称过滤 |
| `limit` | number | 否 | 20 | 返回数量上限 |
| `offset` | number | 否 | 0 | 分页偏移量 |
| `type` | string | 否 | | 搜索类型：observation / summary / prompt |
| `obs_type` | string | 否 | | 观察类型：discovery / bugfix / feature / decision / change |
| `dateStart` | string | 否 | | 起始日期（ISO 8601） |
| `dateEnd` | string | 否 | | 结束日期（ISO 8601） |

**响应示例**：

```json
{
  "results": [
    {
      "id": 42,
      "title": "修复 JWT 认证过期问题",
      "type": "bugfix",
      "project": "my-app",
      "timestamp": 1707350400,
      "score": 0.95
    }
  ],
  "total": 156,
  "hasMore": true
}
```

### 2.2 构建时间线

```
GET /api/timeline
```

**描述**：以锚点为中心构建时间线上下文。

**查询参数**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `anchor` | number | 是* | | 锚点观察 ID |
| `query` | string | 是* | | 或通过查询定位锚点 |
| `depth_before` | number | 否 | 3 | 锚点前的条目数 |
| `depth_after` | number | 否 | 3 | 锚点后的条目数 |
| `project` | string | 否 | | 项目过滤 |

*`anchor` 和 `query` 至少提供一个。

**响应示例**：

```json
{
  "timeline": [
    {
      "id": 39,
      "title": "读取配置文件",
      "type": "observation",
      "timestamp": 1707340000,
      "isAnchor": false
    },
    {
      "id": 42,
      "title": "修复 JWT 认证过期问题",
      "type": "observation",
      "timestamp": 1707350400,
      "isAnchor": true
    }
  ],
  "anchor": 42
}
```

### 2.3 批量获取观察

```
GET /api/observations
```

**描述**：按 ID 批量获取观察详情。

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `ids` | string | 是 | 逗号分隔的 ID 列表（如 "42,38,35"） |
| `orderBy` | string | 否 | 排序方式：created_at（默认） |
| `project` | string | 否 | 项目过滤 |

**响应示例**：

```json
{
  "observations": [
    {
      "id": 42,
      "title": "修复 JWT 认证过期问题",
      "narrative": "发现 JWT token 的过期时间配置为 1 小时，但刷新 token 逻辑...",
      "facts": [
        "JWT 过期时间配置为 24 小时",
        "刷新 token 有效期为 7 天"
      ],
      "concepts": ["认证", "JWT", "安全"],
      "files_read": ["src/auth/jwt.ts", "src/config.ts"],
      "files_modified": ["src/auth/jwt.ts"],
      "type": "bugfix",
      "project": "my-app",
      "prompt_number": 3,
      "created_at_epoch": 1707350400
    }
  ]
}
```

---

## 3. 会话管理接口

### 3.1 初始化会话

```
POST /api/session/init
```

**描述**：创建新的会话记录。

**请求体**：

```json
{
  "content_session_id": "abc-123-def",
  "project": "my-app",
  "user_prompt": "帮我修复登录 Bug",
  "worker_port": 37777
}
```

**响应**：

```json
{
  "success": true,
  "session_db_id": 156,
  "prompt_number": 1
}
```

### 3.2 记录观察

```
POST /api/session/observation
```

**描述**：提交工具调用的观察数据进行异步处理。

**请求体**：

```json
{
  "content_session_id": "abc-123-def",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  },
  "tool_output": "Tests: 42 passed, 1 failed"
}
```

**响应**：

```json
{
  "success": true,
  "pending_message_id": 789
}
```

### 3.3 生成摘要

```
POST /api/session/summarize
```

**描述**：触发会话摘要的 AI 生成。

**请求体**：

```json
{
  "content_session_id": "abc-123-def"
}
```

**响应**：

```json
{
  "success": true,
  "summary_id": 45
}
```

### 3.4 完成会话

```
POST /api/session/complete
```

**描述**：标记会话为已完成。

**请求体**：

```json
{
  "content_session_id": "abc-123-def"
}
```

**响应**：

```json
{
  "success": true,
  "completed_at": 1707360000
}
```

### 3.5 查询会话状态

```
GET /api/session/status?content_session_id=abc-123-def
```

**响应**：

```json
{
  "session": {
    "id": 156,
    "content_session_id": "abc-123-def",
    "memory_session_id": "sdk-456-ghi",
    "project": "my-app",
    "status": "active",
    "prompt_counter": 5,
    "started_at_epoch": 1707340000,
    "completed_at_epoch": null
  }
}
```

---

## 4. 设置接口

### 4.1 获取设置

```
GET /api/settings
```

**响应**：

```json
{
  "settings": {
    "CLAUDE_MEM_PROVIDER": "claude",
    "CLAUDE_MEM_WORKER_PORT": "37777",
    "CLAUDE_MEM_WORKER_HOST": "127.0.0.1",
    "CLAUDE_MEM_DATA_DIR": "~/.claude-mem",
    "CLAUDE_MEM_CONTEXT_OBSERVATIONS": "50",
    "CLAUDE_MEM_CONTEXT_FULL_COUNT": "5",
    "CLAUDE_MEM_CONTEXT_SESSION_COUNT": "10",
    "CLAUDE_MEM_GEMINI_API_KEY": "",
    "CLAUDE_MEM_GEMINI_MODEL": "gemini-2.5-flash-lite",
    "CLAUDE_MEM_OPENROUTER_MODEL": "xiaomi/mimo-v2-flash:free"
  }
}
```

### 4.2 更新设置

```
POST /api/settings
```

**请求体**：

```json
{
  "CLAUDE_MEM_PROVIDER": "gemini",
  "CLAUDE_MEM_GEMINI_API_KEY": "your-api-key"
}
```

**响应**：

```json
{
  "success": true,
  "updated": ["CLAUDE_MEM_PROVIDER", "CLAUDE_MEM_GEMINI_API_KEY"]
}
```

---

## 5. 记忆接口

### 5.1 手动保存记忆

```
POST /api/memory/save
```

**描述**：用户手动保存重要信息到记忆系统。

**请求体**：

```json
{
  "text": "数据库连接池最大连接数设置为 50 性能最优",
  "title": "数据库连接池配置经验",
  "project": "my-app"
}
```

**响应**：

```json
{
  "success": true,
  "observation_id": 999
}
```

---

## 6. 数据导入导出接口

### 6.1 导出数据

```
GET /api/data/export
```

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `project` | string | 否 | 按项目过滤导出 |

**响应**：

```json
{
  "version": "9.1.1",
  "exported_at": "2026-02-08T10:00:00Z",
  "data": {
    "sessions": [...],
    "observations": [...],
    "summaries": [...],
    "user_prompts": [...]
  }
}
```

### 6.2 导入数据

```
POST /api/data/import
```

**请求体**：与导出格式相同的 JSON 对象。

**响应**：

```json
{
  "success": true,
  "imported": {
    "sessions": 10,
    "observations": 156,
    "summaries": 10,
    "user_prompts": 45
  }
}
```

---

## 7. 日志接口

### 7.1 获取日志

```
GET /api/logs
```

**查询参数**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `level` | string | 否 | all | 日志级别：debug / info / warn / error |
| `limit` | number | 否 | 100 | 返回条目数 |

**响应**：

```json
{
  "logs": [
    {
      "timestamp": "2026-02-08T10:00:00Z",
      "level": "info",
      "message": "Worker started on port 37777",
      "context": {}
    }
  ]
}
```

---

## 8. Viewer UI 接口

### 8.1 获取 UI 页面

```
GET /
```

**响应**：HTML 页面（Viewer UI SPA）。

### 8.2 SSE 事件流

```
GET /stream
```

**描述**：Server-Sent Events 实时推送。

**事件类型**：

| 事件名 | 数据 | 说明 |
|--------|------|------|
| `initial_load` | 最近 50 条数据的数组 | 首次连接时发送 |
| `new_observation` | 单个观察对象 | 新观察入库时 |
| `new_summary` | 单个摘要对象 | 新摘要入库时 |
| `new_prompt` | 单个提示对象 | 新用户提示入库时 |
| `processing_status` | `{ pending, processing }` | 队列状态变化时 |

**SSE 格式**：

```
event: new_observation
data: {"id":42,"title":"修复JWT认证","type":"bugfix",...}

event: processing_status
data: {"pending":3,"processing":1}
```

---

## 9. 健康检查接口

### 9.1 健康检查

```
GET /health
```

**响应**：

```json
{
  "status": "ok",
  "version": "9.1.1",
  "uptime": 3600,
  "database": "connected",
  "chroma": "connected"
}
```

---

## 10. MCP 工具接口

通过 MCP 协议暴露的工具（由 `plugin/scripts/mcp-server.cjs` 提供）：

### 10.1 search

```json
{
  "name": "search",
  "description": "搜索历史记忆",
  "parameters": {
    "query": "string - 搜索关键词",
    "limit": "number - 返回数量（默认 20）",
    "project": "string - 项目过滤",
    "type": "string - 类型过滤",
    "obs_type": "string - 观察类型过滤",
    "dateStart": "string - 起始日期",
    "dateEnd": "string - 结束日期"
  }
}
```

### 10.2 timeline

```json
{
  "name": "timeline",
  "description": "构建时间线上下文",
  "parameters": {
    "anchor": "number - 锚点 ID",
    "query": "string - 锚点查询",
    "depth_before": "number - 前置深度（默认 3）",
    "depth_after": "number - 后置深度（默认 3）",
    "project": "string - 项目过滤"
  }
}
```

### 10.3 get_observations

```json
{
  "name": "get_observations",
  "description": "批量获取观察详情",
  "parameters": {
    "ids": "number[] - 观察 ID 列表",
    "orderBy": "string - 排序字段",
    "limit": "number - 数量限制",
    "project": "string - 项目过滤"
  }
}
```

### 10.4 save_memory

```json
{
  "name": "save_memory",
  "description": "手动保存记忆",
  "parameters": {
    "text": "string - 记忆内容（必填）",
    "title": "string - 标题",
    "project": "string - 项目名称"
  }
}
```

---

## 11. 错误响应格式

所有接口的错误响应遵循统一格式：

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Session not found",
    "details": {}
  }
}
```

**常见错误码**：

| HTTP 状态码 | 错误码 | 说明 |
|------------|--------|------|
| 400 | `INVALID_REQUEST` | 请求参数无效 |
| 404 | `NOT_FOUND` | 资源不存在 |
| 500 | `INTERNAL_ERROR` | 服务器内部错误 |
| 503 | `SERVICE_UNAVAILABLE` | Worker 未就绪（初始化中） |
