# OpenCode 实现

> Thinker + Executor 模式在 [OpenCode](https://github.com/opencode-ai/opencode) 平台的实现指南。

OpenCode 原生支持多 Agent 协作，通过 **Primary Agent + Subagent** 机制实现思考-执行分离，是本模式最自然的载体。

---

## 目录

- [OpenCode Agent 机制](#opencode-agent-机制)
- [配置文件总览](#配置文件总览)
- [architect.md — 思考者（Primary）](#architectmd--思考者primary)
- [coder.md — 执行者（Subagent）](#codermd--执行者subagent)
- [finder.md — 扫描者（Subagent，只读）](#findermd--扫描者subagent只读)
- [安装步骤](#安装步骤)
- [权限控制](#权限控制)
- [测试验证](#测试验证)
- [常见问题](#常见问题)

---

## OpenCode Agent 机制

### 三种核心概念

| 概念 | 说明 |
|------|------|
| **Primary Agent** | 主 Agent，拥有完整的会话上下文和决策权。对应模式中的 **Architect** |
| **Subagent** | 子 Agent，由 Primary 通过 `Task` 工具调用。独立的会话上下文，执行完返回结果。对应 **Coder / Finder** |
| **Task 工具** | Primary 调用 Subagent 的内置工具。可以指定子 Agent 名称、任务描述、所需文件 |

### 工作流程

```
Primary (Architect)  ←── 用户输入
    │
    ├─ Task tool → Subagent (Finder)  ──→ 扫描代码结构 ──→ 返回定位信息
    │
    ├─ Task tool → Subagent (Coder A) ──→ 执行任务包 A ──→ 返回 Summary
    ├─ Task tool → Subagent (Coder B) ──→ 执行任务包 B ──→ 返回 Summary
    │
    └─ 汇总验证 → 输出结果给用户
```

### 关键特性

- **上下文隔离**：每个 Subagent 有独立的会话，不会互相污染
- **并行执行**：多个 Subagent 可以同时被调用（只要子任务无依赖）
- **权限分离**：通过 `permission.task` 控制 Primary 可以调用哪些 Subagent
- **模型独立**：每个 Agent 可以指定不同的模型

---

## 配置文件总览

OpenCode 的 Agent 配置文件使用 YAML front matter + Markdown 正文的格式。每个 Agent 对应一个 `.md` 文件。

本模式需要创建以下文件：

| 文件 | Agent 类型 | 模型 | 角色 |
|------|-----------|------|------|
| `architect.md` | primary | deepseek-chat / v4-pro | 思考者 |
| `coder.md` | subagent | deepseek-chat / v4-flash | 执行者 |
| `finder.md` | subagent | deepseek-chat / v4-flash | 扫描者 |

所有文件放置在 OpenCode 的 Agent 目录中，通常是：

- **Windows**: `C:\Users\<用户名>\.config\opencode\agents\`
- **Linux/macOS**: `~/.config/opencode/agents/`

---

## architect.md — 思考者（Primary）

### 配置

```yaml
---
name: architect
mode: primary
model: deepseek-chat
modelParams:
  - key: model
    value: deepseek-chat
modelTemperature: 0.3
permission:
  tool: task
  subAgents:
    - coder
    - finder
---
```

### Prompt 正文

```markdown
# 角色

你是 **Architect（架构师）**，Thinker+Executor 模式中的深度思考者。

## 核心原则

1. **强制分发** — 没有 write/edit/bash 权限，所有改动必须走 Coder。你只负责思考、拆解、验收。
2. **任务明确** — 输出的任务包必须让 Coder 不用思考就能执行。定位要精确到文件名、函数名、行号。
3. **严格验收** — 对 Coder 返回的 Summary 逐项检查，超出范围的改动必须指出。

## 任务包格式

当你需要调用 Coder 时，必须使用以下格式生成任务包：

```markdown
## 任务包

### 定位
- **目标文件**: [文件路径]
- **修改区域**: [函数名/行号范围]
- **上下文**: [必要的背景信息]

### 分析
- **现状**: [当前代码的行为]
- **约束**: [什么不能动]
- **风险**: [可能的风险]

### 目标
- **期望结果**: [具体的行为变化]
- **验收标准**: [可验证的检查点]
```

## Finder 调用时机

在以下情况必须先调用 Finder 扫描项目结构：
- 接收到与新项目或未知代码库相关的任务
- 需要修改的文件路径不确定
- 需要了解项目中有哪些文件、哪些函数可用

在以下情况可以跳过 Finder，直接调用 Coder：
- 用户已明确指定目标文件和位置
- 项目很小（< 10 个文件）

## 验收清单

收到 Coder 的 Summary 后，逐项检查：

- [ ] 所有任务包目标是否都有对应的完成内容？
- [ ] 变更文件是否都在任务包指定范围内？
- [ ] 是否有超出范围的改动？（不允许）
- [ ] 验证结果是否可信？
- [ ] 遗留问题是否需要跟进？

## 工作流程

1. 用户输入需求
2. 分析需求，判断是否需要 Finder
3. 如果需要 → 调用 Finder 扫描结构 → 根据结果拆解任务
4. 生成任务包 → 调用 Coder 执行
5. 收到 Summary → 逐项验收
6. 重复步骤 4-5 直到所有子任务完成
7. 汇总结果，向用户报告
```

---

## coder.md — 执行者（Subagent）

### 配置

```yaml
---
name: coder
mode: subagent
model: deepseek-chat
modelParams:
  - key: model
    value: deepseek-chat
modelTemperature: 0.2
---
```

### Prompt 正文

```markdown
# 角色

你是 **Coder（执行者）**，Thinker+Executor 模式中的代码执行者。

## 核心原则

1. **不思考** — 你只做一件事：按照任务包的要求改代码。不做决策、不搞设计、不质疑需求。
2. **不越界** — 只动任务包指定的文件和区域。任务包没说的地方不要碰。
3. **不重构** — 不要"顺手"改其他代码。即使看到不好的代码也不要改。
4. **必提交 Summary** — 执行完成后必须提交 Summary，这是 Architect 验收的唯一依据。

## 工作流程

1. 接收 Architect 发来的任务包
2. 读取目标文件，确认当前状态
3. 严格按照任务包的「目标」执行改动
4. 验证结果（编译/测试/检查）
5. 提交 Summary

## Summary 格式

```markdown
## Summary

### 完成内容
- [具体做了什么改动]

### 变更文件
- `文件路径`: [改动说明]

### 验证结果
- [x] [验证项]

### 遗留问题
- [如有，没有则写"无"]
```

## 注意事项

- 有任何不确定的地方，按最保守的方式执行
- 改动完成后必须验证（编译、语法检查或手动确认）
- 不要创建任务包未要求的文件
```

---

## finder.md — 扫描者（Subagent，只读）

### 配置

```yaml
---
name: finder
mode: subagent
model: deepseek-chat
modelParams:
  - key: model
    value: deepseek-chat
modelTemperature: 0.1
---
```

### Prompt 正文

```markdown
# 角色

你是 **Finder（扫描者）**，Thinker+Executor 模式中的代码侦察兵。

## 核心原则

1. **只读** — 你绝不能修改任何文件。你只有读取和查看的权限。
2. **快速** — 扫描代码结构，不深入分析业务逻辑。
3. **结构化输出** — 返回格式化的定位信息，方便 Architect 做决策。
4. **不做决策** — 只提供信息，不给建议。不要说"我觉得应该这样改"。

## 输出格式

```markdown
## 扫描报告

### 项目结构
- [目录/文件结构概览]

### 关键文件
- `文件路径`: [文件用途简述]

### 关键函数/符号
- `文件名::函数名`: [函数用途]

### 注意事项
- [任何需要注意的结构特征]
```

## Finder 的命令

你的所有操作必须使用以下只读命令：

- `read` — 读取文件内容
- `glob` — 按模式搜索文件名
- `grep` — 搜索文件内容

**严禁使用** `write`、`edit` 或任何可修改文件的工具。
```

---

## 安装步骤

### 第一步：复制配置文件

将以下 3 个文件复制到 OpenCode 的 Agent 目录中：

```powershell
# Windows PowerShell
$agentDir = "$env:USERPROFILE\.config\opencode\agents"
New-Item -ItemType Directory -Path $agentDir -Force

# 复制文件（将 <repo-path> 替换为实际路径）
Copy-Item "<repo-path>\implementations\opencode.md" "$agentDir\architect.md"
# 或者直接使用上文的配置内容逐个创建文件
```

如果使用 Git 仓库，也可以直接从仓库复制：

```powershell
Copy-Item "D:\code\thinker-executor-pattern\implementations\opencode.md" "$env:USERPROFILE\.config\opencode\agents\architect.md"
```

> **注意：** `opencode.md` 是本文档（实现指南），不是 Agent 配置文件。你需要根据本文档中的配置示例，**分别创建** `architect.md`、`coder.md`、`finder.md` 三个文件。

### 第二步：创建 coder.md 和 finder.md

确保 Agent 目录下有以下 3 个文件：

```
C:\Users\<用户名>\.config\opencode\agents\
├── architect.md   # Primary Agent (思考者)
├── coder.md       # Subagent (执行者)
└── finder.md      # Subagent (扫描者，只读)
```

### 第三步：重启 OpenCode

重新启动 OpenCode，使新的 Agent 配置生效。

### 第四步：验证

在 OpenCode 中输入以下命令验证配置是否生效：

```
/list agents
```

应能看到 `architect`（primary）、`coder`（subagent）、`finder`（subagent）三个 Agent。

---

## 权限控制

### permission.task 配置

在 `architect.md` 中，通过 `permission` 字段控制 Primary Agent 可以调用哪些 Subagent：

```yaml
permission:
  tool: task
  subAgents:
    - coder
    - finder
```

### 配置解读

| 字段 | 值 | 说明 |
|------|-----|------|
| `permission.tool` | `task` | 限制只能使用 Task 工具调用 Subagent |
| `permission.subAgents` | `[coder, finder]` | 只允许调用 coder 和 finder 这两个 Subagent |

### 权限模型

```
Architect (primary)
    │
    ├─ ✅ 允许: Task → coder    (写代码)
    ├─ ✅ 允许: Task → finder   (扫代码)
    │
    └─ ❌ 拒绝: 直接修改文件    (除非开启紧急模式)
```

> **注意：** OpenCode 的权限控制是通过 Agent 配置中的 `permission` 字段实现的。如果不配置 `permission`，Primary Agent 默认拥有全部权限（包括直接修改文件）。设置了 `permission` 后，Primary 只能通过 Task 调 Subagent 来完成工作，自己不能直接操作文件——这正好强制了"思考者不执行"的原则。

### 如果有多级权限需求

Architect 不应有直接修改文件的能力。不要用 prompt 声明"紧急可动"，应该在权限配置中直接设置 write/edit/bash: deny，从根本上杜绝绕过 Coder 的行为。

---

## 测试验证

### 验证 Agent 加载

```
/list agents
```

预期输出包含：

```
architect (primary): deepseek-chat
coder (subagent): deepseek-chat
finder (subagent): deepseek-chat
```

### 验证 Finder 只读

在 OpenCode 中（确保当前使用的是 Architect）：

```
扫描一下当前目录的代码结构
```

预期行为：
1. Architect 调用 Finder（通过 Task 工具）
2. Finder 执行只读操作（glob / read / grep）
3. 返回结构化的扫描报告
4. **Finder 不修改任何文件**

### 验证完整的 Thinker+Executor 流程

输入以下测试指令：

```
帮我创建一个 calc.c 文件，实现一个 add 函数，能算两个 int 的和。
```

预期流程：

| 步骤 | 角色 | 操作 | 验证点 |
|------|------|------|--------|
| 1 | Architect | 分析需求，判断是否需要 Finder | 输出分析过程 |
| 2 | Architect | 或者直接生成任务包（目标很明确） | 任务包包含定位/分析/目标 |
| 3 | Coder | 接收任务包，创建 calc.c | 文件被创建 |
| 4 | Coder | 验证文件内容，提交 Summary | Summary 包含验证结果 |
| 5 | Architect | 验收 Summary | 确认符合预期 |
| 6 | Architect | 向用户报告最终结果 | 结果正确 |

### 验证权限隔离

```
切换对话或确保当前是 Architect，尝试直接修改文件：
```

Architect 如果配置了 `permission` 限制，应该无法直接修改文件，只能通过 Task 调用 Coder。

---

## 常见问题

### Q: 模型找不到

**问题：** 启动后提示模型不存在或找不到。

**检查清单：**

1. OpenCode 是否配置了 API Key？检查 `C:\Users\<用户名>\.config\opencode\opencode.json`：
   ```json
   {
     "providers": {
       "openai": { "apiKey": "sk-..." },
       "anthropic": { "apiKey": "sk-..." }
     }
   }
   ```
2. `modelParams` 中的 `value` 是否填写了正确的模型名？
3. 尝试先不用该模型运行 OpenCode，确认模型可用的
4. 如果使用 DeepSeek，确保 OpenCode 配置了 DeepSeek provider

### Q: Task 调用失败

**问题：** Architect 调用 Coder 或 Finder 时，Task 工具报错。

**检查清单：**

1. Subagent 配置文件是否正确放置在 Agent 目录下？
2. `name` 字段是否与 `permission.subAgents` 中的名称一致？（大小写敏感）
3. Subagent 的 `mode` 是否为 `subagent`？
4. 重新启动 OpenCode 后是否确认 `/list agents` 能看到所有 Agent？

**常见错误：**

```yaml
# ❌ 错误：name 不一致
# architect.md 中 permission.subAgents: [coder]
# 但文件名是 coder-agent.md → name 应该是 coder-agent

# ✅ 正确：name 与 permission 一致
# 文件名: coder.md → name: coder → permission.subAgents: [coder]
```

### Q: 权限问题

**问题：** Architect 无法调用 Coder，提示无权限。

**检查清单：**

1. `architect.md` 中 `permission.subAgents` 是否包含 `coder` 和 `finder`？
2. `coder.md` 和 `finder.md` 的 `name` 字段是否与 `permission.subAgents` 中的名称匹配？
3. 是否在 architect.md 中正确配置了 `permission.tool: task`？

### Q: Subagent 执行了但文件没改

**问题：** Coder 报告执行成功，但文件内容没有变化。

**可能原因：**

1. 权限问题：Subagent 的模型可能没有文件系统写入权限
2. 路径问题：Coder 可能在错误的目录中创建了文件
3. OpenCode bug：重启 OpenCode 后重试

**排查方法：**

在 Coder 的 prompt 中加入验证步骤，例如创建文件后立即读取验证。

### Q: 怎么确认当前用的是哪个 Agent？

OpenCode 的界面或状态栏通常会显示当前 Agent 的名称。也可以通过输入以下指令确认：

```
你是谁？
```

Architect 会回答"我是 Architect"，Coder 会回答"我是 Coder"。

### Q: 可以用一个文件配置多个 Agent 吗？

**不可以。** OpenCode 要求每个 Agent 一个独立的 `.md` 文件，文件名对应 Agent 名称。

如果你希望共享一些配置（比如 prompt 模板），使用 `include` 功能（如果 OpenCode 支持）或手动复制。

### Q: Agent 配置文件修改后需要重启吗？

**是的。** 任何 Agent 配置文件的修改（包括 prompt、permission、model 等），都需要重启 OpenCode 才能生效。

---

## 参考

- [OpenCode 官方文档](https://github.com/opencode-ai/opencode)
- [Thinker + Executor 模式总览](../README.md)
- [Claude Code 实现](claude-code.md)
- [Cursor 实现](cursor.md)
