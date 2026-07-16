# 插件常见报错修复

> 开发过程中遇到的各类错误及其解决方法，按错误类型分类。

---

## 一、manifest.json 相关

### 报错：`Unexpected token '﻿'` 或 `JSON parse error`

**现象**：安装插件时报 JSON 解析错误，首行出现乱码。

**原因**：Windows PowerShell 的 `Set-Content -Encoding UTF8` 写入 UTF-8 BOM（字节序标记 `EF BB BF`），JSON 解析器不接受。

**修复**：
```powershell
# ❌ 错误写法（带 BOM）
Set-Content -Path "manifest.json" -Value '{}' -Encoding UTF8

# ✅ 正确写法（无 BOM）
[System.IO.File]::WriteAllText(
  "manifest.json",
  '{"manifestVersion":1,"id":"my-plugin","name":"My Plugin","version":"0.1.0","description":"test","trust":"full-access"}',
  [System.Text.UTF8Encoding]::new($false)
)

# ✅ 或使用 edit/write 工具
```

**预防**：永远不用 PowerShell `Set-Content` 或 `Out-File` 写 JSON/MD 文件。

---

### 报错：`plugin source path does not exist`

**现象**：安装插件时提示路径不存在。

**原因**：路径拼写错误或目录结构不对。

**修复**：
```bash
# 确认目录存在
dir "W:/path/to/my-plugin/manifest.json"

# 确认目录结构
my-plugin/
├── manifest.json    ← 必须有
├── routes/
├── tools/
└── skills/
```

---

### 报错：工具不可见

**现象**：安装成功，但对话中调用不了工具。

**原因**：
1. `tools/` 目录没被扫描到
2. 工具函数导出格式不对

**修复**：
```js
// ❌ 错误：没有正确导出
function hello(input, ctx) { return "hi"; }

// ✅ 正确：方式1（默认导出）
export default async function hello(input, ctx) {
  return { ok: true, data: "hi" };
}

// ✅ 正确：方式2（命名导出）
export const name = "hello";
export const description = "Say hello";
export async function execute(input, ctx) {
  return "Hello!";
}
```

**排查**：
```bash
# 查看诊断日志
plugin_dev_diagnostics(pluginId="my-plugin")
```

---

## 二、页面加载相关

### 白屏，body 只有 `200`

**现象**：打开插件页面，看到纯文本 `200`，没有 HTML 内容。

**原因**：`c.html()` 参数顺序写反了。

**修复**：
```js
// ❌ 错误
c.html(200, htmlString)

// ✅ 正确
c.html(htmlString)
```

---

### token 显示 `❌ missing`

**现象**：页面加载成功，但 token 验证失败。

**原因**：`c.req.query` 被当成了 URLSearchParams 对象。

**修复**：
```js
// ❌ 错误：c.req.query 是函数，不是对象
const params = new URLSearchParams(c.req.query);
const token = params.get("token");

// ✅ 正确
const token = c.req.query("token") || "";
```

---

### 外部脚本 403 Forbidden

**现象**：`panel.js` 加载返回 403，浏览器控制台报错。

**原因**：`<script src>` 没带 token 参数。

**修复**：
```html
<!-- ❌ 错误 -->
<script src="/api/plugins/my-plugin/assets/panel.js"></script>

<!-- ✅ 正确 -->
<script src="${base}/assets/panel.js?token=${encodeURIComponent(token)}"></script>
```

---

### HTML 里 JS 报错

**现象**：页面能打开，但 JavaScript 执行出错。

**原因**：内嵌 `<script>` 标签中的 `<\/script>` 少了反斜杠，导致服务端提前闭合标签。

**修复**：
```js
// ❌ 错误：服务端提前闭合 <script>
`<script>console.log("hello")<\/script>`

// ✅ 正确：反斜杠转义
`<script>console.log("hello")<\/script>`
```

---

### 页面加载旧代码

**现象**：修改了代码但页面没变化。

**原因**：服务端缓存了旧的路由文件。

**修复**：
```bash
# 禁用再启用插件
plugin_dev_disable(pluginId="my-plugin")
plugin_dev_enable(pluginId="my-plugin", allowFullAccess=true)

# 或者直接热加载
plugin_dev_reload(pluginId="my-plugin")
```

---

## 三、Token 和鉴权相关

### 403 Forbidden — `missing_credential`

**现象**：直接访问插件 API 返回 `{"error":"forbidden","reason":"missing_credential"}`。

**原因**：URL 没带 `?token=` 参数。

**修复**：
```bash
# 获取 token
$token = (Get-Content "C:\Users\Administrator\.hanako\server-info.json" | ConvertFrom-Json).token

# 访问时带上
curl "http://localhost:14500/api/plugins/my-plugin/page?token=$token"
```

---

### 403 Forbidden — `CAPABILITY_DENIED`

**现象**：iframe 调用宿主能力（如 `hana.external.open`）返回 `CAPABILITY_DENIED`。

**原因**：manifest.json 中没有声明对应的 `hostCapabilities`。

**修复**：
```json
{
  "ui": {
    "hostCapabilities": ["external.open", "clipboard.writeText", "resource.open"]
  }
}
```

> `toast.show` 不需要声明，可以直接用。

---

## 四、HTTP 请求相关

### 400 Bad Request — `missing_query` / `bad_format`

**现象**：API 返回 400，提示参数缺失或格式错误。

**原因**：前端传参不符合后端期望。

**修复**：
```js
// 后端检查参数
app.get("/api/search", (c) => {
  const q = c.req.query("q") || "";
  if (!q) {
    return c.json({ ok: false, code: "missing_query" }, 400);
  }
  // ...
});

// 前端确保传参
fetch("/api/search?q=test")
```

---

### 401 Unauthorized — `auth`

**现象**：API 返回 401，认证失败。

**原因**：token 错误或过期。

**修复**：检查 `server-info.json` 中的 token 是否与请求中的一致。

---

### 403 Forbidden — `forbidden`

**现象**：API 返回 403，无权限。

**原因**：token 没带在 URL 上。

**修复**：确保请求 URL 带 `?token=xxx`。

---

### 404 Not Found — `not_found`

**现象**：API 返回 404，资源不存在。

**原因**：ID 错误或路由未注册。

**修复**：
```js
// 检查路由是否正确注册
app.get("/api/items/:id", (c) => {
  const id = c.req.query("id");
  // ...
});
```

---

### 500 Internal Server Error — `tool_error`

**现象**：API 返回 500，工具内部错误。

**原因**：工具代码抛出未捕获的异常。

**修复**：
```js
// 加上 try-catch
export default async function my_tool(input, ctx) {
  try {
    // ... 业务逻辑
  } catch (err) {
    ctx.log?.error?.("tool error", { error: err.message, stack: err.stack });
    return { ok: false, code: "tool_error", message: err.message };
  }
}
```

---

### 502 Bad Gateway — `connection` / `upstream`

**现象**：API 返回 502，上游服务不可达。

**原因**：外部 API 地址错误或网络不通。

**修复**：
```js
// 检查 URL 是否正确
const res = await fetch("https://api.example.com/data");
if (!res.ok) {
  return c.json({ ok: false, code: "upstream", status: res.status }, 502);
}
```

---

### 504 Gateway Timeout — `timeout`

**现象**：API 返回 504，请求超时。

**原因**：外部服务响应慢或网络问题。

**修复**：
```js
// 添加超时控制
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 15000);

try {
  const res = await fetch(url, { signal: controller.signal });
  clearTimeout(timer);
  // ...
} catch (err) {
  clearTimeout(timer);
  if (err.name === "AbortError") {
    return c.json({ ok: false, code: "timeout" }, 504);
  }
}
```

---

## 五、文件系统相关

### 写入文件后内容为空

**原因**：路径来自用户输入，未检查是否在 `dataDir` 内。

**修复**：
```js
const safePath = path.resolve(dataDir, "safe", input.filename);
if (!safePath.startsWith(dataDir)) {
  return { ok: false, code: "path_traversal" };
}
```

---

### 跨盘路径失败

**现象**：`Copy-Item` 跨盘复制插件时报错。

**修复**：确保使用正斜杠 `/`，不要用反斜杠 `\`。

```powershell
# ❌ 错误
Copy-Item "my-plugin" "C:\Users\Administrator\.hanako\plugins\my-plugin"

# ✅ 正确
Copy-Item -Recurse "my-plugin" "C:/Users/Administrator/.hanako/plugins/my-plugin"
```

---

## 六、LLM 调用相关

### `ctx.model` 为 undefined

**现象**：路由中调用 `ctx.model.sample()` 报错 `Cannot read property 'sample' of undefined`。

**原因**：manifest.json 没有声明 `capabilities: ["model"]`。

**修复**：
```json
{
  "trust": "full-access",
  "capabilities": ["model"]
}
```

> 工具（`tools/*.js`）的 ctx 始终有 `ctx.model`，不需要声明。只有路由（`routes/ui.js`）需要声明。

---

### 模型返回空文本

**原因**：没有同时检查 `response.text` 和 `response.content` 两个字段。

**修复**：
```js
const response = await ctx.model.sample({ messages: [...] });
const text = response?.text || response?.content || "";
```

---

## 七、插件安装相关

### 工具不可见 — 安装后对话中调不了

**现象**：插件安装成功，但对话中找不到工具。

**可能原因和排查**：

| 原因 | 排查方法 |
|------|---------|
| 工具导出格式不对 | 确认 `export default async function` 或 `export const name` + `export async function execute` |
| `tools/` 目录没被扫描 | 确认目录名是 `tools` 而非 `tool` 或 `Tools` |
| 在 dev slot 但没启用 | 调用 `plugin_dev_enable(pluginId="xxx", allowFullAccess=true)` |
| 复制到了正式目录但没重启 | `Copy-Item -Recurse` 后需要重启 Hana 才能加载 |
| manifest.json 有 BOM | 用 `edit` 或 `write` 工具重写，不用 PowerShell |

---

### 安装后工具消失了

**现象**：之前可用的工具突然找不到了。

**原因**：插件目录被覆盖或 manifest.json 的 `id` 改了。

**修复**：
```bash
# 确认插件目录还在
dir "C:\Users\Administrator\.hanako\plugins\my-plugin\manifest.json"

# 确认 manifest.json 的 id 一致
# 如果改了 id，所有工具调用名也会变

# 重新安装
plugin_dev_reset(pluginId="my-plugin")
```

---

### dev slot 和正式目录的区别

| 维度 | dev slot | 正式目录 |
|------|---------|---------|
| 位置 | Hana 管理的隔离目录 | `~/.hanako/plugins/` |
| 修改后 | 热加载生效 | 需重启 Hana |
| 卸载 | `plugin_dev_uninstall` | 手动删除目录 |
| 诊断 | `plugin_dev_diagnostics` | 无专用命令 |
| 适用 | 开发调试 | 日常使用 |

> 开发时先用 dev slot 验证，确认没问题再复制到正式目录。

---

## 八、排查清单

（文件系统相关和 LLM 调用相关已在上方第五、六节覆盖，不再重复。）

---

## 八、排查清单

当遇到问题时，按以下顺序排查：

| 步骤 | 检查项 | 命令 |
|------|--------|------|
| 1 | 插件是否安装成功 | `plugin_dev_diagnostics(pluginId="xxx")` |
| 2 | 页面是否 200 | `curl "http://localhost:14500/api/plugins/xxx/page?token=xxx"` |
| 3 | 外部脚本是否 200 | 检查 Network 面板中 `panel.js` 的状态码 |
| 4 | token 是否传递 | 确认 `<script src>` 带 `?token=` |
| 5 | 工具是否可调用 | `plugin_dev_invoke_tool(pluginId="xxx", toolName="xxx", input={})` |
| 6 | 日志输出 | `plugin_dev_diagnostics(pluginId="xxx")` 查看 logs |
| 7 | 代码缓存 | 禁用再启用插件，或调用 `plugin_dev_reload` |
| 8 | dev slot 是否启用 | `plugin_dev_enable(pluginId="xxx", allowFullAccess=true)` |
