# 常见踩坑记录

> 来自社区实战经验，整理自多个 Hana 插件开发过程。每条都真实踩过。
> 最后更新：2026-07-16

## 🔥 新人最常踩的 5 个坑

1. **[坑 41](#坑-41hana-webview-限制客户端-dom-构建)**：webview 限制客户端 DOM 构建 — 复杂页面走服务端渲染
2. **[坑 4](#坑-4页面-200-不代表插件加载成功)**：页面 200 不代表加载成功 — iframe 需要发 `hana.ready` 握手
3. **[坑 6](#坑-6外部-jsapi-不会继承-authorization-header)**：外部 JS 不带 token → 403 — 子资源需手动透传 token
4. **[坑 5](#坑-5内联脚本可能不执行)**：内联脚本不执行 — 改用外部 JS 文件
5. **[坑 24](#坑-24manifestjson-多余字段导致插件无法加载)**：manifest.json 多余字段导致插件不可见

---

## 一、插件安装与加载

### 坑 1：dev 插件槽 ≠ 正式安装

Hana 的 dev 插件槽适合临时调试，但不是正式安装。

**典型现象**：
- dev 模式下页面能出现
- 重启 HanaAgent 后插件入口消失
- 设置里的正式插件列表看不到

**原因**：dev slot 只是调试加载记录，重启后可能不保留。

**正确做法**：
- 调试用 `plugin_dev_install`
- 交付或长期使用要放进 `.hanako/plugins` 或走正式本地安装流程
- 最好同时打一个 zip，方便重新安装和回滚

### 坑 2：正式插件必须自包含

dev 插件能跑，不代表正式插件能跑。

**典型现象**：route 里引用了 Hana 源码内部文件，dev 环境路径刚好成立，复制到正式目录后路径断掉。

**正确做法**：
- 插件不要 import Hana 仓库内部源码
- 需要的小函数可以复制成插件内部 helper
- 插件只依赖公开 SDK、标准 Node API、manifest 声明能力、公开 bus/API

### 坑 3：源码改动不会自动进入安装版 Hana

改 core 源码只影响源码工作区，当前运行的 HanaAgent 安装版不会自动吃。

**正确做法**：插件修复可以热重载（`plugin_dev_reload`），core 能力变更不能靠复制插件解决。

---

## 二、前端加载与鉴权

### 坑 4：页面 200 不代表插件加载成功

Hana 插件页面是 iframe，宿主会等 iframe 发 ready 握手。

**典型现象**：`/page` HTTP 返回 200，但 Hana 顶部插件入口仍显示"加载失败"。

**原因**：iframe 没有发送 `hana.ready`，或前端脚本没执行。

**最小 ready**：
```js
window.parent.postMessage({
  protocol: 'hana.plugin.ui',
  version: 1,
  kind: 'event',
  type: 'hana.ready',
}, "*");
```

### 坑 5：内联脚本可能不执行

页面能打开但按钮没反应，问题可能在前端脚本没有运行。

**正确做法**：尽量把脚本拆成外部资源（`/assets/panel.js`），避免 Electron/CSP/iframe 环境拦截内联脚本。

### 坑 6：外部 JS/API 不会继承 Authorization header

这是最关键的坑之一。

Hana 宿主通过 `hanaUrl()` 打开 iframe 页面时，会给页面 URL 加 token query。但页面里的外部脚本不会自动继承。

**结果链路**：
```
/page 带 token → 200
/assets/panel.js 不带 token → 403
脚本不执行 → hana.ready 不发送 → 宿主显示加载失败
```

**修法**：页面生成脚本 URL 时透传 token，客户端 API 请求也要带 token。

### 坑 7：Vite/React split chunk 会把 token-query 鉴权打穿

现代前端构建会自动生成很多入口之外的资源请求（Vite、React.lazy()、dynamic import()、cssCodeSplit、modulepreload）。

**结果链路**：
```
page HTML 200
entry JS 200
split chunk 403
lazy CSS 403
React.lazy import reject → 页面白屏
```

**临时规避**：
- 关闭拆包：`build.rollupOptions.output.inlineDynamicImports = true`
- 关闭 CSS 拆分：`build.cssCodeSplit = false`
- 避免 React.lazy / dynamic import
- 构建成单 JS + 单 CSS，在 route HTML 中给二者显式拼 token

---

## 三、代码与构建

### 坑 8：服务端模板字符串会吞客户端 JS

服务端模板字符串生成客户端 JS 时，反斜杠和 `${}` 会被服务端解析。

**修法**：不用正则，改成普通字符串裁剪；客户端代码独立成文件或独立函数。

### 坑 9：插件 route 模块会被运行中进程缓存

复制文件到正式插件目录后，当前 Hana 进程不一定立刻使用新代码。

**处理**：禁用再启用插件，或调用 `plugin_dev_reload`。

### 坑 10：zip 要跟着正式目录同步

修了正式目录后，如果 zip 没更新，下次重新安装又会把旧 bug 装回来。

**正确做法**：每次修正式插件目录后，重新打包 zip；bump manifest version。

---

## 四、权限与能力边界

### 坑 11：权限边界要记牢

| 形态 | 权限 | 能做什么 | 不能做什么 |
|------|------|---------|-----------|
| Tool-only | restricted | tools、skills、commands、agents、ctx.config、ctx.dataDir、bus.emit/subscribe/request | bus.handle、routes、extensions、providers、动态注册工具、lifecycle |
| UI/Runtime | full-access | 以上全部 + bus.handle、routes、extensions、providers、ctx.registerTool、onload/onunload | 不能碰 core 内部状态 |

### 坑 12：full-access 也不该乱碰 core 内部状态

full-access 不等于可以随意写内部状态。涉及 live session、频道调度、上下文压缩，最好走 core bus。没有稳定接口时，宁可按钮禁用，也不要做危险写入。

### 坑 13：EventBus 是插件集成的关键入口，但能力分层

"插件有 EventBus"不等于"core 的所有内部能力都开放了"。需求提给作者时，要提具体 bus topic 和契约。

### 坑 14：调 LLM 的正确姿势

**永远走 `ctx.bus.request("utility:call-text", ...)`**，不要：
- ❌ 声明 `ui.hostCapabilities: ["llm.generate"]`（host 不识别，被 ignored）
- ❌ 自己 fetch 外部 LLM API（绕路且要自己管 key）

```js
const result = await ctx.bus.request("utility:call-text", {
  messages: [{ role: "user", content: "..." }],
  temperature: 0.4,
  maxTokens: 800,
  operation: "my-plugin-id",
}, { timeoutMs: 60_000 });
```

---

## 五、调试与排障

### 坑 15：HTTP 分层定位很重要

遇到"插件加载失败"不要只猜，要分层测：

```
/page 是否 200
/assets/panel.js 是否 200
panel.js 是否语法正确
/api/status 是否 200
HTML 里 script src 是否带 token
JS 是否发送 hana.ready
```

### 坑 16：插件文件输出必须走 SessionFile

工具生成文件时要用 `toolCtx.stageFile()`，不要手写 `file://`、本地路径或伪协议。

### 坑 17：判断插件方案的四问

1. 这是 Agent 自动调用的能力，还是用户点界面使用的能力
2. 是否需要长期后台运行或监听事件
3. 是否需要 HTTP route、iframe UI、provider、extension
4. 是否需要 core 内部能力，如果需要，是否已有稳定 bus/API

---

---

## 六、我们自己的踩坑（闲不住/弹幕/补给站等）

### 坑 18：重复 import 导致路由静默失败

同一个文件里有两行 `import { join } from 'node:path'`，Hana 的 ES module 加载会报错。

**典型现象**：日志中有 `[ERROR] [plugin-manager] route "xxx.js" failed to load: Identifier 'join' has already been declared`，但插件在 Hana 界面可能显示为 loaded，因为部分路由加载成功了。

**正确做法**：每次改完代码，检查是否有重复 import；用 `plugin_dev_diagnostics` 确认路由状态。

### 坑 19：Python 子进程锁住 dev 插件目录

插件包含 Python 子进程时，Python 进程会锁住 `plugins-dev/插件名/` 目录。

**典型现象**：dev 插件无法卸载、无法重装、目录删除失败。

**排查**：`Get-Process -Name python | Where-Object { $_.CommandLine -match "插件名" }`
**解决**：先 `Stop-Process -Id xxx -Force` 终止进程，再删目录重装。

### 坑 20：Hana 页面系统可能丢弃内联 `<script>`

在 `routes/page.js` 中返回的 HTML 包含内联 `<script>` 标签，但 Hana 的 webview 可能 strip 掉这些标签。

**典型现象**：页面 HTML 加载了，但所有 JS 交互都不生效，就像 JS 没执行一样。

**正确做法**：改用外部脚本文件 `<script src="api/app.js">` 加载，避免内联。

### 坑 21：fetch 请求默认无超时

前端 fetch 默认没有超时机制，如果服务器不响应，请求会一直挂起。

**典型现象**：页面显示"正在加载..."但永远不消失，用户不知道是请求挂了还是数据量大。

**正确做法**：所有 fetch 必须加 `AbortSignal.timeout(5000)`。

### 坑 22：`$('#app')` 在 render 函数中可能返回 null

在客户端 JS 中使用 jQuery 选择器 `$('#app')`，在某些作用域中可能返回 null，导致后续 `.innerHTML` 调用失败。

**典型现象**：render() 不报错但页面不更新，因为早期 `if (!app) return` 直接返回了。

**正确做法**：改用 `document.getElementById('app')` 直接查找，更可靠。

### 坑 23：前端调试不能用 DevTools — 分步调试法

Hana 插件页面用 webview 渲染，F12 DevTools 不开放给插件内部。

**正确做法**：分步调试法，在关键节点设置可见的 DOM 文本标记：
```javascript
document.getElementById('app').textContent = '⚙ 脚本已加载';
// ... 初始化 ...
document.getElementById('app').textContent = '请求: ' + url;
// ... fetch ...
document.getElementById('app').textContent = '状态: ' + resp.status;
// ... 渲染 ...
document.getElementById('app').textContent = '渲染中...';
```
用户报告的文本变化可以精确告诉你是哪一步失败了。

### 坑 24：manifest.json 多余字段导致插件无法加载

manifest.json 中包含了不被当前版本支持的字段，插件管理器解析失败。

**典型现象**：插件在列表中不显示，或加载时报错但原因不明。

**正确做法**：只保留 manifest.json 的必要字段，不确定的字段先删掉试试；参考 [07-plugin-structure.md](07-plugin-structure.md)。

### 坑 25：webview 隔离导致前端 fetch 跨域失败

Hana 的 webview 隔离机制可能导致前端 fetch 请求被拦截或跨域。

**典型现象**：前端 fetch 全部失败，但后端 API 单独测试正常。

**正确做法**：
- 数据通过服务端渲染直接嵌入 HTML，前端不走 fetch
- 或改用后端驱动的方案，前端只负责展示

### 坑 26：社区版和 dev 版共存时的 shadow 关系

dev 插件会 shadow（覆盖）同 id 的社区版插件，但 shadow 不是绝对的。

**典型现象**：安装 dev 版后，功能表现不一致，不知道用的是哪个版本。

**排查**：用 `plugin_dev_diagnostics` 检查 `shadowedBy` 字段和 `devSlots` 数组。

---

## 七、更多实战经验（来自闲不住/弹幕/花酿等）

### 坑 27：CSS unicode 转义语法混淆

CSS 中 `\2728` 是正确写法，`\u2728` 是 JavaScript 写法。在 CSS content 属性中用错了会导致字符显示为乱码。

**正确做法**：CSS 中直接写实际字符（如 `'✨'`），不用转义。

### 坑 28：整体替换配置导致动态数据丢失

用新对象整体替换配置时，旧的动态数据（用户设置、状态值）丢失。

**正确做法**：替换前备份旧值，替换后恢复：`if (old.userData) new.userData = old.userData`。

### 坑 29：数据结构变更未兼容旧数据

新增字段或修改结构后，已有用户的数据没有迁移，导致功能异常。

**正确做法**：数据加载时检测旧格式并自动迁移。每次数据格式变更写迁移逻辑。

### 坑 30：CSS 在 `<head>` 中可能被丢弃

Hana 的 page 系统可能只渲染 `<body>` 内的内容，丢弃 `<head>` 中的 `<style>`。

**解决**：用 JS 动态注入关键 CSS，或把 `<style>` 放在 `<body>` 内。

### 坑 31：`window.location.href` 跳转导致页面刷新

在插件页面中用 `window.location.href` 跳转会刷新页面，中断交互体验（弹窗关闭、滚动位置丢失）。

**正确做法**：改用 fetch，API 端返回 JSON 而非 redirect。

### 坑 32：模块顶层代码执行导致插件加载失败

模块顶层代码出错（文件不存在、依赖缺失）会导致整个插件加载中断，无明显错误提示。

**正确做法**：所有可能出错的操作放进函数内，用 try-catch 包裹，仅在 onload 或实际调用时才执行。

### 坑 33：错误被静默吞噬

功能不生效但没有任何错误日志。try-catch 只捕获不记录，Promise 没有 catch。

**正确做法**：每个 catch 块都要打日志。异步函数正确处理错误，关键路径加状态标记。

### 坑 34：用户配置被插件更新覆盖

配置文件放在插件目录下，更新时被覆盖。

**正确做法**：用户配置必须存在 `~/.hanako/data/插件名/` 目录下，与插件代码分离。`.gitignore` 排除数据文件。

### 坑 35：`new URL(c.req.url)` 在路由中崩溃

`c.req.url` 在 Hana 路由中可能是相对路径，`new URL()` 要求绝对 URL。

**正确做法**：用 `c.req.query('key')` 解析参数，不要自己解析 URL。

### 坑 36：外部子进程配置不同步

Hana 插件 spawn 了外部进程，两边的配置文件独立，Hana 侧改了配置但外部进程感知不到。

**解决**：Hana 侧加代理路由，保存配置时通过 HTTP 转发给外部进程；外部进程提供 `/config` 接口支持运行时更新。

### 坑 37：不要往会话 JSONL 写 `role: "system"`

Hana 的聊天 UI 解析器只认 `role: "user"` 和 `role: "assistant"`。写入 `role: "system"` 会导致 UI 崩溃。

**教训**：往会话文件写消息前，先查 Hana 知识库里消息格式的过滤逻辑。

### 坑 38：异步调用不等前一步完成

`session:abort` 是异步的，不等它完成就调用 `session:send`，会因 session 仍处于 streaming 状态而触发 `session_busy` 错误。

**教训**：有严格先后顺序的异步调用，前一步必须 `await`。

### 坑 39：数据写入在消息注入之前

如果先触发助手回复、再写数据，助手可能在回复中调用工具查数据时发现不存在。

**教训**：任何会被助手回复中调用的工具所读取的数据，必须先写入再触发回复。时序是：create → save → trigger。

### 坑 40：BOM 头破坏 JSONL

UTF-8 BOM（`\uFEFF`）出现在 JSONL 首行会导致 JSON.parse 失败。PowerShell 和部分编辑器默认会加 BOM。

**教训**：处理 JSONL 时先检查并去除 BOM。写入时显式指定 `encoding='utf-8'`（无 BOM）。

### 坑 41：Hana webview 限制客户端 DOM 构建

插件页面在某些条件下可以返回 HTML，但 JS 对 DOM 的修改可能被 webview 静默拦截。

**典型现象**：
- 页面返回 "200"，HTML 壳里有 "加载中..."
- 内联 `<script>` 的简单测试能执行并修改 DOM（如设 `textContent`）
- 但同样代码放在嵌套 IIFE、异步回调、或被 `src` 属性加载时，DOM 修改**不生效**
- `innerHTML = x`（设值）不生效，但 `innerHTML += x`（追加）生效
- `createElement('script')` 动态加载的脚本能执行 JS 但 DOM 操作被拦截

**根因**：Hana 插件页面的 webview/iframe 环境对不同加载方式的 JS 做差异化处理：内联脚本的同步 DOM 操作最可靠；外部脚本、动态脚本、异步回调的 DOM 修改容易被拦截。具体拦截边界取决于 webview 的 CSP 和执行上下文隔离策略。

**正确做法**：
1. **服务端渲染完整 HTML**。在 `routes/api.js` 的 `/page` 路由中用字符串拼接生成完整的设置面板 HTML（含 `<style>`、所有卡片、表单元素），然后一次性返回
2. 客户端 `app.js` 只负责事件绑定、API 调用、toast 提示，**不碰 `innerHTML = ...`** 来构建 UI
3. 把 app.js 内容通过 `readFileSync` 读取后内联到 HTML 中（`<script>${appJs}</script>`），不要用 `<script src>` 或 `createElement('script')` 动态加载
4. CSS 放在 `<head>` 的 `<style>` 标签中，不要通过 JS 注入
5. 页面加载完成后发送 `hana.ready` 握手信号：
   ```js
   window.parent?.postMessage({protocol:'hana.plugin.ui',version:1,kind:'event',type:'hana.ready'},"*");
   ```

**排查步骤**：
1. IIFE 第一行加 `document.title = 'OK'` 确认 JS 被加载
2. 用 `document.getElementById('app').textContent = 'STEP1'` 确认 DOM 操作生效
3. 在嵌套函数/异步回调中分别加诊断标记，逐层定位失效点
4. 如果复杂 DOM 操作一律失败，放弃客户端渲染，走服务端渲染

**实战案例**：弹幕插件「在干嘛」（zaiganma）的设置页面。反复试过内联脚本、外部脚本、动态脚本、异步 XHR → fetch 替换，都不稳定。最终在 route 层构建完整 HTML，客户端只做事件绑定，一次成功。

**关联**：坑 5（内联脚本可能不执行）、坑 30（CSS 在 `<head>` 中可能被丢弃）

---

## 快速排障清单

插件入口加载失败时按顺序查：

1. 插件是否在正式目录或正式安装记录里
2. `manifest.json` 是否能被扫描
3. route 是否自包含，没有 import Hana 内部源码
4. `/page` 是否 200
5. `/assets/panel.js` 是否 200，是否带 token
6. `panel.js` 是否语法正确
7. 如果是 Vite/React，split chunk、modulepreload、lazy CSS 是否也都是 200
8. 脚本是否发送 `hana.ready`
9. 当前进程是否还缓存旧 route
10. zip 是否同步更新
11. 这个需求到底属于插件能力，还是需要 core bus

**LLM 调用排障追加**：
12. manifest 是否声明了 `ui.hostCapabilities: ["llm.generate"]`？删掉
13. 插件后端路由是否调 `ctx.bus.request("utility:call-text", ...)`？
14. 调 LLM 返回空？检查宿主 agent 的 utility 模型是否配置
15. 前端看不到 LLM 返回文本？检查字段名是否一致（`data.ok` 而不是 `data.success`）