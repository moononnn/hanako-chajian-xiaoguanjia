# 插件 API 参考
> Hana 插件开发的核心 API 完整字段说明
> 对应：plugin-scaffold-walkthrough.md + plugin-patterns.md + plugin-best-practices.md
> 最后更新：2026-06-27（新增 capabilities model/session 声明说明）

## 我从哪里开始查

| 你的角色 | 先看这里 |
|---------|---------|
| 写 manifest.json | [五、manifest.json Schema 参考](#五manifestjson-schema-参考) |
| 写工具函数 | [三、工具上下文 toolCtx](#三工具上下文-toolctx--ctx) |
| 写后端路由 | [一、路由上下文 ctx](#一路由上下文-ctx) + [二、路由处理器 c](#二路由处理器-c) |
| 写前端页面 | [十、Plugin SDK 宿主通信](#十plugin-sdk-宿主通信03453) |
| 调 LLM | [四、ctx.model.sample()](#四ctxmodelsample--调用宿主-llm) |
| 操作文件 | [八、ResourceIO 统一资源访问](#八resourceio-统一资源访问034119) |
| 调外部 API | [十一、HTTP 客户端](#十一http-客户端node-环境) |
| 查字段 | [一～十四全文索引](#) |

---

## 一、路由上下文 `ctx`

`ctx` 是服务端路由（`routes/ui.js`）的顶层上下文，由框架注入。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `ctx.pluginId` | `string` | 插件 ID，如 `"my-first-plugin"` |
| `ctx.dataDir` | `string` | 插件数据根目录（绝对路径），用于持久化 |
| `ctx.pluginDir` | `string` | 插件源码目录（绝对路径），用于读取 assets |
| `ctx.log?.info?.()` | `function` | 结构化日志（info 级别） |
| `ctx.log?.error?.()` | `function` | 结构化日志（error 级别） |
| `ctx.model?.sample()` | `function` | 调用宿主 LLM（仅 full-access） |
| `ctx.resources` | `object` | ResourceIO 统一资源访问层（0.341.19+） |

### 示例

```js
export default function (app, ctx) {
  const pid = ctx.pluginId;
  const dd = ctx.dataDir;

  ctx.log?.info?.("plugin loaded", { pluginId: pid });

  app.get("/api/hello", (c) => {
    return c.json({ ok: true, pluginId: pid });
  });
}
```

---

## 二、路由处理器 `c`（Koa 风格）

`c` 是每个路由回调的第一个参数，提供请求/响应操作。

### 请求读取

| 方法 | 说明 | 示例 |
|------|------|------|
| `c.req.query("key")` | 获取 URL 查询参数（函数，不是对象） | `c.req.query("token")` |
| `c.req.query("key", "default")` | 带默认值的查询参数 | `c.req.query("bookId", "")` |
| `c.req.path` | 请求路径 | `"/api/plugins/xxx/page"` |
| `c.req.json()` | 解析 JSON body（POST/PUT） | `const body = await c.req.json()` |
| `c.req.body` | 原始 body | - |

### 响应写入

| 方法 | 说明 | 示例 |
|------|------|------|
| `c.json(obj)` | 返回 JSON | `return c.json({ ok: true })` |
| `c.json(obj, status)` | 返回 JSON + 状态码 | `return c.json({ ok: false }, 400)` |
| `c.html(content)` | 返回 HTML | `return c.html(htmlString)` |
| `c.html(content, status)` | 返回 HTML + 状态码 | `return c.html(htmlString, 200)` |
| `c.text(content)` | 返回纯文本 | `return c.text("Not found", 404)` |
| `c.body(content)` | 返回原始 body | `return c.body(buffer)` |
| `c.header(key, value)` | 设置响应头 | `c.header("Content-Type", "text/plain")` |

### 示例

```js
app.get("/api/example", (c) => {
  const q = c.req.query("q") || "";
  const page = Number(c.req.query("page") || 1);

  if (!q) {
    return c.json({ ok: false, code: "missing_query" }, 400);
  }

  return c.json({ ok: true, query: q, page });
});

app.post("/api/update", async (c) => {
  const body = await c.req.json().catch(() => ({}));
  const name = body.name || "";
  return c.json({ ok: true, name });
});
```

---

## 三、工具上下文 `toolCtx` / `ctx`

工具函数接收两个参数：`input`（用户输入）和 `ctx`（宿主上下文）。

### `input` 参数

| 字段 | 类型 | 说明 |
|------|------|------|
| `input` | `object` | 用户传入的参数对象，结构由 `parameters` 定义 |

### `ctx` 参数

和路由上下文 `ctx` 是同一个对象，包含以下可用字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `ctx.pluginId` | `string` | 插件 ID |
| `ctx.dataDir` | `string` | 数据目录（持久化根路径） |
| `ctx.log?.info?.()` | `function` | 日志：`ctx.log.info("msg", { key: value })` |
| `ctx.log?.error?.()` | `function` | 错误日志：`ctx.log.error("msg", { error: err.message })` |
| `ctx.model?.sample()` | `function` | 调用 LLM（仅 full-access，见下文） |

### 工具函数签名

**方式1：推荐（带 name/description/parameters）**
```js
export const name = "tool_name";
export const description = "工具描述";
export const parameters = {
  type: "object",
  properties: {
    param1: { type: "string", description: "参数说明" }
  },
  required: ["param1"]
};

export async function execute(input, toolCtx) {
  // input.param1 就是用户传入的值
  return "结果";
}
```

**方式2：简化（默认导出）**
```js
export default async function tool_name(input, ctx) {
  return { ok: true, data: "结果" };
}
```

---

## 四、`ctx.model.sample()` — 调用宿主 LLM

**权限要求**：full-access
**来源**：legado-companion/tools/_lib/llm.js

**注意**：0.333.0+ 起，媒体适配器调用时 caller context 自动保留（session / pluginId），无需手动传递。

```js
const response = await ctx.model.sample({
  messages: [
    { role: "system", content: "你是一个有用的助手。" },
    { role: "user", content: "用户问题" },
  ],
  maxTokens: 2000,        // 可选，默认值取决于宿主配置
  temperature: 0.7,       // 可选
  topP: 1,                // 可选
  stop: ["\n\n\n"],       // 可选，停止序列
});

// 返回值字段
response.text       // 主文本（如果模型返回的是 text 格式）
response.content    // 备用字段（部分模型用 content）
response.usage      // 可选，token 使用量 { promptTokens, completionTokens, totalTokens }
```

### 返回结构参考

```js
{
  text: "模型生成的文本",
  content: "",           // 备用
  usage: {               // 可选
    promptTokens: 100,
    completionTokens: 50,
    totalTokens: 150
  },
  model: "模型ID",       // 可选
  finishReason: "stop"   // 可选
}
```

---

## 五、`manifest.json` Schema 参考

### 完整字段

```json
{
  "manifestVersion": 1,           // 必需，当前版本为 1
  "id": "my-plugin",              // 必需，小写、连字符、无空格
  "name": "My Plugin",            // 必需，显示名
  "version": "0.1.0",             // 必需，语义化版本
  "description": "...",           // 必需
  "trust": "full-access",         // 必需，"restricted" 或 "full-access"
  "contributes": {                // 可选，声明页面/挂件/工具
    "page": {                     // 可选，iframe 页面
      "title": "Page Title",
      "route": "/page",           // URL 路径
      "icon": "<svg>...</svg>"    // 内联 SVG
    },
    "widget": {                   // 可选，挂件
      "title": "Widget Title",
      "id": "my-widget",
      "icon": "<svg>...</svg>"
    }
  },
  "ui": {                         // 可选（0.345.3+），声明 iframe 宿主能力
    "hostCapabilities": [         // 需要授权的 iframe 宿主能力列表
      "external.open",
      "clipboard.writeText",
      "resource.open"
    ]
  },
  "network": {                    // 可选，声明外部 HTTP 数据访问边界
    "allowedHosts": ["api.example.com"],
    "methods": ["GET", "POST"]
  },
  "interface": {                  // 可选， richer metadata
    "displayName": "...",
    "shortDescription": "...",
    "category": "Productivity",   // Productivity / Dev / Media / Games / Social / Other
    "icon": "data:image/svg+xml,...", // 可选，base64 SVG
    "screenshots": [],            // 可选
    "tags": [],                   // 可选
    "author": "...",              // 可选
    "homepage": "...",            // 可选
    "license": "MIT"              // 可选
  }
}
```

### capabilities 声明（0.341.19+）

插件需要访问用户资源时，在 manifest.json 中声明能力：

```json
{
  "capabilities": [
    "resource.read",
    "resource.search",
    "resource.write"
  ]
}
```

| 能力 | 说明 |
|------|------|
| `resource.read` | 读取资源（stat/read/list） |
| `resource.search` | 搜索资源 |
| `resource.write` | 写入资源（write/edit/mkdir/delete/copy/rename/move/trash） |

### model / session capability（0.341.19+）

插件需要调用宿主 LLM 或通过路由访问会话 API 时，在 manifest.json 中声明对应能力。

```json
{
  "trust": "full-access",
  "capabilities": ["model", "session"],
  ...
}
```

| 能力 | 说明 |
|------|------|
| `model` | 声明后，路由 ctx（`routes/ui.js` 的 ctx）会注入 `ctx.model.sample()`，可用于调用宿主 LLM。不声明时路由 ctx 没有 `ctx.model`，只能通过 Agent 工具链调用 LLM |
| `session` | 声明后可通过路由 ctx 访问会话相关 API |

**参考**：圆桌讨论插件 (`hana-roundtable`) 声明了 `["model", "session"]`。

```js
// routes/ui.js — 声明 capabilities: ["model"] 后可用
export default function(app, ctx) {
  app.post("/api/analyze", async (c) => {
    // ctx.model 现在可用了
    const response = await ctx.model.sample({
      messages: [
        { role: "system", content: "你是分析师" },
        { role: "user", content: "分析这只股票" }
      ],
      maxTokens: 2000,
    });
    return c.json({ ok: true, text: response?.text || "" });
  });
}
```

**注意事项**：
- `capabilities: ["model"]` 需要 `trust: "full-access"` 才生效
- 工具（`tools/*.js`）的 ctx **始终**有 `ctx.model`（不需要声明 capabilities）
- 只有路由（`routes/ui.js`）的 ctx 需要通过 capabilities 声明来获取 model
- 如果路由里 `ctx.model` 为 undefined，检查 manifest 是否声明了 capabilities

### sessionPermission 声明（0.341.19+）

工具需要声明 `sessionPermission`，让 Hana 的 session 权限模式能做精确判断。

```js
export const name = "create_note";
export const description = "Create a markdown note.";
export const sessionPermission = { kind: "plugin_output" };
```

`describeSideEffect()` 是独立导出的函数，不是 `sessionPermission` 的属性：

```js
export function describeSideEffect(input) {
  return {
    kind: "session_file_output",
    summary: "Write a markdown file and register as SessionFile.",
    ruleId: "plugin-output-session-file"
  };
}
```
```

### trust 选项

| 值 | 权限 | 适用场景 |
|------|------|---------|
| `"restricted"` | 沙箱，无文件系统访问，无 iframe | 纯工具插件 |
| `"full-access"` | 可读写文件、iframe、网络 | 有页面或需要持久化的插件 |

---

## 六、`c.req.query()` 详细说明

**重要**：`c.req.query` 是一个函数，不是 URLSearchParams 对象。

```js
// ✅ 正确
const token = c.req.query("token") || "";
const page = Number(c.req.query("page") || 1);
const multi = c.req.query("tags"); // 返回字符串，如 "a,b,c"

// ❌ 错误
const params = new URLSearchParams(c.req.query); // c.req.query 不是对象
const token = params.get("token");
```

### 多值参数

如果 URL 是 `?tag=a&tag=b`：
```js
// c.req.query 返回第一个值 "a"
const first = c.req.query("tag");

// 多值需要通过 c.req.rawQuery 或自行解析
// 当前框架没有直接提供多值接口，建议前端用 JSON 数组传参
```

---

## 七、日志 API 详细说明

### 签名

```js
ctx.log?.info?.("消息", { key: value, ... });
ctx.log?.error?.("消息", { key: value, ... });
```

### 规则

| 规则 | 说明 |
|------|------|
| 可选链 `?.` | 框架可能不提供 log，用 `?.` 防止崩溃 |
| 第二个参数 | 结构化对象，会被序列化为 JSON |
| 不要 log 敏感信息 | token、密码、完整 URL 禁止输出 |
| 日志级别 | `info` 正常流程，`error` 异常处理 |

### 示例

```js
// 好
ctx.log?.info?.("fetch shelf success", { count: books.length });
ctx.log?.error?.("fetch shelf failed", { code: err.code, message: err.message });

// 坏
ctx.log?.info?.("user token", token);                    // 泄露 token
ctx.log?.info?.("url", "http://user:pass@host/api");     // 泄露凭据
```

---

## 八、文件系统 API

**注意**：涉及 `dataDir` 文件写入的操作必须走系统2（慢思考）。

### 插件私有数据（`ctx.dataDir`）

插件私有数据始终可用（`full-access` 插件），无需声明能力。

#### 路径规范

```js
const { dataDir, pluginId } = ctx;

// 读取配置
const fp = path.join(dataDir, pluginId, "config.json");

// 写入配置
const dir = path.join(dataDir, pluginId);
fs.mkdirSync(dir, { recursive: true });
fs.writeFileSync(path.join(dir, "config.json"), JSON.stringify(obj));

// 缓存
const cacheDir = path.join(dataDir, pluginId, "cache");
const cachePath = path.join(cacheDir, `${key}.json`);
```

### 视频资产（0.333.0+）

`assets/` 目录现在支持视频文件（mp4/webm）。iframe 中通过 <video> 标签引用：

### 媒体交付（0.341.19+）

工具需要交付文件时，使用 `stageFile()` 把本地文件登记成当前 session 的 SessionFile，并直接复用返回的 `mediaItem`。

```js
export const name = "generate_image";
export const sessionPermission = { kind: "session_file_output" };

export async function execute(input, ctx) {
  const filePath = path.join(ctx.dataDir, "output.png");
  // ... 生成图片 ...
  
  const sf = await ctx.stageFile({
    filePath,
    kind: "plugin_output"
  });
  
  return { ok: true, mediaItem: sf };
}
```

**注意**：
- 宿主不支持的视频格式会自动降级为 session file
- 注意 CSP 限制，视频不能跨域加载

```js
// routes/ui.js 中返回视频路径
app.get("/api/video", (c) => {
  return c.json({ url: \`/api/plugins/\${ctx.pluginId}/assets/demo.mp4\` });
});
```

**注意**：
- 视频放在 `assets/` 下，通过路由 `/api/plugins/<pluginId>/assets/` 访问
- 宿主不支持的视频格式会自动降级为 session file
- 注意 CSP 限制，视频不能跨域加载

### 安全规则

| 规则 | 说明 |
|------|------|
| 不要写源码目录 | `ctx.pluginDir` 只读，`ctx.dataDir` 可写 |
| 路径检查 | 如果路径来自用户输入，用 `path.resolve` 检查是否在 dataDir 内 |
| 原子写入 | 重要文件先写 `.tmp` 再 rename，避免半写崩溃 |
| 编码 | 所有文本文件用 UTF-8 无 BOM |
| 跨盘路径 | 0.333.0+ 已修复 Windows 跨盘插件 SDK 路径问题，确保用正斜杠 `/`（不要用反斜杠 `\`） |

### ResourceIO（0.341.19+）

如需访问用户资源（本地文件、SessionFile、挂载等），使用 `ctx.resources` 统一 API。详见 **第八节：ResourceIO 统一资源访问**。

> **迁移建议**：`ctx.dataDir` 操作保持不变；需要读写用户文件时用 `ctx.resources` 替代 `fs.*`。

---

## 八、ResourceIO 统一资源访问（0.341.19+）

**来源**：`feat(resource-io): converge file tools and watches` 系列 commit

0.341.19 引入了全新的 ResourceIO 系统，把所有文件操作（本地文件、挂载、SessionFile、URL、Resource）统一到一个 API。

### 声明能力

在 `manifest.json` 中声明插件需要的资源能力：

```json
{
  "capabilities": [
    "resource.read",
    "resource.search",
    "resource.write"
  ]
}
```

| 能力 | 说明 |
|------|------|
| `resource.read` | 读取资源（stat/read/list） |
| `resource.search` | 搜索资源（包括 provider 选项中的文件名搜索） |
| `resource.write` | 写入资源（write/edit/mkdir/delete/copy/rename/move/trash） |

### 请求级上下文（0.345.3+）

每个进入插件 route 的 HTTP 请求都会得到一份独立的请求级上下文 `pluginRequestContext`。新 route 建议通过 `@hana/plugin-runtime` 的 `getPluginRequestContext(c)` 读取：

```js
import { getPluginRequestContext } from "@hana/plugin-runtime";

app.post("/create-session", async (c) => {
  const reqCtx = getPluginRequestContext(c);
  // reqCtx.principal 本次请求的来源身份（owner 设备 / 本插件 iframe surface…）
  // reqCtx.agentId 代理层注入的当前 agent id
  // reqCtx.capabilityGrant { accessLevel, declaredPermissions, legacyDeclaration }
  
  // 老写法仍兼容：c.req.context.get("pluginRequestContext")
});
```

### 媒体交付

工具需要交付文件时，使用 `stageFile()` 把本地文件登记成当前 session 的 SessionFile，并直接复用返回的 `mediaItem`。

```js
export const name = "generate_image";
export const sessionPermission = { kind: "session_file_output" };

export async function execute(input, ctx) {
  const filePath = path.join(ctx.dataDir, "output.png");
  // ... 生成图片 ...
  
  const sf = await ctx.stageFile({
    filePath,
    kind: "plugin_output"
  });
  
  return { ok: true, mediaItem: sf };
}
```

### 可视化卡片（0.345.3+）

工具可以在聊天中自动渲染可视化卡片，在返回值的 `details.card` 中声明。当前有两条稳定形态：

**WebView / iframe 卡片**：用于插件自己的 Web UI、远程网站、单独 HTML 或复杂浏览器 UI。

```js
return {
  content: [{ type: "text", text: "数据摘要..." }],
  details: {
    card: {
      type: "webview",      // 或 "iframe"（兼容）
      route: "/card/chart?symbol=sh600519",
      title: "贵州茅台 日K",
      description: "贵州茅台 现价1450.00 涨跌+2.11%"
    }
  }
};
```

**原生聊天 surface 卡片**：把插件自己创建的 plugin_private session 作为原生聊天 transcript 嵌进当前聊天流。

```js
import { createChatSurfaceCard, createSession } from "@hana/plugin-runtime";

const child = await createSession(ctx, {
  kind: "tavern-run",
  visibility: "plugin_private",
  cwd: ctx.dataDir
});

return {
  content: [{ type: "text", text: "已创建插件私有会话。" }],
  details: {
    card: createChatSurfaceCard(ctx, child.child, {
      title: "Tavern run",
      description: "插件私有会话 transcript"
    })
  }
};
```

> **注意**：`chat.surface` 在 main 当前版本是薄兼容层，只提供原生 transcript 展示；复杂 composer、可组合 native cards 和组件生态会由 Infinity Chalkboard 逐步补齐。

### 注意事项

- `ctx.resources` 是 0.341.19 新增的统一层，替代了原来分散的 `fs.*` 调用
- 插件私有数据（`ctx.dataDir`）仍然可用，不需要声明能力
- 用户资源操作需要在 manifest.json 中声明 `capabilities`
- ResourceIO 是用户资源的唯一权限入口，`local-file`/`mount`/`session-file`/`resource`/`url` 都是资源身份，不等于插件能拿到宿主本机路径
- `stageFile()` 只用于插件生成物进入 SessionFile 交付链路，不用于修改用户源文件
- `ctx.resources.materialize(ref)` 用于把资源实体化成本机路径

```js
export default async function my_tool(input, ctx) {
  // 构造 resource ref
  const ref = { kind: "local-file", path: input.filePath };
  
  // 读取资源
  const file = await ctx.resources.read(ref);
  const content = file.toString("utf-8");
  
  return { ok: true, content };
}
```

### 写入资源

```js
export default async function my_tool(input, ctx) {
  const ref = { kind: "local-file", path: input.filePath };
  
  // 写入内容
  await ctx.resources.write(
    ref,
    { content: input.content, encoding: "utf-8" }
  );
  
  return { ok: true, written: input.filePath };
}
```

### 资源操作方法

| 方法 | 说明 | 需要能力 |
|------|------|----------|
| `ctx.resources.stat(ref)` | 获取资源元信息 | `resource.read` |
| `ctx.resources.read(ref)` | 读取资源内容 | `resource.read` |
| `ctx.resources.list(ref)` | 列出目录内容 | `resource.read` |
| `ctx.resources.search(query)` | 搜索资源 | `resource.search` |
| `ctx.resources.write(ref, data)` | 写入资源 | `resource.write` |
| `ctx.resources.edit(ref, edits)` | 编辑资源 | `resource.write` |
| `ctx.resources.mkdir(ref)` | 创建目录 | `resource.write` |
| `ctx.resources.delete(ref)` | 删除资源 | `resource.write` |
| `ctx.resources.copy(from, to)` | 复制资源 | `resource.write` |
| `ctx.resources.rename(from, to)` | 重命名资源 | `resource.write` |
| `ctx.resources.move(from, to)` | 移动资源 | `resource.write` |
| `ctx.resources.trash(ref)` | 移到回收站 | `resource.write` |
| `ctx.resources.materialize(ref)` | 物化成本机路径 | `resource.read` |
| `ctx.resources.watch(ref)` | 监听单个资源变更 | `resource.read` |
| `ctx.resources.subscribe(refs)` | 监听一组资源变更 | `resource.read` |

### Resource Watch Helpers（0.345.3+）

新增资源监听辅助方法，替代手动轮询：

```js
// 监听单个资源
c const watchHandle = await ctx.resources.watch(ref);
// watchHandle.unsubscribe() 停止监听

// 监听一组资源
c const subHandle = await ctx.resources.subscribe([refA, refB]);
// 返回 { subscriptionId, resourceKeys, unsubscribe, close }
// 资源变化通过插件 bus 的 resource.changed / resource.deleted / resource.renamed 事件到达
// 消费侧按 resourceKeys 过滤后再刷新或重渲染
```

> **生命周期**：`unsubscribe()` 交给 `register()` / `finally` 管理；短任务在 `finally` 中释放。

### Resource Ref 格式

```js
// 本地文件
{ kind: "local-file", path: "/absolute/path/to/file" }

// 挂载文件
{ kind: "mount", mountId: "xxx", path: "/path/in/mount" }

// SessionFile
{ kind: "session-file", fileId: "xxx" }

// Resource 记录
{ kind: "resource", resourceId: "xxx" }

// URL
{ kind: "url", url: "https://example.com/file" }
```

### 与旧 API 的关系

- `ctx.dataDir` 仍然可用，用于插件私有数据
- `ctx.resources` 是新增的统一层，适用于用户资源（本地文件、SessionFile 等）
- 插件私有数据（`dataDir`）不需要声明能力，`full-access` 插件自动可用
- 用户资源操作需要声明对应的 `capabilities`

---

## 十、Plugin SDK 宿主通信（0.345.3+）

**来源**：`feat(plugin): add native chat surface cards` / `feat(plugin): stabilize SDK discovery and dev loop context`

新插件建议使用 `@hana/plugin-sdk` 发送握手和宿主请求，替代手写 `postMessage`。

### 基础用法

```js
import { hana } from "@hana/plugin-sdk";

// 发送 ready 握手
window.parent.postMessage({ type: "ready" }, "*");

// 调整 iframe 高度
await hana.ui.resize({ height: 320 });

// Toast 提示
await hana.toast.show({ message: "已刷新", type: "success" });

// 打开外部链接
await hana.external.open("https://example.com");

// 写入剪贴板
await hana.clipboard.writeText("复制内容");

// 打开资源（SessionFile 等）
await hana.resources.open({
  resource: { kind: "session-file", fileId: "sf_1" }
});
```

### 能力声明

宿主只接受来自当前 iframe window 且 origin 匹配的消息。SDK 请求会经过 capability registry；当前内置能力包括：

| 能力 | 是否需要授权 | 说明 |
|------|-------------|------|
| `toast.show` | 否 | 无权限要求的 toast 提示 |
| `external.open` | 是 | 打开外部链接 |
| `clipboard.writeText` | 是 | 写入剪贴板 |
| `resource.open` | 是 | 打开资源 |
| `resource.pick` | 是 | 选择资源 |
| `resource.requestAccess` | 是 | 申请资源访问权限 |

需要在 manifest.json 中声明需要的宿主能力：

```json
{
  "manifestVersion": 1,
  "ui": {
    "hostCapabilities": ["external.open", "clipboard.writeText", "resource.open"]
  }
}
```

> 未声明的敏感能力会返回 `CAPABILITY_DENIED`。未知能力名会在加载时被忽略；`toast.show` 不需要声明。

> `hana.resources.*` 只是在 iframe 中向宿主发请求：可以请求打开资源、选择资源、申请访问权限，但不能直接读取或写入文件内容。真正的用户资源读写必须走服务端 `ctx.resources`。

### 底层兼容

底层仍保留 `hana.host.request(type, payload)`，用于未来 capability 或实验能力；稳定能力优先使用 typed helpers（`hana.toast`、`hana.external` 等）。

---

## 十一、HTTP 客户端（Node 环境）

### 0.333.0+ 推荐使用 Plugin Network Fetch Contract

插件现在有了统一的网络请求合同（`feat: add plugin network fetch contract`），替代直接裸 `fetch`。

**优势**：自动处理超时、重试、错误分类、日志注入。

```js
// 旧方式（仍然可用）
const res = await fetch(url);

// 新方式（推荐）
import { pluginFetch } from "@hana/plugin-runtime";
const res = await pluginFetch(url, { timeout: 15000, retries: 2 });
```

### 基本用法（直接 fetch）

在服务端（`tools/*.js`、`routes/ui.js`）可以直接使用全局 `fetch`。

### 基本用法

```js
const res = await fetch("https://api.example.com/data", {
  method: "GET",
  headers: {
    "Accept": "application/json",
    "Authorization": `Bearer ${token}`,
  },
  signal: controller.signal,  // 用于超时
});

if (!res.ok) {
  throw new Error(`HTTP ${res.status}`);
}

const data = await res.json();
```

### 超时模式

```js
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 15000);

try {
  const res = await fetch(url, { signal: controller.signal });
  clearTimeout(timer);
  // ...
} catch (err) {
  clearTimeout(timer);
  if (err.name === "AbortError") {
    // 超时
  }
}
```

### WebSocket

```js
const ws = new WebSocket("wss://example.com/ws");
ws.onopen = () => { ws.send(JSON.stringify({ type: "hello" })); };
ws.onmessage = (event) => { const data = JSON.parse(event.data); };
ws.onerror = (err) => { /* ... */ };
ws.onclose = () => { /* ... */ };
```

---

## 十二、常见错误码参考

| HTTP 状态 | code | 说明 | 处理建议 |
|-----------|------|------|---------|
| 400 | `missing_query` / `bad_format` | 参数缺失或格式错误 | 检查前端传参 |
| 401 | `auth` | 认证失败 | 检查 token/凭据 |
| 403 | `forbidden` | 无权限 | 检查 token 是否带在 URL 上 |
| 404 | `not_found` | 资源不存在 | 检查 ID 是否正确 |
| 500 | `tool_error` | 工具内部错误 | 查看日志 |
| 502 | `connection` / `upstream` | 上游服务不可达 | 检查网络/服务地址 |
| 504 | `timeout` | 请求超时 | 检查网络或增大超时时间 |

---

## 十三、开发工作流速查

```bash
# 1. 创建目录
New-Item -ItemType Directory -Force -Path "plugin/routes", "plugin/tools", "plugin/assets", "plugin/skills/plugin"

# 2. 写文件（注意无 BOM）
[System.IO.File]::WriteAllText("plugin/manifest.json", jsonContent, [System.Text.UTF8Encoding]::new($false))

# 3. 安装到 dev slot
plugin.dev.install --path "W:/path/to/plugin"

# 4. 启用
plugin_dev_enable(pluginId="my-plugin", allowFullAccess=true)

# 5. 验证工具调用
plugin_dev_invoke_tool(pluginId="my-plugin", toolName="tool_name", input={...})

# 6. 查看诊断
plugin_dev_diagnostics(pluginId="my-plugin")

# 7. 热加载（修改代码后）
plugin_dev_reload(pluginId="my-plugin")

# 8. 发布到正式目录
Copy-Item -Recurse "plugin" "C:\Users\Administrator\.hanako\plugins\my-plugin"
```

---

## 十四、常见问题

**Q: 修改了代码但页面没变化？**
→ 禁用再启用插件，或调用 `plugin_dev_reload`

**Q: 工具调用返回 `undefined`？**
→ 检查工具是否 `export default` 或 `export const execute`

**Q: dataDir 路径是什么？**
→ 通常是 `C:\Users\Administrator\.hanako\plugins-data\<pluginId>\`

**Q: 怎么调试？**
→ `ctx.log?.info?.("debug", { data });` 然后查看 `plugin_dev_diagnostics` 的 logs

**Q: 怎么在 iframe 里调用工具？**
→ 通过 `window.parent.postMessage` 发送请求，由 host 代理调用；或通过 `api()` 函数调用内部 API（`routes/ui.js` 里注册的 `/api/*`）

**Q: 新插件用什么方式做 iframe 通信？**
→ 推荐使用 `@hana/plugin-sdk` 的 typed helpers（`hana.toast`、`hana.external`、`hana.clipboard` 等），详见 **第十节：Plugin SDK 宿主通信**

**Q: 怎么在聊天中渲染卡片？**
→ 工具返回值中声明 `details.card`，支持 `type: "webview"`（或 `"iframe"`）和 `type: "chat.surface"` 两种形态，详见 **ResourceIO → 可视化卡片**
