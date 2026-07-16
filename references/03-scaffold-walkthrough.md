# 插件脚手架 Walkthrough
> 从零到跑：用 Phase 1 模板做一个"能加载"的 iframe 页面插件
> 对应：plugin-api-reference.md + plugin-patterns.md + plugin-best-practices.md
> 最后更新：2026-06-27（0.345.3 适配：WebView/chat.surface 卡片 / Plugin SDK / hostCapabilities / request context / 资源监听）

---

## 目标

做一个**最小但合格**的 Hana 插件：
- 有一个 iframe 页面（`/page`）
- 页面发送 ready 握手
- 外部脚本带 token 不 403
- 有一个工具（`tools/hello.js`）

**不包含**：构建工具、React、Tailwind、复杂 UI。先跑通原生 JS 轻量模式。

---

## 插件结构全景图

```
my-first-plugin/
├── manifest.json          # 插件身份证——声明 ID、版本、权限、页面
├── routes/
│   └── ui.js             # 服务端入口——注册 iframe 页面路由
├── tools/
│   └── hello.js           # Agent 工具——可以被对话直接调用
├── assets/
│   └── panel.js           # 前端脚本——验证 token 透传（可选）
└── skills/
    └── my-first-plugin/
        └── SKILL.md       # Agent 知识——教 Agent 怎么用这个插件
```

**每个文件的作用**：
| 文件 | 生命周期角色 | 运行环境 |
|------|-------------|---------|
| `manifest.json` | 插件加载时读取，决定插件有没有页面、工具、权限 | 服务端启动时 |
| `routes/ui.js` | 用户访问 `/page` 时执行，返回 HTML | 服务端（Node） |
| `tools/hello.js` | 对话中调用工具时执行，返回结果 | 服务端（Node） |
| `assets/panel.js` | iframe 加载时执行，操作 DOM | 浏览器端 |
| `skills/SKILL.md` | Agent 读取，学习如何调用工具 | Agent 上下文 |

**精简原则**：
- 不需要的目录**直接删掉**，不要留空目录。
- 纯工具插件最小只需要 `manifest.json`（或不用）+ `tools/hello.js`。
- 有 iframe 页面再加 `routes/ui.js` + `assets/`。
- `skills/`、`commands/`、`agents/` 按需添加。

**何时触发系统2（慢思考）**：
- 涉及 iframe 页面 + token 透传 → 必须走系统2
- 涉及 `toolCtx.dataDir` 文件写入 → 必须走系统2
- 涉及多个工具同时注册 → 必须走系统2
- 涉及权限变更（restricted ↔ full-access）→ 必须走系统2
- 其他情况走系统1（快速匹配模板）

---

## 前置条件

- HanaAgent 已安装并运行
- 知道当前 server 端口（默认 `14500`，可在 `C:\Users\Administrator\.hanako\server-info.json` 查看 `port` 字段）
- 有文本编辑器和终端

---

## 步骤 1：创建插件目录

**操作**：在工作目录创建以下结构

```bash
# Windows PowerShell
New-Item -ItemType Directory -Force -Path `
  "W:/Games/Hanako/Work/my-first-plugin/routes", `
  "W:/Games/Hanako/Work/my-first-plugin/tools", `
  "W:/Games/Hanako/Work/my-first-plugin/assets", `
  "W:/Games/Hanako/Work/my-first-plugin/skills/my-first-plugin"
```

**预期结果**：
```text
W:\Games\Hanako\Work\my-first-plugin\
├── routes\
├── tools\
├── assets\
└── skills\
    └── my-first-plugin\
```

**失败对照**：
- 如果报 `AccessDenied`：检查工作目录是否有写权限
- 如果目录没创建：确认路径拼写正确，使用正斜杠 `/` 或双反斜杠 `\\`

---

## 步骤 2：写 manifest.json

**文件**：`manifest.json`

```json
{
  "manifestVersion": 1,
  "id": "my-first-plugin",
  "name": "My First Plugin",
  "version": "0.1.0",
  "description": "Walkthrough 最小插件",
  "trust": "full-access",
  "contributes": {
    "page": {
      "title": "My Page",
      "route": "/page",
      "icon": "<svg viewBox=\"0 0 24 24\" fill=\"none\" stroke=\"currentColor\" stroke-width=\"1.5\"><circle cx=\"12\" cy=\"12\" r=\"10\"/></svg>"
    }
  },
  "interface": {
    "displayName": "My First Plugin",
    "shortDescription": "Walkthrough 最小插件示例",
    "category": "Productivity"
  }
}
```

**关键规则**：
| 字段 | 规则 |
|------|------|
| `id` | 必须和目录名完全一致（小写、连字符、无空格） |
| `trust` | 有 iframe 页面必须 `full-access`；纯工具插件可省略 |
| `contributes.page` | 声明 iframe 页面入口，`route` 是 URL 路径 |
| `contributes.widget` | 可选，声明挂件（本 walkthrough 不用） |
| `icon` | 内联 SVG 字符串，不能引用外部文件 |
| `capabilities` | 可选（0.341.19+），声明资源访问能力（见下方） |
| `ui.hostCapabilities` | 可选（0.345.3+），声明 iframe 需要的宿主能力（见下方） |

### capabilities 声明（0.341.19+）

如果你的插件需要访问用户资源（本地文件、SessionFile 等），在 manifest.json 中添加 `capabilities`：

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

**关于 surfaces**：
- 本 walkthrough 用 `contributes.page`，**不是** `surfaces.pages`。
- 如果插件只有工具，不需要写 `contributes`，也不需要 `full-access`。

**关于 ResourceIO（0.341.19+）**：
- 插件私有数据（`ctx.dataDir`）仍然可用，无需声明能力
- 访问用户资源（本地文件、SessionFile）需要使用 `ctx.resources` 统一 API
- 新增资源监听辅助方法 `ctx.resources.watch()` / `ctx.resources.subscribe()`（0.345.3+）
- 详见 `plugin-api-reference.md` 中的 ResourceIO 章节

**写入要求**：
- **必须用无 BOM 的 UTF-8 编码**。Windows PowerShell 的 `Set-Content -Encoding UTF8` 会带 BOM（`EF BB BF`），导致 JSON parser 报错。
- 正确写法：
  ```powershell
  [System.IO.File]::WriteAllText(
    "W:/Games/Hanako/Work/my-first-plugin/manifest.json",
    '{"manifestVersion":1,"id":"my-first-plugin","name":"My First Plugin","version":"0.1.0","description":"Walkthrough 最小插件","trust":"full-access","contributes":{"page":{"title":"My Page","route":"/page","icon":"<svg viewBox=\"0 0 24 24\" fill=\"none\" stroke=\"currentColor\" stroke-width=\"1.5\"><circle cx=\"12\" cy=\"12\" r=\"10\"/></svg>"}},"interface":{"displayName":"My First Plugin","shortDescription":"Walkthrough 最小插件示例","category":"Productivity"}}',
    [System.Text.UTF8Encoding]::new($false)
  )
  ```

**检查点**：
- [ ] 文件存在且内容无 BOM（用编辑器打开第一行应是 `{`，不是乱码）
- [ ] `id` = `my-first-plugin`
- [ ] `contributes.page.route` = `/page`

---

## 步骤 3：写 routes/ui.js

**文件**：`routes/ui.js`

```js
export default function (app, ctx) {
  app.get("/page", (c) => {
    const token = c.req.query("token") || "";
    const base = `/api/plugins/${ctx.pluginId}`;
    const html = `<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>My First Plugin</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      color: #e0e0e0;
      background: #1a1a2e;
    }
    #app { min-height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; }
    .box { padding: 2rem; border: 1px solid #333; border-radius: 8px; }
  </style>
</head>
<body>
  <div id="app">
    <div class="box">
      <h1>Hello from iframe</h1>
      <p>token: ${token ? "✅ received" : "❌ missing"}</p>
    </div>
  </div>

  <script>window.HANA_TOKEN=${JSON.stringify(token)};window.HANA_PLUGIN_BASE=${JSON.stringify(base)};</script>
  <script src="${base}/assets/panel.js?token=${encodeURIComponent(token)}"><\/script>
  <script>
    window.parent.postMessage({ type: "ready" }, "*");
    window.parent.postMessage({
      protocol: "hana.plugin.ui",
      version: 1,
      kind: "event",
      type: "hana.ready"
    }, "*");

    function authQuery() {
      const params = new URLSearchParams(window.location.search);
      const t = params.get("token");
      return t ? "?token=" + encodeURIComponent(t) : "";
    }
    function apiUrl(path) {
      return "/api/plugins/my-first-plugin" + path + authQuery();
    }
    async function pluginApi(path, options) {
      const res = await fetch(apiUrl(path), {
        headers: { "Content-Type": "application/json" },
        ...options
      });
      if (!res.ok) {
        const text = await res.text();
        throw new Error("API " + path + " " + res.status + ": " + text);
      }
      return res.json();
    }
  </script>
</body>
</html>`;
    return c.html(html);
  });
}
```

**关键规则**：
| 规则 | 说明 |
|------|------|
| `c.req.query("token")` | `c.req.query` 是函数，不是 URLSearchParams |
| `c.html(html)` | 只传 HTML 字符串，不要传状态码 |
| `<\\/script>` | 反斜杠防止服务端提前闭合 `<script>` 标签 |
| `ctx.pluginId` | 动态获取插件 ID，不要硬编码 `/my-first-plugin` |
| `window.HANA_TOKEN` | 注入 token，供前端 fetch 使用 |
| `window.HANA_PLUGIN_BASE` | 注入 base 路径，供前端拼接 API URL |

**检查点**：
- [ ] `app.get("/page")` 返回 HTML
- [ ] HTML 中 `<script src>` 带 `?token=` 参数
- [ ] HTML 内嵌 ready 双发脚本
- [ ] `authQuery()` 和 `apiUrl()` 已定义
- [ ] `c.html()` 只传 HTML 字符串

**失败对照**：
| 现象 | 原因 | 修法 |
|------|------|------|
| 页面白屏，body 只有 `200` | `c.html(200, html)` 参数顺序写反 | 改成 `c.html(html)` |
| token 显示 `❌ missing` | 用了 `new URLSearchParams(c.req.query)` | 改成 `c.req.query("token")` |
| HTML 里 JS 报错 | `<\\/script>` 少了反斜杠 | 补上反斜杠 |
| 前端 fetch 403 | `window.HANA_TOKEN` 没注入 | 确认 `window.HANA_TOKEN=${JSON.stringify(token)}` |

---

## 步骤 4：写 tools/hello.js

**文件**：`tools/hello.js`

```js
export const name = "hello";
export const description = "Say hello to someone";

export const parameters = {
  type: "object",
  properties: {
    name: {
      type: "string",
      description: "要打招呼的人名"
    }
  },
  required: ["name"]
};

export async function execute(input, toolCtx) {
  toolCtx.log.info("hello", { name: input.name });
  return `Hello, ${input.name}! 来自 my-first-plugin。`;
}
```

**关键规则**：
- `name`：简短、无空格、全小写，将成为工具调用名 `my-first-plugin_hello`
- `parameters`：合法 JSON Schema，`required` 数组列出必填字段
- `execute`：返回字符串或对象，结果会传给 Agent
- `sessionPermission`：可选（0.341.19+），声明工具权限行为（见下方）

### sessionPermission 声明（0.341.19+）

工具现在需要声明 `sessionPermission`，让 Hana 的 session 权限模式能做精确判断：

```js
// 只读工具
export const sessionPermission = { readOnly: true };

// 插件输出文件
export const sessionPermission = { kind: "plugin_output" };

// SessionFile 输出（媒体交付）
export const sessionPermission = { kind: "session_file_output" };
```

### ui.hostCapabilities 声明（0.345.3+）

如果插件的 iframe 需要调用宿主能力（打开外部链接、剪贴板、资源操作等），在 manifest.json 中声明：

```json
{
  "ui": {
    "hostCapabilities": ["external.open", "clipboard.writeText", "resource.open"]
  }
}
```

> 详见 `plugin-patterns.md` 模式 18：Plugin SDK 宿主通信

### 可视化卡片（0.345.3+）

工具返回值中可以声明 `details.card` 来在聊天中渲染卡片：

```js
// WebView 卡片
c return {
  content: [{ type: "text", text: "数据摘要" }],
  details: {
    card: {
      type: "webview",
      route: "/card/chart?symbol=sh600519",
      title: "股票图表",
      description: "贵州茅台 日K"
    }
  }
};

// 原生聊天 surface 卡片
c import { createChatSurfaceCard, createSession } from "@hana/plugin-runtime";
c const child = await createSession(ctx, {
  kind: "tavern-run",
  visibility: "plugin_private",
  cwd: ctx.dataDir
});
c return {
  content: [{ type: "text", text: "已创建会话" }],
  details: {
    card: createChatSurfaceCard(ctx, child.child, {
      title: "Tavern run",
      description: "插件私有会话 transcript"
    })
  }
};
```

**检查点**：
- [ ] `name` 简短、无空格
- [ ] `parameters` 是合法的 JSON Schema
- [ ] `execute` 返回字符串或对象

**失败对照**：
| 现象 | 原因 | 修法 |
|------|------|------|
| 工具不可见 | `tools/` 目录没被扫描 | 检查目录结构，重启 Hana |
| 工具调用报参数错误 | `parameters` 格式不对 | 对照 JSON Schema 规范检查 |

---

## 步骤 5：写 assets/panel.js（可选外部脚本）

**文件**：`assets/panel.js`

```js
console.log("[my-first-plugin] panel.js loaded, token auth works");

window.addEventListener("DOMContentLoaded", () => {
  const app = document.getElementById("app");
  if (app) {
    const info = document.createElement("p");
    info.style.marginTop = "1rem";
    info.style.fontSize = "0.75rem";
    info.style.color = "#6C5CE7";
    info.textContent = "外部脚本执行成功 ✅ | my-first-plugin";
    app.querySelector(".box").appendChild(info);
  }
});
```

**用途**：验证 token 透传是否正常工作。如果 HTML 中 `<script src>` 正确带了 token，这个文件会被执行。

**检查点**：
- [ ] 浏览器控制台能看到 `[my-first-plugin] panel.js loaded`
- [ ] 页面上出现"外部脚本执行成功 ✅"
- [ ] Network 面板中 `panel.js` 请求带 `?token=` 参数

**失败对照**：
| 现象 | 原因 | 修法 |
|------|------|------|
| 控制台无输出 | `panel.js` 加载失败 | 检查 Network 面板状态码 |
| 403 Forbidden | script src 没带 token | 确认 `<script src="...panel.js?token=...">` |

---

## 步骤 5.5：写 skills/my-first-plugin/SKILL.md（可选知识注入）

**文件**：`skills/my-first-plugin/SKILL.md`

```markdown
---
name: my-first-plugin
description: my-first-plugin — Walkthrough 最小插件示例
---

# My First Plugin

Walkthrough 最小插件示例。

## 工具清单

| 工具 | 用途 |
|------|------|
| `my-first-plugin_hello` | 向某人打招呼 |

## 使用示例

```
my-first-plugin_hello(name="月曦夜")
```
```

**关键规则**：
- frontmatter 的 `name` 必须和目录名一致
- 工具清单覆盖所有已实现工具
- 使用示例必须是可执行的调用格式
- **0.333.0+ skills label**：frontmatter `name` 现在在 Skills 管理面板中显示为 label，建议使用可读名称而非纯插件 ID（如 `my-first-plugin` → `我的第一个插件`）

---

## 步骤 6：安装到 dev slot

**操作**：在插件根目录执行

```bash
# 方式1：命令行（推荐）
plugin.dev.install --path "W:/Games/Hanako/Work/my-first-plugin"

# 方式2：HanaAgent 设置 → 插件 → 开发工具 → 安装
```

**预期结果**：返回 JSON，包含 `"ok": true`、`"status": "loaded"`

**失败对照**：
| 错误 | 原因 | 修法 |
|------|------|------|
| `Unexpected token '﻿'` | manifest.json 带 BOM | 用无 BOM UTF-8 重写文件 |
| `plugin source path does not exist` | 路径错误或目录被删 | 确认路径存在且拼写正确 |

**安装后验证**：
```bash
# 检查插件是否加载
# 在对话中直接问 Hana："列出已安装的插件"
# 或查看设置 → 插件 → 开发工具
```

**检查点**：
- [ ] 插件出现在已安装列表
- [ ] 顶部入口能看到 "My Page"

---

## 步骤 7：验证 iframe 加载

### 7.1 分层检查

```bash
# 1. 页面是否 200（带 token）
curl "http://localhost:14500/api/plugins/my-first-plugin/page?token=<YOUR_TOKEN>"

# 2. 外部脚本是否 200（关键！）
curl "http://localhost:14500/api/plugins/my-first-plugin/assets/panel.js?token=<YOUR_TOKEN>"

# 3. 工具是否可调用
# 在对话中直接调用工具：
# my-first-plugin_hello(name="月曦夜")
```

**获取 token**：
```powershell
# 读取 server token
$token = (Get-Content "C:\Users\Administrator\.hanako\server-info.json" | ConvertFrom-Json).token
# 使用
curl "http://localhost:14500/api/plugins/my-first-plugin/page?token=$token"
```

### 7.2 HTTP 分层定位表

| 层级 | 预期 | 如果失败 |
|------|------|---------|
| `/page` | 200 + HTML | 检查 route 文件是否被进程缓存（禁用再启用） |
| `/assets/panel.js` | 200 + JS | 检查 HTML 中 script src 是否带 token |
| 工具调用 | 返回 "Hello, xxx!" | 检查 tools 目录和 manifest |

### 7.3 浏览器检查

打开 Hana，进入插件页面：
- [ ] 页面显示 "Hello from iframe"
- [ ] 显示 `token: ✅ received`
- [ ] 显示 "外部脚本执行成功 ✅"
- [ ] 浏览器控制台无 403 错误
- [ ] 没有等待 7 秒才显示内容（ready 及时发送）

---

## 步骤 8：发布到正式目录

dev slot 验证通过后：

```bash
# 1. 复制到正式插件目录
Copy-Item -Recurse "W:/Games/Hanako/Work/my-first-plugin" "C:\Users\Administrator\.hanako\plugins\my-first-plugin"

# 2. 打包 zip（保持同步）
Compress-Archive -Path "W:/Games/Hanako/Work/my-first-plugin/*" -DestinationPath "W:/Games/Hanako/Work/my-first-plugin-v0.1.0.zip"

# 3. 重启 Hana 或禁用再启用插件
```

**检查点**：
- [ ] 重启后插件入口仍在
- [ ] 工具仍可调用
- [ ] 页面仍可加载

### Runtime Packaging 规则（0.333.0+）

发布时需要区分**必须打包**和**可以忽略**的文件：

| 必须打包 | 可以忽略 |
|---------|---------|
| `manifest.json` | `node_modules/`（插件不应依赖 npm 包） |
| `routes/ui.js` | `.git/` |
| `tools/` 下所有 `.js` | `*.md`（开发文档） |
| `assets/` 下所有文件 | `*.bak` / `*.tmp` |
| `skills/` 下所有文件 | 临时文件 |

**0.333.0+ 变更**：
- 插件目录结构变更时（如新增 `assets/video.mp4`），`Copy-Item` 跨盘复制现在更稳定（`fix: support Windows cross-drive plugin SDK paths`）
- 打包前确保 `assets/` 下的视频文件大小合理（建议单文件 < 50MB）

| 现象 | 根本原因 | 修法 |
|------|---------|------|
| 页面白屏，body 只有 `200` | `c.html(200, html)` 参数顺序写反 | 改成 `c.html(html)` |
| manifest 安装时报 `Unexpected token '﻿'` | Windows 写文件带 BOM | 用 `[System.IO.File]::WriteAllText(..., Encoding.UTF8NoBOM)` |
| 页面报 `missing_credential` | 访问 URL 没带 `?token=` | 从 `server-info.json` 取 token 拼接 |
| token 显示 `❌ missing` | 用了 `new URLSearchParams(c.req.query)` | 改成 `c.req.query("token")` |
| 外部脚本 403 | `panel.js` 的 script src 没带 token | 确保 `<script src="...panel.js?token=...">` |
| 工具不可调用 | tools 目录没被扫描 | 检查插件目录结构，重启 Hana |
| 页面加载旧代码 | route 缓存 | 禁用再启用插件 |
| Copy-Item 跨盘失败 | 0.333.0 之前 | 0.333.0+ 已修复，确保用正斜杠 `/` |

## Token 机制说明

Hana 插件 iframe 的鉴权通过 query token 实现：
- 服务端 token 存储在 `C:\Users\Administrator\.hanako\server-info.json` 的 `token` 字段。
- 当 Hana host 打开插件页面时，会自动拼接 `?token=...` 到 iframe URL。
- 插件内部 API（如 `/assets/panel.js`）也需要这个 token，因此 HTML 里必须透传：`<script src="${base}/assets/panel.js?token=...">`。
- 如果你在外部浏览器直接测试，必须手动带上这个 token，否则会收到 `{"error":"forbidden","reason":"missing_credential"}`。

## 验收标准

这个 walkthrough 跑通后，你具备：

1. **一个能跑的 iframe 页面** — 知道 page route、HTML 骨架、ready 握手、token 透传的最小组合
2. **一个能调用的工具** — 知道 tools/*.js 的标准格式和命名空间
3. **排查能力** — 遇到白屏知道按 `/page` → `/assets/panel.js` → 工具 分层查
4. **扩展基础** — 基于 `plugin-page.html` 模板，可以替换 UI 内容、加 React/Vite、加更多工具

这是 Phase 1 的"合格线"：**生成的插件能加载、能交互、能排查。**
