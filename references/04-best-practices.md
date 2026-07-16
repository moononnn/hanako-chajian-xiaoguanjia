# 插件开发最佳实践
> 从踩坑和经验中提炼的规则、模式和反模式
> 对应：plugin-api-reference.md + plugin-patterns.md + plugin-scaffold-walkthrough.md
> 最后更新：2026-06-27

---

## 核心原则

### 1. 指令应该离执行点最近，离系统提示词最远

这是插件架构设计的核心原则。Claude Code 的七种指令传递方法（CLAUDE.md / Rules / Skills / Hooks / Output Styles / System Prompt Append）揭示了一个通用规律：

> **越靠近执行的指令越可靠，越远离执行的指令越容易丢失。**

应用到插件开发：

| 层级 | 载体 | 可靠性 | 例子 |
|------|------|--------|------|
| L0：系统提示词 | manifest + capabilities | 低——被截断就丢 | 插件 ID、版本号 |
| L1：Hook | lifecycle hooks | 高——平台保证触发 | 会话结束写日记 |
| L2：工具 | tools/*.js | 高——按需调用 | 搜索图片、生成语音 |
| L3：Skill | skills/*/SKILL.md | 中——Agent 知识 | "这个插件能做什么" |
| L4：记忆 | 用户记忆 | 低——换人丢失 | "我的审美偏好" |

**原则**：
- L0-L2 是**系统的事**，不依赖任何人的记忆
- L3 是**桥梁**，让 Agent 知道插件能干什么
- L4 是**个人的**，换了人就没了

### 2. 最小可用，渐进增强

不要一开始就搞全套。按这个顺序来：

```
Phase 1: 能跑通（manifest + 1 个工具）
    ↓
Phase 2: 能持久化（config/credentials/cache）
    ↓
Phase 3: 能联动（hooks + 事件）
    ↓
Phase 4: 能分发（打包 + 版本管理 + 文档）
```

### 3. 一个插件只做一件事

职责边界清晰的插件更容易：
- 测试和调试
- 复用和维护
- 升级和替换

需要多个功能时，拆成多个插件，通过事件或共享数据联动。

---

## 开发流程

### 新建插件 Checklist

```
□ 1. 确定插件职责边界（只做一件事）
□ 2. 创建目录结构（参照 scaffold-walkthrough）
□ 3. 写 manifest.json（注意 id 和目录名一致，UTF-8 无 BOM）
□ 4. 写至少 1 个工具（tools/*.js）
□ 5. 写 SKILL.md（教 Agent 怎么用）
□ 6. 安装到 dev slot 验证
□ 7. 跑通：页面加载 + 工具调用 + 技能匹配
□ 8. 发布到正式目录
```

### 开发工作流

```bash
# 1. 创建插件骨架
# （参照 plugin-scaffold-walkthrough.md 的步骤 1-5）

# 2. 安装到 dev slot
plugin.dev.install --path "W:/path/to/my-plugin"

# 3. 启用
plugin_dev_enable(pluginId="my-plugin", allowFullAccess=true)

# 4. 验证工具调用
plugin_dev_invoke_tool(pluginId="my-plugin", toolName="hello", input={name: "test"})

# 5. 查看诊断（日志、错误、加载状态）
plugin_dev_diagnostics(pluginId="my-plugin")

# 6. 热加载（修改代码后重新加载）
plugin_dev_reload(pluginId="my-plugin")

# 7. 发布
Copy-Item -Recurse "W:/path/to/my-plugin" "C:\Users\Administrator\.hanako\plugins\my-plugin"
```

### 调试三板斧

1. **`ctx.log?.info?.()`** — 在工具代码里打日志
2. **`plugin_dev_diagnostics()`** — 查看加载状态和日志
3. **分层检查** — `/page`（200?）→ `/assets/*.js`（200? 带 token?）→ 工具调用（有结果?）

---

## 常见陷阱与反模式

### 反模式 1：把所有规则塞进 SKILL.md

**症状**：SKILL.md 写了 500 行，包含配置说明、使用示例、注意事项、历史变更……

**问题**：SKILL.md 是"Agent 知识"，不是"项目文档"。它应该在会话启动时只加载名称+描述，正文按需加载。太长了占 Token 预算。

**正确做法**：
- SKILL.md 只写：名称、描述、工具清单、使用示例（3-5 行）
- 详细的配置说明、注意事项放在 README 或单独的文档里

### 反模式 2：用 PowerShell 写 JSON/MD 文件

**症状**：`Set-Content -Encoding UTF8` 写入文件带 BOM，JSON parser 报错。

**正确做法**：
```powershell
# ❌ 带 BOM
Set-Content -Path "file.json" -Value '{}' -Encoding UTF8

# ✅ 无 BOM
[System.IO.File]::WriteAllText("file.json", '{}', [System.Text.UTF8Encoding]::new($false))

# ✅ 或用 edit/write 工具（推荐，不用 shell）
```

**铁律**：修改 manifest.json 或 SKILL.md 永远不用 PowerShell Set-Content/Out-File。

### 反模式 3：插件职责过大

**症状**：一个插件既有工具、又有页面、又有 hook、又有技能，代码量超过 2000 行。

**问题**：
- 难以测试和调试
- 更新风险大（改一个小功能可能导致整个插件失效）
- 不利于复用

**正确做法**：
- 一个插件只做一件事
- 需要多个功能时拆成多个插件，通过事件总线联动
- 共享逻辑抽到 `tools/_lib/` 里

### 反模式 4：硬编码路径

**症状**：`path.join("C:/Users/Administrator/.hanako/plugins-data/...")`

**问题**：换机器、换用户名就挂了。

**正确做法**：
- 用 `ctx.dataDir` 获取插件数据目录
- 用 `ctx.resources` 访问用户资源
- 路径全部用正斜杠 `/`，不要用反斜杠 `\`

### 反模式 5：忽略 sessionPermission

**症状**：工具返回文件但宿主不显示，或者权限模式拦截。

**正确做法**：每个工具都声明 `sessionPermission`：
```js
export const sessionPermission = { readOnly: true };           // 只读
export const sessionPermission = { kind: "plugin_output" };    // 插件输出
export const sessionPermission = { kind: "session_file_output" }; // 媒体交付
```

### 反模式 6：把"一次性偏好"写成持久化配置

**症状**：用户选了"深色模式"，你把这个写到 manifest 或 config 里，变成全局生效。

**问题**：偏好是会话级的，不是插件级的。

**正确做法**：偏好通过工具参数传递，不写持久化存储。

### 反模式 7：路由里调用 ctx.model 不声明 capabilities

**症状**：`routes/ui.js` 中调用 `ctx.model.sample()` 报错 `Cannot read property 'sample' of undefined`。

**原因**：工具层的 `ctx.model` 始终可用，但路由层需要 manifest 声明 `capabilities: ["model"]`。

**正确做法**：
```json
{
  "trust": "full-access",
  "capabilities": ["model"]
}
```
> 在 `routes/ui.js` 里调 `ctx.model` 前，务必在 manifest 声明对应 capability。

### 反模式 8：manifest.json 遗漏 manifestVersion

**症状**：安装插件失败，提示缺少版本声明。

**原因**：漏写了 `"manifestVersion": 1`。

**正确做法**：`manifestVersion` 是必须字段，值固定为 `1`。每个 manifest 第一行就写。

---

## 安全规则

### 文件写入

| 规则 | 说明 |
|------|------|
| 不要写源码目录 | `ctx.pluginDir` 只读，只写 `ctx.dataDir` |
| 路径检查 | 路径来自用户输入时，用 `path.resolve` 检查是否在 dataDir 内 |
| 原子写入 | 重要文件先写 `.tmp` 再 rename，避免半写崩溃 |
| 编码 | 所有文本文件用 UTF-8 无 BOM |

### 网络请求

| 规则 | 说明 |
|------|------|
| 不要裸 fetch | 用 `pluginFetch`（0.333.0+），自动处理超时、重试 |
| 不要硬编码 URL | URL 从配置或用户输入读取 |
| 不要 log 敏感信息 | token、密码、完整 URL 禁止输出到日志 |

### 权限

| 规则 | 说明 |
|------|------|
| 最小权限原则 | 不需要 `full-access` 就用 `restricted` |
| 声明能力 | 需要访问用户资源时在 manifest 声明 `capabilities` |
| trust 和 capabilities 配合 | `capabilities: ["model"]` 需要 `trust: "full-access"` |

---

## 性能规则

### 上下文成本控制

| 方法 | 策略 |
|------|------|
| SKILL.md | 只写名称+描述常驻，正文按需加载 |
| Rules | 用 path-scoped 限制作用域 |
| Subagents | 旁路任务用 subagent，不污染主上下文 |
| Hooks | 配置在外层，不占上下文窗口 |

### 缓存策略

| 场景 | 策略 | TTL |
|------|------|-----|
| API 请求 | 文件缓存（`tools/_lib/cache.js`） | 7 天 |
| 图片/媒体 | 缩略图 + 原图分离 | 按需 |
| 会话数据 | `ctx.dataDir` 持久化 | 永久（用户可清理） |

---

## 测试策略

### 单元测试（工具函数）

```js
// tools/__tests__/config.test.js
import { readConfig, writeConfig } from "../_lib/config.js";

// 测试1：空配置返回默认值
// 测试2：写入后读取一致
// 测试3：损坏的 JSON 文件不崩溃
```

### 集成测试（dev slot 验证）

```bash
# 1. 安装到 dev slot
plugin.dev.install --path "W:/path/to/test-plugin"

# 2. 启用
plugin_dev_enable(pluginId="test-plugin", allowFullAccess=true)

# 3. 调用工具
plugin_dev_invoke_tool(pluginId="test-plugin", toolName="test_tool", input={...})

# 4. 查看诊断
plugin_dev_diagnostics(pluginId="test-plugin")
```

### 回归测试（发布前）

```bash
# 1. 禁用 dev slot 插件
plugin_dev_disable(pluginId="test-plugin")

# 2. 复制到正式目录
Copy-Item -Recurse "W:/path/to/test-plugin" "C:\Users\Administrator\.hanako\plugins\test-plugin"

# 3. 重启验证
# 4. 确认工具可调用、页面可加载
```

---

## 文档规范

### manifest.json

```json
{
  "manifestVersion": 1,
  "id": "plugin-name",
  "name": "显示名称",
  "version": "0.1.0",
  "description": "一句话描述插件功能",
  "trust": "full-access",
  "interface": {
    "displayName": "显示名称",
    "shortDescription": "简短描述",
    "category": "Productivity",
    "author": "作者名",
    "homepage": "https://github.com/xxx",
    "license": "MIT"
  }
}
```

### SKILL.md

```markdown
---
name: 技能显示名
description: 一句话描述技能用途和触发场景
---

# 技能名

## 工具清单

| 工具名 | 用途 |
|--------|------|
| `plugin_tool` | 简要说明 |

## 使用示例

```
plugin_tool(param="value")
```

## 注意事项

- 一行一条，不超过 5 条
```

### README.md（可选，发布用）

```markdown
# 插件名

一句话介绍。

## 功能
- 功能1
- 功能2

## 安装
1. 复制到 `~/.hanako/plugins/`
2. 重启 Hana

## 配置
见 manifest.json 或插件页面。
```

---

## 版本管理

### 语义化版本

```
主版本.次版本.修订版本

0.1.0 — 初始版本
0.1.1 — Bug 修复
0.2.0 — 新功能（向下兼容）
1.0.0 — 稳定发布
```

### 发布流程

```bash
# 1. 更新 manifest.json 版本号
# 2. 更新 CHANGELOG.md（如果有）
# 3. 打包
Compress-Archive -Path "plugin/*" -DestinationPath "plugin-v1.0.0.zip"
# 4. 复制到正式目录
Copy-Item -Recurse "plugin" "C:\Users\Administrator\.hanako\plugins\plugin"
# 5. 重启 Hana
```

---

## 参考文档

- **API 参考**：`plugin-api-reference.md`——所有字段、方法、权限的完整说明
- **模式手册**：`plugin-patterns.md`——20 个可复制的代码模板
- **脚手架 Walkthrough**：`plugin-scaffold-walkthrough.md`——从零到跑的最小步骤
- **路线图**：`plugin-roadmap.md`——从"能用"到"可靠"的演进方向
