---
name: plugin-dev-guide
description: Hana 插件开发完整指引，覆盖从立项、开发、调试、交接、备份到审查的全生命周期。MANDATORY TRIGGERS：开发插件、写插件、做插件、插件开发、继续开发、换窗口、新窗口见、交接、插件备份、审查插件、插件完成了。
SOFT TRIGGERS（卡住时触发）：卡住了、又不行、还是没解决、翻踩坑、查坑、修不好了、搞不定
default-enabled: true
references:
  - path: references/01-api-reference.md
    title: API 参考
  - path: references/02-plugin-patterns.md
    title: 代码模式（20 个模板）
  - path: references/03-scaffold-walkthrough.md
    title: 脚手架教程
  - path: references/04-best-practices.md
    title: 最佳实践
  - path: references/05-roadmap.md
    title: 架构路线图
  - path: references/06-error-fixes.md
    title: 常见报错修复
  - path: references/07-plugin-structure.md
    title: 插件结构速查
  - path: references/08-ponytail-overlay.md
    title: Ponytail 覆盖规则
  - path: references/09-common-pitfalls.md
    title: 常见踩坑记录（40 条）
  - path: references/10-success-patterns.md
    title: 成功经验模式（30 条）
  - path: scripts/create_hana_plugin.py
    title: 自动化脚手架脚本
  - path: assets/sdk
    title: 内置 SDK tarballs
trigger_keywords:
---

# 插件开发指引

## 这个 skill 解决什么

不只是技术参考手册。**首次激活时，简要告知用户你可以做的五件事**：项目日志管理、换窗口交接、自动备份、回归测试、多 agent 审查。

核心痛点：

### 自动触发规则

以下规则对所有使用此 skill 的助手生效，不需用户提醒：

**当调试同一个问题出现以下任一情况，必须建议用户查踩坑记录：**
- 尝试第3种修复方案仍无效
- 连续2种不同思路的修复尝试均失败
- 用户说出卡住了又不行还是没解决等词时

**当出现以下任一情况，必须建议用户换窗口，且主动做好交接记录后再换：**
- 连续第3次修复尝试后仍无效
- 已尝试2种以上不同思路的修复方案
- 用户说出又不行还是没解决卡住了

1. **换窗口后上下文丢失** → 项目日志文件（PROJECT_LOG.md）持久化状态
2. **修 bug 搞坏其他功能而不自知** → 功能验证清单 + 新窗口启动时自动后端回归测试
3. **改坏了回不去** → 关键节点自动备份，保留多版本快照
4. **缺少多视角审查** → 开发中轻量审查 + 完成后完整审查

---

## 开发全流程总览

```
🏁 初始化 → 🔨 日常开发（修bug/做功能，循环）
                  ↓ 搞不定了
              🔄 换窗口交接 → 🚀 新窗口继续 → 循环日常开发
                                             ↓ 做完了
                                        ✅ 最终审查 → 🎉 完成
```

---

## 快速导航

| 用户问... | 去哪里 |
|---------|--------|
| 插件能做什么、不能做什么 | [边界校验](#边界校验) |
| 该选什么插件形态 | [插件形态选择](#插件形态选择) |
| 从零创建插件 | [流程一：初始化](#流程一初始化新项目) |
| 换窗口继续开发 | [流程四：新窗口启动](#流程四新窗口启动) |
| 忘记进度了 | [流程四：新窗口启动](#流程四新窗口启动) |
| 插件跑不起来，一步步排查 | [调试与排障](#调试与排障) |
| 怎么测试插件 | [调试与排障](#调试与排障) |
| 看插件目录结构 | [references/07-plugin-structure.md](references/07-plugin-structure.md) |
| 备份当前进度 | [流程五：备份机制](#流程五备份机制) |
| 找人审查代码 | [流程六：最终审查](#流程六最终审查插件完成) |
| 查 API 字段 | [references/01-api-reference.md](references/01-api-reference.md) |
| 找代码模板 | [references/02-plugin-patterns.md](references/02-plugin-patterns.md) |
| 看踩坑记录 | [references/09-common-pitfalls.md](references/09-common-pitfalls.md) |
| 看成功经验模式 | [references/10-success-patterns.md](references/10-success-patterns.md) |
| 修具体报错 | [references/06-error-fixes.md](references/06-error-fixes.md) |

### 按场景查文档

| 遇到的情况 | 先读这份 |
|---|---|
| 具体报错信息 | references/06-error-fixes.md |
| 代码不知道怎么写 | references/02-plugin-patterns.md |
| 插件加载/行为异常 | references/09-common-pitfalls.md |
| 设计插件结构时 | references/07-plugin-structure.md |
| 想优化代码质量 | references/04-best-practices.md |

---

## 边界校验

判断需求是否在 Hana 插件能力边界内：

- ✅ **可做**：Agent 调用函数、iframe 页面/卡片、lifecycle hook、EventBus、后台任务、Session/Agent 管理、模型调用、媒体生成、文件处理
- ❌ **超出边界**：原生渲染器组件、代码沙箱、细粒度权限提示、远程发布自动安装
- ❌ 超出边界时：明确告知用户当前插件体系的边界，建议最接近的替代方案

> **Core Rule**：插件是独立的产品边界。插件状态存储在 plugin data 中，通过 manifest 贡献和 SDK API 暴露行为，禁止导入 Hana 渲染器或宿主内部实现。

---

## 插件形态选择

| 形态 | 适合场景 | 权限 |
|------|---------|------|
| **Tool-only** | Agent 调用函数，无需 UI | `restricted` |
| **Runtime** | lifecycle、EventBus、task、schedule、动态工具 | `full-access` |
| **UI** | page、widget、iframe 卡片 | `full-access` |

---

## 项目日志文件

每个插件项目在**源码根目录**下维护一个 `PROJECT_LOG.md`，结构如下：

```markdown
# [插件名] 开发日志

## 项目缘起
（为什么做这个插件，想解决什么问题）

## 想要的效果
（用户视角：插件做完了是什么样的体验）

## 当前版本
（manifest.json 里的 version）

## 外部依赖
（插件依赖的外部进程/服务，如 Python 子进程端口、外部 API key 等）

## 当前进度
（最近完成了什么，接下来要做什么）

## 已知问题 / TODO
- [ ] ...

## 经验记录
### 🐛 踩坑
（带时间戳的完整上下文：日期 + 症状 + 原因 + 修复方式。修完 bug 后追加。需要快速检索时，从这里提取症状→修复的映射即可，不需要单独维护映射表。）
### ✅ 成功经验
（做了什么 + 为什么这个方案好）

## 功能验证清单
### 后端工具
- [ ] `tool_name` - 描述 - 自动: 是/否 - 输入：xxx / 预期返回：xxx
### 前端 UI
- [ ] 功能描述 - 如何验证（人眼/手动操作）

## 备份记录
（自动维护，记录每次备份的时间、节点、文件路径）

## 审查记录
- YYYY-MM-DD — 审查类型（开发中/最终）— 审查人 — 主要结论
```

---

## 流程一：初始化（新项目）

**触发**：用户说"开发 xx 插件""做 xx 插件"等，且项目目录下不存在 PROJECT_LOG.md

**步骤**：
1. 确认项目源码目录
2. **如果项目目录下还没有插件骨架**（manifest.json、tools/、routes/ 等），先调用 hana-plugin-creator skill 生成 scaffold
3. 创建 `PROJECT_LOG.md`，填入项目缘起和想要的效果（和用户确认后写入）
4. 创建 `_backups/` 目录
5. 告知用户：「项目日志已初始化，可以开始开发了」

> 本 skill 负责**项目管理**（日志、备份、审查、交接），hana-plugin-creator 负责**代码生成**（scaffold、SDK、模板）。初始化时两者配合使用。

---

## 流程二：日常开发

### 修完 bug 后

1. 追加到「经验记录 → 🐛 踩坑」（症状 + 原因 + 修复）
3. 轻量提醒：「🔍 相关功能要不要顺手测一下？」

### 完成功能模块后

1. 更新「当前进度」
2. 追加到「经验记录 → ✅ 成功经验」（做了什么 + 为什么这个方案好）
3. 更新「功能验证清单」（新增/修改相关条目）
4. 执行备份（见流程五）

### 用户说"找人看看"（开发中轻量审查）

1. 确认要看什么模块/文件
2. 派一个 subagent 做代码审查，审阅指定代码并给出反馈
3. 将反馈呈现给用户，讨论是否采纳

---

## 流程三：换窗口交接

**触发**：用户说"换窗口了""新窗口见""换个窗口"等

**步骤**：
1. 执行最后一次备份（描述写"交接前快照"）
2. 更新 PROJECT_LOG.md：
   - 「当前进度」更新为交接状态（已完成、卡在哪、接下来做什么）
   - 「已知问题 / TODO」列出当前未解决的 bug（现象 + 复现步骤 + 试过的方向）
3. 告知用户：「交接文档已更新，数据都在 PROJECT_LOG.md 里了。还没弄好，等我更新完再换。」

> ⚠️ 交接文档只写**事实**，不写**判断**（不要写"我觉得问题肯定是 xxx"）。让新窗口自己判断。

---

## 流程四：新窗口启动

**触发**：用户说"继续开发 xx 插件""继续鼓捣 xx"等

### 前置检查

1. 检查 PROJECT_LOG.md 是否存在。如果不存在，回退到流程一（初始化）
2. 检查"允许 Agent 插件开发工具"是否已开启。未开启则提示用户先去设置开启

### 第一步：读取日志

1. 读取 PROJECT_LOG.md
2. 向用户展示摘要：项目是什么、当前进度、上次卡在哪、接下来要做什么

### 第二步：自动后端回归测试

1. 读取「功能验证清单 → 后端工具」中标记为「自动: 是」的条目
2. 对每条调用 `plugin_dev_invoke_tool` 测试。**如果插件未启用，跳过测试并提示用户先启用**
3. 汇总结果：

```
📋 后端回归测试结果：
✅ tool_name_1 — 正常
❌ tool_name_3 — 返回错误：xxx
⚠️ tool_name_4 — 插件未启用，跳过
```

4. 如果有失败的，立即告知用户

### 第三步：列出前端验证清单

1. 读取「功能验证清单 → 前端 UI」部分
2. 提醒用户：「后端测试跑完了，前端以下功能需要你手动验证」

### 第四步：继续开发

询问用户：「从哪里开始？」

---

## 流程五：备份机制

### 触发时机

- 完成一个功能模块后
- 换窗口交接前
- 做重大改动前
- 用户说"备份一下""备份插件"时

### 执行方式

```powershell
# Windows 环境（Hana 默认运行在 Windows 上）
$name = "插件名_$(Get-Date -Format 'yyyy-MM-dd_HHmmss')_描述"
Compress-Archive -Path "插件源码目录\*" -DestinationPath "插件源码目录\_backups\$name.zip"
```

### 保留策略

保留最近 **5** 个备份。备份时检查总大小，超过 500MB 则删除最旧的。

### 完整步骤

1. 执行压缩命令
2. 检查备份数量，超过 5 个删除最旧；检查总大小，超过 500MB 继续清理
3. 在 PROJECT_LOG.md「备份记录」中追加条目
4. 告知用户备份完成

---

## 流程六：最终审查（插件完成）

**触发**：用户说"插件做完了""插件完成了"等

**步骤**：
1. 先做一次最终备份（描述写"完成版本"）
2. 更新 PROJECT_LOG.md「当前进度」为"已完成"
3. 并行派多个 subagent 审查，每个聚焦一个维度（根据当前环境可用的 agent 自适应选择）：

| 审查维度 | 关注点 |
|---|---|
| 代码质量 | 逻辑正确性、错误处理、边界情况、安全性 |
| 用户体验 | 交互流程是否顺畅、错误提示是否友好、命名是否直观 |
| 逻辑一致性 | 配置项是否矛盾、默认值是否合理、状态管理是否有漏洞 |

4. 汇总所有审查结果，按优先级（严重/建议/可选）呈现，追加到「审查记录」
5. 询问用户是否要修改，还是直接打包发布

---

## 调试与排障

### 调试操作链

```bash
# 1. 安装到 dev slot
plugin_dev_install(sourcePath="插件源码目录")

# 2. 启用
plugin_dev_enable(pluginId="my-plugin", allowFullAccess=true)

# 3. 验证工具调用
plugin_dev_invoke_tool(pluginId="my-plugin", toolName="tool_name", input={...})

# 4. 查看诊断
plugin_dev_diagnostics(pluginId="my-plugin")

# 5. 热加载（修改代码后）
plugin_dev_reload(pluginId="my-plugin")
```

> 注意：Agent 可见的 dev 工具默认关闭，需在设置 → 插件 → 权限中开启"允许 Agent 插件开发工具"。

### 分层定位法

遇到异常时逐层排查：**后端 API → 外部进程 → 前端 JS → 通信层**。不跳层、不猜，每层确认后再往下。

---

## 预置调试映射表

以下是最高频的几条，初始化时自动写入 PROJECT_LOG.md。完整列表（40 条）见 [references/09-common-pitfalls.md](references/09-common-pitfalls.md)。

| 症状 | 可能原因 | 修复方向 |
|---|---|---|
| 页面 200 但显示加载失败 | iframe 没发 `hana.ready` | 检查前端脚本是否发送 ready 握手 |
| `/assets/panel.js` 403 | 外部脚本未带 token | 页面生成 script URL 时透传 token |
| 前端改完不生效 | 插件没重新加载 | `plugin_dev_reload` 或重新打开插件抽屉 |
| 工具调用返回 undefined | 工具未在 manifest.json 声明 | 检查 manifest.json 的 tools 字段 |
| 修改静态文件后页面没变化 | 浏览器缓存 | `plugin_dev_reload` |
| 路由 404 | 路由未注册或 import 重复 | 检查日志中是否有 ERROR |

---

## 重要规则

### 始终要做的事

- ✅ 修完重要的跨会话 bug 后记录到「经验记录」
- ✅ 新窗口启动时自动跑后端回归测试
- ✅ 关键节点执行备份
- ✅ 交付前走多 agent 审查

### 不要做的事

- ❌ 交接文档中写主观判断
- ❌ import Hana 内部源码
- ❌ 文件交付用手写路径（走 `stageFile()`）
- ❌ 擅自删除备份文件（`_backups/` 下的 zip）

---

## 术语表

| 术语 | 一句话解释 |
|------|-----------|
| dev slot | Hana 的开发调试槽，重启后清空，适合临时调试 |
| full-access | 插件权限等级，可读写文件、注册路由、lifecycle hook |
| restricted | 插件权限等级，只能提供 tools/skills/commands，不能注册路由 |
| manifest | `manifest.json`，插件的身份证和权限声明文件 |
| route | `routes/` 下注册的 HTTP 端点 |
| bus / EventBus | 插件间通信的消息总线 |
| `hana.ready` | iframe 页面加载完成后向宿主发的握手信号 |
| token query | 页面 URL 上的鉴权参数，子资源需手动透传 |
| split chunk | 前端构建工具自动拆分的 JS/CSS 文件，可能因鉴权失败 |
| PROJECT_LOG.md | 本 skill 定义的项目日志文件 |

---

## 内置资源

| 资源 | 说明 |
|------|------|
| `scripts/create_hana_plugin.py` | 插件脚手架生成器 |
| `assets/sdk/*.tgz` | SDK 包（4 个） |
| `references/01-api-reference.md` | 完整 API 字段说明 |
| `references/02-plugin-patterns.md` | 20 个可复制代码模板 |
| `references/03-scaffold-walkthrough.md` | 从零到跑的最小步骤 |
| `references/04-best-practices.md` | 反模式、安全、测试、版本管理 |
| `references/05-roadmap.md` | 架构演进方向 |
| `references/06-error-fixes.md` | 常见报错修复 |
| `references/07-plugin-structure.md` | 插件目录结构速查 |
| `references/08-ponytail-overlay.md` | ponytail 决策阶梯覆盖 |
| `references/09-common-pitfalls.md` | 40 条社区实战踩坑记录 |
| `references/10-success-patterns.md` | 30 条成功经验模式 |