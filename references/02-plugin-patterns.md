# 插件模式手册
> 常见功能的可复制代码模板
> 每个模式都是"复制 → 改三行 → 能用"的程度
> 对应：plugin-scaffold-walkthrough.md 的 Phase 2+ 扩展
> 最后更新：2026-06-27（新增模式 20：路由内调用宿主 LLM — capabilities model 声明）

---

## 使用方式

1. 确认你的插件已经能跑通 walkthrough（Phase 1 合格）
2. 找到需要的模式
3. 复制模板代码
4. 替换 TODO 标注的三处
5. 在 routes/ui.js 注册路由（如果需要 iframe 调用）
6. 在 skills/SKILL.md 补充工具说明

---

## 目录结构（带 lib）

```
my-plugin/
├── manifest.json
├── routes/
│   └── ui.js              # 服务端路由
├── tools/
│   ├── _lib/              # 共享工具库（可选但推荐）
│   │   ├── credentials.js # 凭据加密存储
│   │   ├── cache.js       # 文件缓存
│   │   └── http.js        # HTTP 请求封装
│   ├── my_tool.js         # 具体工具
│   └── ...
├── assets/
│   └── panel.js           # 前端脚本
└── skills/
    └── my-plugin/
        └── SKILL.md
```

---

## 模式 1：持久化配置（无加密）

**适用**：保存用户设置、偏好、开关状态
**复杂度**：★☆☆☆☆

```js
// tools/_lib/config.js
import fs from "node:fs";
import path from "node:path";

export function readConfig(dataDir, pluginId) {
  const fp = path.join(dataDir, pluginId, "config.json");
  if (!fs.existsSync(fp)) return {};
  try { return JSON.parse(fs.readFileSync(fp, "utf-8")); }
  catch { return {}; }
}

export function writeConfig(dataDir, pluginId, obj) {
  const dir = path.join(dataDir, pluginId);
  fs.mkdirSync(dir, { recursive: true });
  fs.writeFileSync(path.join(dir, "config.json"), JSON.stringify(obj, null, 2));
}
```

**使用示例**：
```js
// tools/my_tool.js
import { readConfig, writeConfig } from "./_lib/config.js";

export default async function my_tool(input, ctx) {
  const { dataDir } = ctx;
  const pluginId = ctx.pluginId; // TODO: 替换为你的插件 ID

  // 读
  const cfg = readConfig(dataDir, pluginId);
  const theme = cfg.theme || "dark";

  // 写
  writeConfig(dataDir, pluginId, { ...cfg, theme: input.theme });

  return { ok: true, theme: input.theme };
}
```

**在 routes/ui.js 注册 API**（供前端调用）：
```js
app.get("/api/config", (c) => {
  const cfg = readConfig(ctx.dataDir, ctx.pluginId);
  return c.json({ ok: true, ...cfg });
});

app.post("/api/config", async (c) => {
  const body = await c.req.json().catch(() => ({}));
  writeConfig(ctx.dataDir, ctx.pluginId, body);
  return c.json({ ok: true });
});
```

---

## 模式 2：持久化凭据（AES-256-GCM 加密）

**适用**：存储 API key、服务地址、密码等敏感信息
**复杂度**：★★★☆☆
**来源**：legado-companion/tools/_lib/credentials.js

```js
// tools/_lib/credentials.js
// 直接复制 legado-companion 的 credentials.js 即可
// 使用时把 "legado-companion" 替换为你的插件 ID
```

**简化版（非加密，仅本地开发用）**：
```js
// tools/_lib/credentials.js
import fs from "node:fs";
import path from "node:path";

export function readCredentials(dataDir, pluginId) {
  const fp = path.join(dataDir, pluginId, "credentials.json");
  if (!fs.existsSync(fp)) return {};
  try { return JSON.parse(fs.readFileSync(fp, "utf-8")); }
  catch { return {}; }
}

export function writeCredentials(dataDir, pluginId, creds) {
  const dir = path.join(dataDir, pluginId);
  fs.mkdirSync(dir, { recursive: true });
  fs.writeFileSync(path.join(dir, "credentials.json"), JSON.stringify(creds, null, 2));
}
```

---

## 模式 3：HTTP 请求封装（0.333.0+ 推荐使用 Network Fetch Contract）

**适用**：调用外部 REST API、带超时、错误分类
**复杂度**：★★☆☆☆
**来源**：legado-companion/tools/_lib/legado-api.js

> **0.333.0+ 变更**：插件现在有统一的网络请求合同（`feat: add plugin network fetch contract`）。
> 推荐优先使用 contract，它自动处理超时、重试、错误分类、日志注入。
> 如果不需要高级特性，直接用下面的裸 `fetch` 封装也可以。

```js
// tools/_lib/http.js
const DEFAULT_TIMEOUT_MS = 15000;

export class AuthError extends Error {
  constructor(m) { super(m); this.name = "AuthError"; this.code = "auth"; }
}
export class TimeoutError extends Error {
  constructor(m) { super(m); this.name = "TimeoutError"; this.code = "timeout"; }
}
export class ConnectionError extends Error {
  constructor(m) { super(m); this.name = "ConnectionError"; this.code = "connection"; }
}

export async function fetchJson(url, options = {}, timeoutMs = DEFAULT_TIMEOUT_MS) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const res = await fetch(url, { ...options, signal: controller.signal });
    clearTimeout(timer);
    if (!res.ok) {
      const text = await res.text().catch(() => "");
      throw new ConnectionError(`HTTP ${res.status}: ${text.slice(0, 500)}`);
    }
    return await res.json();
  } catch (err) {
    clearTimeout(timer);
    if (err.name === "AbortError") throw new TimeoutError(`超时 (${timeoutMs}ms)`);
    if (err.code === "ECONNREFUSED" || err.code === "ENOTFOUND") {
      throw new ConnectionError(`无法连接: ${err.message}`);
    }
    throw err;
  }
}
```

**使用示例**：
```js
// tools/my_api_tool.js
import { fetchJson, ConnectionError } from "./_lib/http.js";

export default async function my_api_tool(input, ctx) {
  try {
    const data = await fetchJson(`https://api.example.com/data?q=${encodeURIComponent(input.q)}`);
    return { ok: true, data };
  } catch (err) {
    if (err instanceof ConnectionError) {
      return { ok: false, code: "connection", message: err.message };
    }
    return { ok: false, code: "error", message: err.message };
  }
}
```

---

## 模式 4：WebSocket 通信

**适用**：实时数据、搜索、推送
**复杂度**：★★★☆☆
**来源**：legado-companion/tools/_lib/legado-api.js 的 searchBooks

```js
// tools/my_ws_tool.js
export default async function my_ws_tool(input, ctx) {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      ws.close();
      reject(new Error("WebSocket timeout"));
    }, 15000);

    let ws;
    try {
      ws = new WebSocket("wss://example.com/ws");
    } catch (err) {
      clearTimeout(timeout);
      reject(new Error(`WebSocket 创建失败: ${err.message}`));
      return;
    }

    ws.onopen = () => {
      ws.send(JSON.stringify({ type: "query", q: input.q }));
    };

    ws.onmessage = (event) => {
      clearTimeout(timeout);
      ws.close();
      try {
        const data = JSON.parse(event.data);
        resolve({ ok: true, data });
      } catch {
        resolve({ ok: true, raw: event.data });
      }
    };

    ws.onerror = (err) => {
      clearTimeout(timeout);
      reject(new Error(`WebSocket 错误: ${err.message || "unknown"}`));
    };
  });
}
```

---

## 模式 5：文件缓存

**适用**：减少重复请求、加速加载、离线可用
**复杂度**：★★☆☆☆
**来源**：legado-companion/tools/_lib/cache.js

```js
// tools/_lib/cache.js
import fs from "node:fs";
import path from "node:path";

const CACHE_TTL_MS = 7 * 24 * 60 * 60 * 1000; // 7 天，按需调整

export function getCacheDir(dataDir, pluginId) {
  return path.join(dataDir, pluginId, "cache");
}

export function readCache(dataDir, pluginId, key) {
  const fp = path.join(getCacheDir(dataDir, pluginId), `${key}.json`);
  if (!fs.existsSync(fp)) return null;
  try {
    const content = JSON.parse(fs.readFileSync(fp, "utf-8"));
    if (Date.now() - content.timestamp > CACHE_TTL_MS) {
      fs.unlinkSync(fp);
      return null;
    }
    return content.data;
  } catch { return null; }
}

export function writeCache(dataDir, pluginId, key, data) {
  const dir = getCacheDir(dataDir, pluginId);
  fs.mkdirSync(dir, { recursive: true });
  fs.writeFileSync(path.join(dir, `${key}.json`), JSON.stringify({
    timestamp: Date.now(),
    data,
  }));
}
```

**使用示例**：
```js
// tools/my_tool.js
import { readCache, writeCache } from "./_lib/cache.js";

export default async function my_tool(input, ctx) {
  const { dataDir } = ctx;
  const pluginId = ctx.pluginId; // TODO: 替换

  // 先读缓存
  const cached = readCache(dataDir, pluginId, "my_key");
  if (cached) return { ok: true, data: cached, cached: true };

  // 缓存未命中，获取新数据
  const fresh = await fetchExpensiveData(input.q);

  // 写入缓存
  writeCache(dataDir, pluginId, "my_key", fresh);

  return { ok: true, data: fresh, cached: false };
}
```

---

## 模式 6：调用宿主 LLM

**适用**：对话、总结、翻译、问答
**复杂度**：★★☆☆☆
**来源**：legado-companion/tools/_lib/llm.js

```js
// tools/_lib/llm.js
export async function callLLM(ctx, prompt, systemPrompt = "你是一个有用的助手。") {
  if (!ctx || !ctx.model) {
    return { ok: false, message: "LLM 上下文不可用" };
  }

  try {
    const response = await ctx.model.sample({
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: prompt },
      ],
      maxTokens: 2000,
    });
    return { ok: true, text: response?.text || response?.content || "" };
  } catch (err) {
    return { ok: false, message: `LLM 调用失败: ${err.message}` };
  }
}
```

**使用示例**：
```js
// tools/my_llm_tool.js
import { callLLM } from "./_lib/llm.js";

export default async function my_llm_tool(input, ctx) {
  const result = await callLLM(
    ctx,
    `请总结以下内容：\n${input.text}`,
    "你是一个专业的文本摘要助手。"
  );

  if (!result.ok) return result;

  return {
    ok: true,
    summary: result.text,
  };
}
```

---

## 模式 7：iframe 内部 API 代理

**适用**：前端需要调用外部 API，但存在 CORS；或需要隐藏 API key
**复杂度**：★★☆☆☆
**来源**：weread-companion/routes/ui.js 的 proxy-image

```js
// 在 routes/ui.js 中添加
app.get("/api/proxy", async (c) => {
  const url = c.req.query("url") || "";
  if (!/^https?:\/\//i.test(url)) {
    return c.json({ ok: false, code: "bad_url" }, 400);
  }

  try {
    const resp = await fetch(url);
    if (!resp.ok) return c.json({ ok: false, code: "upstream" }, 502);

    const buf = Buffer.from(await resp.arrayBuffer());
    const ct = resp.headers.get("content-type") || "application/octet-stream";

    return new Response(buf, {
      headers: {
        "Content-Type": ct,
        "Cache-Control": "public, max-age=86400",
      },
    });
  } catch (err) {
    return c.json({ ok: false, code: "proxy_failed", message: err.message }, 500);
  }
});
```

**前端调用**：
```js
async function loadImage(url) {
  const res = await fetch(`/api/proxy?url=${encodeURIComponent(url)}`);
  const blob = await res.blob();
  return URL.createObjectURL(blob);
}
```

---

## 模式 8：前端状态与 API 调用

**适用**：iframe 页面需要和宿主通信、读写配置、调用工具
**复杂度**：★★★☆☆

```js
// assets/panel.js
(function() {
  "use strict";

  const base = window.HANA_PLUGIN_BASE || "/api/plugins/my-first-plugin";
  const token = window.HANA_TOKEN || "";

  function authQuery() {
    const t = new URLSearchParams(window.location.search).get("token");
    return t ? `?token=${encodeURIComponent(t)}` : "";
  }

  async function api(path, options = {}) {
    const url = `${base}${path}${authQuery()}`;
    const res = await fetch(url, {
      headers: { "Content-Type": "application/json" },
      ...options,
    });
    if (!res.ok) {
      const text = await res.text();
      throw new Error(`${path} ${res.status}: ${text}`);
    }
    return res.json();
  }

  // 初始化
  window.addEventListener("DOMContentLoaded", async () => {
    // 1. 读取配置
    const cfg = await api("/api/config");
    console.log("[my-plugin] config:", cfg);

    // 2. 调用工具（通过 iframe API）
    // 注意：直接调用工具需要走宿主协议，见模式 9

    // 3. 发送 ready
    window.parent.postMessage({
      protocol: "hana.plugin.ui",
      version: 1,
      kind: "event",
      type: "hana.ready",
    }, "*");
  });
})();
```

---

## 模式 9：与宿主通信（postMessage 协议）

**适用**：iframe 需要和 Hana host 交换数据、触发动作
**复杂度**：★★★☆☆

**iframe → Host（发送事件）**：
```js
// 基础事件
window.parent.postMessage({ type: "ready" }, "*");

// 标准协议事件
window.parent.postMessage({
  protocol: "hana.plugin.ui",
  version: 1,
  kind: "event",
  type: "hana.ready",
  data: { /* 任意数据 */ },
}, "*");
```

**Host → iframe（接收指令）**：
```js
window.addEventListener("message", (event) => {
  // 生产环境应验证 event.origin
  const msg = event.data;

  if (msg.type === "set-theme") {
    document.body.dataset.hanaTheme = msg.theme;
  }

  if (msg.type === "resize") {
    // host 通知 iframe 尺寸变化
    document.body.style.height = msg.height + "px";
  }
});
```

**常见事件类型**：
| 事件名 | 方向 | 用途 |
|--------|------|------|
| `ready` | iframe → host | 告知已加载完成 |
| `hana.ready` | iframe → host | 标准协议 ready |
| `set-theme` | host → iframe | 切换主题 |
| `resize` | host → iframe | 通知尺寸变化 |
| `navigate` | host → iframe | 切换内部页面 |

---

## 模式 10：完整工具返回值规范

**适用**：所有工具的统一返回格式
**复杂度**：★☆☆☆☆

```js
// 成功（简单）
return "纯文本结果";

// 成功（结构化）
return {
  ok: true,
  data: { /* 任意数据 */ },
};

// 失败
return {
  ok: false,
  code: "错误码",      // 可选，用于前端分类处理
  message: "用户可读的错误信息",
  hint: "修复建议",     // 可选
};

// 媒体不支持时降级为 session file（0.333.0+）
return {
  ok: true,
  data: { /* 媒体数据 */ },
  fallbackType: "session_file",  // 宿主不支持此格式时自动转为 session file
};
```

**错误码约定**：
| code | 含义 | HTTP 映射 |
|------|------|-----------|
| `no_service` | 未配置服务地址 | 400 |
| `auth` | 认证失败 | 401 |
| `not_found` | 资源不存在 | 404 |
| `connection` | 网络错误 | 502 |
| `timeout` | 请求超时 | 504 |
| `bad_format` | 参数格式错误 | 400 |
| `tool_error` | 工具内部错误 | 500 |

---

## 模式 11：日志与调试

**适用**：开发时排查问题
**复杂度**：★☆☆☆☆

```js
// tools/my_tool.js
export default async function my_tool(input, ctx) {
  // 1. 结构化日志
  ctx.log?.info?.("my_tool start", { input, pluginId: ctx.pluginId });

  try {
    const result = await doSomething(input);
    ctx.log?.info?.("my_tool success", { result });
    return { ok: true, result };
  } catch (err) {
    // 2. 错误日志（不要泄露敏感信息）
    ctx.log?.error?.("my_tool failed", {
      message: err.message,
      code: err.code,
      // 不要 log password / token / 完整 URL
    });
    return { ok: false, code: "error", message: err.message };
  }
}
```

**查看日志**：
```bash
# 方式1：plugin_dev_diagnostics
# 在对话中调用插件诊断工具查看 logs 数组

# 方式2：查看 dev run 文件
Get-Content "C:\Users\Administrator\.hanako\plugin-dev-runs\my-first-plugin\dev_xxx.json"
```

---

## 模式 13：iframe API Fetch Helper（0.333.0+）

**适用**：iframe 内统一 API 请求，替代手写 fetch
**复杂度**：★☆☆☆☆
**来源**：`feat: add plugin iframe API fetch helper`

框架现在提供了统一的 iframe API fetch helper，自动处理 token 注入、base URL 拼接、错误分类。

```js
// assets/panel.js — 使用框架提供的 helper
(async function() {
  // 1. 框架已注入 base URL 和 token
  const base = window.HANA_PLUGIN_BASE || "/api/plugins/my-plugin";
  const token = window.HANA_TOKEN || "";

  // 2. 使用 helper（框架自动处理 authQuery）
  const helper = window.__hanaPluginApi || {
    // 降级：框架未注入时使用手写版本
    async fetch(path, options = {}) {
      const url = base + path + (token ? "?token=" + encodeURIComponent(token) : "");
      const res = await fetch(url, {
        headers: { "Content-Type": "application/json" },
        ...options,
      });
      if (!res.ok) {
        const text = await res.text();
        throw new Error(path + " " + res.status + ": " + text);
      }
      return res.json();
    }
  };

  // 3. 调用
  const cfg = await helper.fetch("/api/config");
  console.log("config:", cfg);
})();
```

**与手写 fetch 的区别**：
| 方面 | 手写 fetch | iframe helper |
|------|-----------|---------------|
| token 注入 | 手动拼接 `authQuery()` | 自动 |
| base URL | 硬编码或 `window.HANA_PLUGIN_BASE` | 框架管理 |
| 错误分类 | 自己写 | 框架统一返回 `{code, message}` |
| 降级 | 无 | 框架未注入时自动 fallback 到手写 |

> **注意**：如果你的插件已经有成熟的 `api()` 函数（见模式 8），不需要立即迁移。
> 新插件推荐使用 helper，老插件可以渐进迁移。

---

## 模式 12：渐进式迁移（restricted → full-access）

**适用**：先做纯工具，后来需要 iframe 页面
**复杂度**：★★☆☆☆

**步骤**：
1. 初始：`trust` 字段写 `"restricted"`，只有 `tools/`
2. 需要 iframe：改成 `"full-access"`，加 `routes/ui.js` + `assets/`
3. 在 manifest 加 `contributes.page`
4. 重新安装插件（或禁用再启用）

**注意**：权限变更必须走系统2（慢思考），因为涉及安全边界。

---

## 模式 14：ResourceIO 文件操作（0.341.19+）

**适用**：读写用户本地文件、SessionFile 传递、挂载文件操作
**复杂度**：★★☆☆☆
**来源**：`feat(resource-io): converge file tools and watches` 系列 commit

### 基本读取

```js
// tools/read_file.js
export const name = "read_file";
export const description = "Read a file from the user's system.";
export const sessionPermission = { readOnly: true };
export const parameters = {
  type: "object",
  properties: {
    path: { type: "string", description: "文件路径" }
  },
  required: ["path"]
};

export async function execute(input, ctx) {
  const ref = { kind: "local-file", path: input.path };
  const file = await ctx.resources.read(ref);
  const content = file.toString("utf-8");
  return { ok: true, content, path: input.path };
}
```

### 基本写入

```js
// tools/write_file.js
export const name = "write_file";
export const description = "Write content to a file on the user's system.";
export const sessionPermission = { kind: "plugin_output" };
export const parameters = {
  type: "object",
  properties: {
    path: { type: "string", description: "文件路径" },
    content: { type: "string", description: "文件内容" }
  },
  required: ["path", "content"]
};

export async function execute(input, ctx) {
  const ref = { kind: "local-file", path: input.path };
  await ctx.resources.write(ref, { content: input.content, encoding: "utf-8" });
  return { ok: true, written: input.path };
}
```

### 列出目录

```js
// tools/list_dir.js
export const name = "list_dir";
export const description = "List files in a directory.";
export const sessionPermission = { readOnly: true };
export const parameters = {
  type: "object",
  properties: {
    path: { type: "string", description: "目录路径" }
  },
  required: ["path"]
};

export async function execute(input, ctx) {
  const ref = { kind: "local-file", path: input.path };
  const entries = await ctx.resources.list(ref);
  return { ok: true, entries };
}
```

### 搜索文件

```js
// tools/search_files.js
export const name = "search_files";
export const description = "Search for files by name pattern.";
export const sessionPermission = { readOnly: true };
export const parameters = {
  type: "object",
  properties: {
    pattern: { type: "string", description: "搜索模式（支持 glob）" },
    scope: { type: "string", description: "搜索范围路径", default: "/" }
  },
  required: ["pattern"]
};

export async function execute(input, ctx) {
  const results = await ctx.resources.search({
    pattern: input.pattern,
    scope: input.scope || "/"
  });
  return { ok: true, results };
}
```

### 文件操作（复制/移动/删除）

```js
// tools/copy_file.js
export const name = "copy_file";
export const description = "Copy a file from source to destination.";
export const sessionPermission = { kind: "plugin_output" };
export const parameters = {
  type: "object",
  properties: {
    source: { type: "string" },
    destination: { type: "string" }
  },
  required: ["source", "destination"]
};

export async function execute(input, ctx) {
  const from = { kind: "local-file", path: input.source };
  const to = { kind: "local-file", path: input.destination };
  await ctx.resources.copy(from, to);
  return { ok: true, copied: input.source, to: input.destination };
}

// tools/delete_file.js
export const name = "delete_file";
export const description = "Delete a file (moves to trash).";
export const sessionPermission = { kind: "plugin_output" };
export const parameters = {
  type: "object",
  properties: {
    path: { type: "string" }
  },
  required: ["path"]
};

export async function execute(input, ctx) {
  const ref = { kind: "local-file", path: input.path };
  await ctx.resources.trash(ref);
  return { ok: true, deleted: input.path };
}
```

### Resource Ref 格式速查

| 格式 | 示例 | 说明 |
|------|------|------|
| 本地文件 | `{ kind: "local-file", path: "/abs/path" }` | 绝对路径 |
| 挂载文件 | `{ kind: "mount", mountId: "xxx", path: "/in/mount" }` | 挂载点内的文件 |
| SessionFile | `{ kind: "session-file", fileId: "xxx" }` | 当前会话的文件 |
| Resource | `{ kind: "resource", resourceId: "xxx" }` | Resource 记录 |
| URL | `{ kind: "url", url: "https://..." }` | 远程 URL |

### 注意事项

- `ctx.resources` 是 0.341.19 新增的统一层，替代了原来分散的 `fs.*` 调用
- 插件私有数据（`ctx.dataDir`）仍然可用，不需要声明能力
- 用户资源操作需要在 manifest.json 中声明 `capabilities`
- 路径必须用正斜杠 `/`，不要用反斜杠 `\`

---

## 模式 15：sessionPermission 声明（0.341.19+）

**适用**：工具需要返回文件、声明副作用、让权限系统做精确判断
**复杂度**：★☆☆☆☆
**来源**：`fix(permission): add plugin tool permission contract`

0.341.19 引入了 plugin permission contract，工具现在需要声明 `sessionPermission`。这是权限系统的核心契约。

### 只读工具（最简单）

```js
export const name = "read_config";
export const description = "Read plugin configuration.";
export const sessionPermission = { readOnly: true };
// ...
```

### 插件输出文件

```js
export const name = "create_note";
export const description = "Create a markdown note in plugin data.";
export const sessionPermission = { kind: "plugin_output" };
// 在 dataDir 下创建文件，不需要 stageFile
```

### SessionFile 输出（推荐用于媒体交付）

```js
export const name = "generate_image";
export const description = "Generate an image and return as SessionFile.";
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

### 带副作用描述（高级）

```js
export const name = "create_note";
export const description = "Create a markdown note in plugin data.";
export const sessionPermission = {
  kind: "plugin_output",
  describeSideEffect: () => ({
    kind: "session_file_output",
    summary: "Write a markdown file under plugin data and register it as SessionFile media.",
    ruleId: "plugin-output-session-file"
  })
};
```

### sessionPermission 值速查

| 值 | 说明 | 适用场景 |
|----|------|----------|
| `{ readOnly: true }` | 只读，无副作用 | 读取配置、查询数据 |
| `{ kind: "plugin_output" }` | 插件输出文件 | 在 dataDir 下创建文件 |
| `{ kind: "session_file_output" }` | SessionFile 输出 | 媒体交付、文件返回给宿主 |

### 与 capabilities 的关系

- `sessionPermission`：声明**单个工具**的行为和权限
- `capabilities`（manifest.json）：声明**整个插件**的资源访问能力
- 两者配合使用：capabilities 决定插件能做什么，sessionPermission 告诉宿主每个工具的具体行为

---

## 模式 16：Deferred Interludes（延迟插播，0.341.19+）

**适用**：需要异步推送消息、延迟投递、工作流编排的插件
**复杂度**：★★★☆☆
**来源**：`fix: order deferred interludes before replies` 系列 commit

0.341.19 引入了 deferred interludes 机制，允许插件在回复之前插入延迟消息。这对需要异步操作的插件（如长任务进度、批量处理结果）很有用。

### 基本模式：延迟消息推送

```js
// tools/long_task.js
export const name = "long_task";
export const description = "Run a long task with progress updates.";
export const sessionPermission = { kind: "plugin_output" };

export async function execute(input, ctx) {
  const results = [];
  
  for (let i = 0; i < input.count; i++) {
    // 模拟耗时操作
    await new Promise(r => setTimeout(r, 1000));
    results.push({ step: i + 1, status: "done" });
    
    // 推送进度（deferred interlude）
    // 宿主会在回复之前收集并投递这些插播消息
    ctx.bus?.emit?.("progress", {
      pluginId: ctx.pluginId,
      step: i + 1,
      total: input.count,
      data: results
    });
  }
  
  return { ok: true, results };
}
```

### 与 bus 通信

```js
// 发送事件
c ctx.bus.emit("event_type", data);

// 订阅事件（在工具中）
ctx.bus.subscribe("event_type", (data) => { /* ... */ });

// 请求-响应模式
c ctx.bus.request("tool_call", input).then(res => { /* ... */ });
```

### 注意事项

- 延迟插播会在回复之前按序排序（`order deferred interludes before replies`）
- 插播消息绑定到已消费的输入（`bind deferred interludes to consumed inputs`）
- 投递过程是幂等的（`preserve deferred interlude deliveries`）
- 如果你的插件不需要异步消息推送，可以忽略此模式

---

## 模式 17：WebView / chat.surface 卡片（0.345.3+）

**适用**：在聊天中自动渲染可视化卡片，替代纯文本回复
**复杂度**：★★☆☆☆
**来源**：`feat(plugin): add native chat surface cards`

工具返回值中声明 `details.card`，宿主会自动渲染为聊天卡片。

### WebView / iframe 卡片

用于插件自己的 Web UI、远程网站、单独 HTML 或复杂浏览器 UI。

```js
export const name = "chart_stock";
export const description = "Show stock chart as a card.";
export const sessionPermission = { kind: "plugin_output" };
export const parameters = {
  type: "object",
  properties: {
    symbol: { type: "string", description: "股票代码，如 sh600519" }
  },
  required: ["symbol"]
};

export async function execute(input, ctx) {
  return {
    content: [{ type: "text", text: `贵州茅台 现价1450.00 涨跌+2.11%` }],
    details: {
      card: {
        type: "webview",      // 或 "iframe"（旧卡兼容）
        route: `/card/chart?symbol=${input.symbol}&period=daily`,
        title: `贵州茅台 ${input.symbol} 日K`,
        description: `贵州茅台 现价1450.00 涨跌+2.11%`
      }
    }
  };
}
```

**字段说明**：
| 字段 | 说明 |
|------|------|
| `type` | `"webview"`（推荐）或 `"iframe"`（兼容旧卡） |
| `route` | 插件路由路径，iframe 自行从该路径拉数据渲染 |
| `title` | 卡片标题（可选） |
| `description` | 纯文本摘要，用于 IM 平台降级显示和插件卸载后的 fallback |

> `details.card` 中插件 route 发送的自定义消息如果携带同样的 `plugin_card`，也会被提取成 `details.card`，历史回放时保持一致。

### 原生聊天 surface 卡片

把插件自己创建的 plugin_private session 作为原生聊天 transcript 嵌进当前聊天流。

```js
import { createChatSurfaceCard, createSession } from "@hana/plugin-runtime";

export const name = "run_tavern";
export const description = "Run a tavern session and show as chat card.";
export const parameters = {
  type: "object",
  properties: {
    prompt: { type: "string", description: "初始提示词" }
  },
  required: ["prompt"]
};

export async function execute(input, ctx) {
  // 创建插件私有会话
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
}
```

> **注意**：`chat.surface` 在 main 当前版本是薄兼容层，只提供原生 transcript 展示；复杂 composer、可组合 native cards 和组件生态会由 Infinity Chalkboard 逐步补齐。

---

## 模式 18：Plugin SDK 宿主通信（0.345.3+）

**适用**：iframe 与宿主通信，替代手写 postMessage
**复杂度**：★☆☆☆☆
**来源**：`feat(plugin): stabilize SDK discovery and dev loop context`

新插件建议使用 `@hana/plugin-sdk` 的 typed helpers，替代手写 `window.parent.postMessage`。

### 基础用法

```js
// assets/panel.js
import { hana } from "@hana/plugin-sdk";

// 1. 发送 ready 握手
window.parent.postMessage({ type: "ready" }, "*");

// 2. 调整 iframe 高度
await hana.ui.resize({ height: 320 });

// 3. Toast 提示
await hana.toast.show({ message: "已刷新", type: "success" });

// 4. 打开外部链接
await hana.external.open("https://example.com");

// 5. 写入剪贴板
await hana.clipboard.writeText("复制内容");

// 6. 打开资源
await hana.resources.open({
  resource: { kind: "session-file", fileId: "sf_1" }
});
```

### 能力声明

需要在 manifest.json 中声明 iframe 需要的宿主能力：

```json
{
  "ui": {
    "hostCapabilities": ["external.open", "clipboard.writeText", "resource.open"]
  }
}
```

| 能力 | 是否需要授权 | 说明 |
|------|-------------|------|
| `toast.show` | 否 | 无权限要求的 toast 提示 |
| `external.open` | 是 | 打开外部链接 |
| `clipboard.writeText` | 是 | 写入剪贴板 |
| `resource.open` / `resource.pick` / `resource.requestAccess` | 是 | 资源相关操作 |

> 未声明的敏感能力会返回 `CAPABILITY_DENIED`。`toast.show` 不需要声明。

> `hana.resources.*` 只是在 iframe 中向宿主发请求，不能直接读取或写入文件内容。真正的用户资源读写必须走服务端 `ctx.resources`。

### 与 postMessage 的区别

| 方面 | 手写 postMessage | Plugin SDK |  |
|------|-----------------|------------|--|
| 握手 | 手动 `window.parent.postMessage` | 手动（仍需发 ready） |
| 能力注册 | 无 | 自动经过 capability registry |
| 错误处理 | 自己处理 | 返回 `CAPABILITY_DENIED` |
| 类型安全 | 无 | typed helpers |
| 降级 | 无 | 底层仍保留 `hana.host.request(type, payload)` |

> 如果你的插件已经有成熟的 `api()` 函数（见旧版模式 8），不需要立即迁移。新插件推荐使用 SDK，老插件可以渐进迁移。

---

## 模式 19：资源监听（0.345.3+）

**适用**：监听文件/资源变更，自动刷新 UI 或触发重算
**复杂度**：★★☆☆☆
**来源**：`feat(plugin): add resource watch helpers`

0.345.3 新增了 `ctx.resources.watch()` 和 `ctx.resources.subscribe()` 辅助方法，替代手动轮询。

### 监听单个资源

```js
// tools/watch_single.js
export const name = "watch_file";
export const description = "Watch a file for changes.";
export const sessionPermission = { readOnly: true };
export const parameters = {
  type: "object",
  properties: {
    path: { type: "string", description: "要监听的文件路径" }
  },
  required: ["path"]
};

export async function execute(input, ctx) {
  const ref = { kind: "local-file", path: input.path };
  
  // 开始监听
  const watchHandle = await ctx.resources.watch(ref);
  
  // watchHandle 包含 unsubscribe() 方法
  // 资源变化通过插件 bus 的 resource.changed / resource.deleted / resource.renamed 事件到达
  
  return {
    ok: true,
    watching: input.path,
    message: "监听已启动，资源变化会通过 bus 事件通知"
  };
}
```

### 监听一组资源

```js
// tools/watch_multiple.js
export const name = "watch_directory";
export const description = "Watch a directory for changes.";
export const sessionPermission = { readOnly: true };
export const parameters = {
  type: "object",
  properties: {
    paths: {
      type: "array",
      items: { type: "string" },
      description: "要监听的路径列表"
    }
  },
  required: ["paths"]
};

export async function execute(input, ctx) {
  const refs = input.paths.map(p => ({ kind: "local-file", path: p }));
  
  // 监听一组资源
  const subHandle = await ctx.resources.subscribe(refs);
  
  // 返回 { subscriptionId, resourceKeys, unsubscribe, close }
  // 消费侧按 resourceKeys 过滤后再刷新或重渲染
  
  return {
    ok: true,
    subscriptionId: subHandle.subscriptionId,
    watching: input.paths.length + " 个资源"
  };
}
```

### 生命周期管理

```js
// register() 中启动，finally 中释放
c app.get("/setup-watch", async (c) => {
  const ref = { kind: "local-file", path: "/path/to/watch" };
  const handle = await ctx.resources.watch(ref);
  
  // 注册清理
  ctx.register(() => {
    return handle.unsubscribe();
  });
  
  return c.json({ ok: true, watched: ref.path });
});

// 短任务直接在 finally 中释放
c try {
  const handle = await ctx.resources.watch(ref);
  // ... 使用 handle ...
} finally {
  handle?.unsubscribe();
}
```

### 注意事项

- `ctx.resources.watch(ref)` 用于单个资源，`ctx.resources.subscribe([refA, refB])` 用于一组资源
- 返回 `{ subscriptionId, resourceKeys, unsubscribe, close }`
- 生命周期插件应把 `unsubscribe` 交给 `register()` / `finally`；短任务应在 `finally` 中释放
- 资源变化仍通过插件 bus 的 `resource.changed` / `resource.deleted` / `resource.renamed` 事件到达
- 消费侧按 `resourceKeys` 过滤后再刷新或重渲染

---

## 模式 20：路由内调用宿主 LLM

**适用**：iframe 面板或路由 API 需要触发 LLM 分析的场景（如股票分析、内容总结、智能问答）
**复杂度**：★★☆☆☆
**前置条件**：manifest.json 声明 `"capabilities": ["model"]` + `"trust": "full-access"`

> **核心原理**：工具的 ctx 始终有 `ctx.model`，但路由 ctx 默认没有。必须在 manifest.json 中声明 `capabilities: ["model"]` 才会注入。

### manifest.json

```json
{
  "manifestVersion": 1,
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "0.1.0",
  "description": "需要在路由中调用 LLM 的插件",
  "trust": "full-access",
  "capabilities": ["model"],
  "contributes": {
    "page": {
      "title": "Analysis",
      "route": "/page"
    }
  }
}
```

### routes/ui.js

```js
export default function(app, ctx) {
  // ctx.model 现在可用了（因为 manifest 声明了 capabilities: ["model"]）

  app.post("/api/analyze", async (c) => {
    const body = await c.req.json().catch(() => ({}));
    const prompt = body.prompt || "";

    if (!prompt) {
      return c.json({ ok: false, code: "missing_prompt" }, 400);
    }

    try {
      const response = await ctx.model.sample({
        messages: [
          { role: "system", content: "你是一个专业的分析师。" },
          { role: "user", content: prompt },
        ],
        maxTokens: 2000,
      });

      return c.json({
        ok: true,
        text: response?.text || response?.content || "",
      });
    } catch (err) {
      return c.json(
        { ok: false, code: "llm_error", message: err.message },
        500
      );
    }
  });

  // 可选：GET 快捷接口
  app.get("/api/quick-analyze", async (c) => {
    const q = c.req.query("q") || "";
    if (!q) {
      return c.json({ ok: false, code: "missing_q" }, 400);
    }

    try {
      const response = await ctx.model.sample({
        messages: [
          { role: "system", content: "简洁回答用户问题。" },
          { role: "user", content: q },
        ],
        maxTokens: 1000,
      });

      return c.text(response?.text || "", 200);
    } catch (err) {
      return c.text(err.message, 500);
    }
  });
}
```

### assets/panel.js（前端调用示例）

```js
async function analyze(text) {
  const res = await fetch("/api/plugins/my-plugin/api/analyze", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ prompt: text }),
  });
  const data = await res.json();
  if (!data.ok) throw new Error(data.message || "分析失败");
  return data.text;
}

// 使用
analyze("分析一下贵州茅台的基本面").then(result => {
  console.log(result);
});
```

### 排查清单

| 症状 | 检查项 |
|------|--------|
| `ctx.model` 为 undefined | manifest.json 是否声明了 `"capabilities": ["model"]`？ |
| 工具里 `ctx.model` 可用但路由里不可用 | 正常现象 — 工具始终有 model，路由需要声明 |
| 报 `CAPABILITY_DENIED` | 是否同时设置了 `"trust": "full-access"`？ |
| 模型返回空文本 | 检查 `response?.text || response?.content` 两个字段 |

---

## 附录：工具函数签名速查

```js
// 标准工具（推荐）
export default async function tool_name(input, ctx) {
  // input: 前端传入的参数对象
  // ctx: 宿主上下文
  //   ctx.pluginId: 插件 ID
  //   ctx.dataDir: 数据目录（持久化根路径）
  //   ctx.log?.info?.() / ctx.log?.error?.(): 日志
  //   ctx.model?.sample(): 调用 LLM（仅 full-access）
  // return: 字符串 或 { ok, data/code/message }
}

// 带 name/description/parameters 的工具（对话可识别参数）
export const name = "tool_name";
export const description = "工具描述";
export const parameters = { type: "object", properties: {...}, required: [...] };
export async function execute(input, toolCtx) { ... }
```

---

## 附录：路由注册速查

```js
// routes/ui.js
export default function (app, ctx) {
  const pid = ctx.pluginId;

  // iframe 页面
  app.get("/page", (c) => c.html(renderShell(c, ctx, "page")));

  // 挂件
  app.get("/widget", (c) => c.html(renderShell(c, ctx, "widget")));

  // 静态资源（可选，如果不用 renderShell inline）
  // app.get("/assets/*", (c) => serveAsset(c, ctx));

  // 内部 API（供 iframe fetch 调用）
  app.get("/api/config", (c) => { ... });
  app.post("/api/config", async (c) => { ... });
  app.get("/api/some-data", async (c) => { ... });
  app.post("/api/some-action", async (c) => { ... });
}
```

---
