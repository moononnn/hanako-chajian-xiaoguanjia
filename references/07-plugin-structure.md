# 插件结构速查

> 快速查看不同插件类型需要的最小目录结构，复制即用。
> 最后更新：2026-06-27

---

## 最小结构速查表

| 类型 | 最小目录 | 说明 |
|------|---------|------|
| **Tool-only** | `manifest.json` + `tools/hello.js` | 仅工具，可无 manifest |
| **UI 页面** | `manifest.json` + `routes/ui.js` + `assets/panel.js` | 有 iframe 页面 |
| **UI + Widget** | 同上 + manifest 加 `contributes.widget` | 页面 + 小组件 |
| **Runtime** | `manifest.json` + `index.js` | lifecycle/EventBus |
| **Full（默认）** | 以上全部 | tool + UI + lifecycle |

---

## Tool-only 插件（最小）

```
my-tool-plugin/
├── tools/
│   └── hello.js          # 工具实现
```

**可选**：加 manifest 让插件更正式：
```
├── manifest.json          # 插件身份证
```

**manifest.json 最小示例**：
```json
{
  "manifestVersion": 1,
  "id": "my-tool-plugin",
  "name": "My Tool",
  "version": "0.1.0",
  "description": "A simple tool plugin",
  "trust": "restricted"
}
```

**工具导出方式**（二选一）：
```js
// 方式1：默认导出
export default async function hello(input, ctx) {
  return { ok: true, data: "Hello!" };
}
```
```js
// 方式2：命名导出
export const name = "hello";
export const description = "Say hello";
export async function execute(input, ctx) {
  return { ok: true, data: "Hello!" };
}
```

---

## UI 页面插件

```
my-ui-plugin/
├── manifest.json          # 声明 page 贡献
├── routes/
│   └── ui.js             # iframe 页面路由
├── assets/
│   ├── panel.js          # 前端脚本
│   └── panel.css         # 样式（可选）
└── skills/
    └── my-ui-plugin/
        └── SKILL.md       # Agent 知识
```

**manifest.json 关键配置**：
```json
{
  "manifestVersion": 1,
  "id": "my-ui-plugin",
  "name": "My UI Plugin",
  "version": "0.1.0",
  "description": "Plugin with iframe page",
  "trust": "full-access",
  "contributes": {
    "page": {
      "title": "My UI Plugin",
      "route": "/page",
      "icon": "<svg viewBox=\"0 0 24 24\" fill=\"none\" stroke=\"currentColor\"><path d=\"M5 5h14v14H5z\"/></svg>"
    }
  },
  "ui": {
    "hostCapabilities": ["external.open", "clipboard.writeText"]
  }
}
```

---

## Runtime 插件（lifecycle + EventBus）

```
my-runtime-plugin/
├── manifest.json          # 声明 full-access
├── index.js               # lifecycle 入口
└── tools/
    └── my-tool.js         # 关联工具（可选）
```

**index.js 示例**：
```js
const HANA_BUS_SKIP = Symbol.for("hana.event-bus.skip");

export default class Plugin {
  async onload() {
    const ctx = this.ctx;
    if (ctx.bus?.handle) {
      this.register(ctx.bus.handle("my-plugin:status", (payload) => {
        if (payload?.pluginId && payload.pluginId !== ctx.pluginId) return HANA_BUS_SKIP;
        return { ok: true, pluginId: ctx.pluginId };
      }));
    }
    ctx.log.info("my-runtime-plugin loaded");
  }

  async onunload() {
    this.ctx.log.info("my-runtime-plugin unloaded");
  }
}
```

---

## React 模板插件（带构建）

```
my-react-plugin/
├── manifest.json          # 声明 full-access
├── package.json           # 依赖和脚本
├── tsconfig.json          # TypeScript 配置
├── vite.config.ts         # Vite 构建配置
├── routes/
│   └── ui.js             # iframe shell
├── ui/
│   ├── Panel.tsx          # React 组件
│   └── panel.css          # 样式
└── tools/
    └── hello.js           # 工具
```

**package.json 最小示例**：
```json
{
  "name": "my-react-plugin",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "dependencies": {
    "@hana/plugin-sdk": "file:vendor/sdk/hana-plugin-sdk-*.tgz",
    "@hana/plugin-components": "file:vendor/sdk/hana-plugin-components-*.tgz",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^5.0.0",
    "@types/react": "^19.0.0",
    "typescript": "^5.0.0",
    "vite": "^7.0.0"
  },
  "scripts": {
    "build:ui": "vite build",
    "typecheck": "tsc --noEmit"
  }
}
```

**构建产物**：`vite build` 后输出到 `assets/panel.js` 和 `assets/panel.css`。

---

## SDK 包职责速查

| 包名 | 职责 | 何时需要 |
|------|------|---------|
| `@hana/plugin-runtime` | 运行时 API（toolCtx、EventBus、Task、defineTool、defineBusHandler） | 写工具、lifecycle、EventBus、后台任务 |
| `@hana/plugin-sdk` | iframe 通信、宿主能力调用（hana.ready、hana.external.open、hana.clipboard） | 写 iframe 前端 |
| `@hana/plugin-components` | React UI 组件（CardShell、Button、Switch、TextInput） | 用 React 构建 iframe UI |
| `@hana/plugin-protocol` | 插件协议类型定义（manifest 结构、工具参数 schema） | 类型检查、IDE 提示（开发时） |

> 如果用 `--template direct` 或无 React 构建，不需要 SDK 包。
> 如果用 `--template professional-react` 或 `guided-react`，需要 `plugin-sdk` + `plugin-components`。
> `plugin-runtime` 只在需要 `defineTool`/`defineBusHandler` 时用到（默认模板已包含）。

---

## 目录约定

| 目录 | 作用 | 是否必须 |
|------|------|---------|
| `manifest.json` | 插件身份声明 | 推荐有（非强制） |
| `tools/` | Agent 工具 | 有工具时必须有 |
| `routes/` | 页面/组件路由 | 有 iframe 时必须有 |
| `assets/` | 前端脚本/样式/静态资源 | 有 UI 时必须有 |
| `skills/` | Agent 知识文档 | 推荐有 |
| `index.js` | Lifecycle 入口 | 有 lifecycle 时必须有 |
| `vendor/sdk/` | 内置 SDK tarball | React 模板且 `sdk-mode bundled` 时必须有 |
| `scripts/` | 构建脚本 | 按需 |
| `commands/` | 命令插件 | 命令插件用 |
| `agents/` | 代理配置 | 插件自有代理用 |

**精简原则**：不需要的目录直接删掉，不要留空目录。

---

## 安装后验证清单

插件安装后，按以下清单快速验证：

| 检查项 | 命令/方法 | 预期 |
|--------|----------|------|
| 插件是否加载 | `plugin_dev_diagnostics(pluginId="xxx")` | status: "loaded" |
| 工具是否可见 | `plugin_dev_invoke_tool(pluginId="xxx", toolName="xxx", input={})` | 返回 `{ok: true}` |
| 页面是否 200 | 打开插件页面 URL | HTML 内容，非 200 纯文本 |
| 外部脚本是否 200 | 浏览器 Network 面板中 `panel.js` | 200，带 token 参数 |
| Token 是否传递 | `plugin_dev_diagnostics` 查看 token 状态 | 绿色 ✅ |
| 安装到正式 | `Copy-Item -Recurse` 到 `~/.hanako/plugins/` | 重启后生效 |
