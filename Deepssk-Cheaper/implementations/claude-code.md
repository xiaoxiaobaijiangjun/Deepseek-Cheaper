# Claude Code 实现指南

> 基于 [Claude Code Sub-agents](https://docs.anthropic.com/en/docs/claude-code/sub-agents) 机制，实现 Thinker + Executor 模式。

---

## 目录

1. [Claude Code Agent 机制](#claude-code-agent-机制)
2. [架构总览](#架构总览)
3. [Architect 配置：CLAUDE.md](#architect-配置claudemd)
4. [Coder Subagent 配置](#coder-subagent-配置)
5. [Finder Subagent 配置](#finder-subagent-配置)
6. [Task 工具调用方式](#task-工具调用方式)
7. [安装步骤](#安装步骤)
8. [测试验证](#测试验证)
9. [模型选择建议](#模型选择建议)
10. [目录结构](#目录结构)

---

## Claude Code Agent 机制

### `.claude/agents/` 目录

Claude Code 通过项目根目录下的 `.claude/agents/` 目录来定义 subagent。每个 Markdown 文件对应一个 subagent：

```
.claude/
├── CLAUDE.md           # 主 Agent 指令（Architect 角色）
└── agents/
    ├── coder.md         # 执行者 subagent
    └── finder.md        # 侦察兵 subagent
```

### 工作方式

- **CLAUDE.md** 定义主会话的架构和行为准则（充当 Architect 角色）
- **`.claude/agents/*.md`** 每个文件定义一个 subagent，包含指令和角色定义
- **Task 工具** 是主 Agent 调用 subagent 的唯一接口
- 每个 subagent 运行在**独立上下文**中，只看到自己的指令和传入的任务

### 关键特性

| 特性 | 说明 |
|------|------|
| **上下文隔离** | 每个 subagent 独立上下文，互不干扰 |
| **角色分离** | 不同模型分配给不同角色（Opus 思考 + Haiku 执行） |
| **可编程** | Subagent 指令可精确控制行为、权限 |
| **CLI 兼容** | 也可通过 `claude Task -p "..." --subagent coder` 命令行调用 |

---

## 架构总览

```
┌──────────────────────────────────────────────┐
│  Claude Code 主会话 (Architect / Opus)        │
│  CLAUDE.md: "只思考，不执行"                   │
│                                               │
│  收到需求 → 分析 → 调用 Finder →              │
│  拆解任务 → 调用 Coder → 验收 Summary          │
└──────────────┬────────────────────────────────┘
               │ Task 工具
               ├──────────────────────────────┐
               │                              │
               ▼                              ▼
┌─────────────────────────┐  ┌─────────────────────────┐
│ Coder Subagent (Haiku)  │  │ Finder Subagent (Haiku)  │
│ .claude/agents/coder.md │  │ .claude/agents/finder.md │
│                         │  │                         │
│ 职责: 读文件、改代码     │  │ 职责: 只读扫描代码结构   │
│ 权限: Read+Write+Execute│  │ 权限: Read only          │
│ 输出: Summary           │  │ 输出: 结构化定位信息     │
└─────────────────────────┘  └─────────────────────────┘
```

---

## Architect 配置：CLAUDE.md

将以下内容保存为项目根目录的 `.claude/CLAUDE.md`：

```markdown
# Architect 角色指令

你是 **思考者（Architect）**，**不直接修改代码**。

## 核心原则

1. **只思考，不执行** — 你分析需求、拆解任务、验收结果。改代码的事交给 Coder。
2. **先了解再决策** — 不确定项目结构时，先用 `Task` 调用 finder 扫描。
3. **任务包必须精确** — 给 Coder 的任务包必须包含：定位、分析、目标。
4. **严格验收** — Coder 提交 Summary 后逐项检查，超范围改动必须拒绝。
5. **默认分发** — 能交给 Coder 的就不要自己动手。

## 工作流程

```
收到需求
  ↓
[1] 分析需求，拆解为独立子任务
[2] 需要了解代码结构？→ Task subagent=finder → 获取定位信息
[3] 生成任务包（定位+分析+目标）
[4] Task subagent=coder → 传入任务包
[5] 收集 Summary → 逐项验收
[6] 全部通过 → 告知用户完成
     部分失败 → 生成修正任务包 → 再调 Coder
```

## 任务包格式

```markdown
## 任务包

### 定位
- **目标文件**: <文件路径>
- **修改区域**: <函数名/行号范围>
- **上下文**: <相关背景>

### 分析
- **现状**: <当前状态>
- **约束**: <不能动的部分>
- **风险**: <需要注意的风险>

### 目标
- **期望结果**: <具体改动描述>
- **验收标准**: <如何验证>
```

## 验收清单

收到 Coder 的 Summary 后检查：

- [ ] 所有任务目标是否都有对应完成内容？
- [ ] 变更文件是否在指定范围内？
- [ ] 有无超出范围的改动？（**不允许**）
- [ ] 验证结果是否可信？
- [ ] 遗留问题是否需要跟进？

## 角色约束

| 能力 | 权限 |
|------|------|
| 读文件 | ✅ |
| 写文件 | ❌（紧急时可 ask 用户确认） |
| 执行命令 | ❌（紧急时可 ask 用户确认） |
| 调用 subagent | ✅ 通过 Task 工具 |
```

---

## Coder Subagent 配置

将以下内容保存为 `.claude/agents/coder.md`：

```markdown
# Coder 智能体 — 代码执行者

你是**执行者**，不是思考者。职责：解析任务包 → 执行改动 → 提交 summary。

## 核心工作流

```
收到任务包
    ↓
[1] 解析任务包 — 理解定位、分析、目标
    ↓
[2] 读取目标文件 — 确认当前状态
    ↓
[3] 执行改动 — 严格按目标执行
    ↓
[4] 验证结果 — 编译/测试/检查
    ↓
[5] 提交 summary — 报告完成情况
```

## Summary 格式（必须遵守）

```markdown
## Summary

### 完成内容
- [具体做了什么改动]

### 变更文件
- `文件路径`: [改动说明]

### 验证结果
- [x] 编译通过 / 测试通过 / 手动验证
- [如有失败，说明原因]

### 遗留问题
- [如有，没有则写"无"]
```

## 执行原则

### 必须做
- **严格按任务包执行** — 目标是什么就改什么
- **改动范围精确** — 只动任务包指定的文件和区域
- **提交 summary** — 每次执行完必须提交
- **保留原有风格** — 匹配项目现有代码风格

### 绝对不做
- **不超出范围** — 任务包没说的文件/功能不要碰
- **不重构** — 不要"顺手"改其他代码
- **不加功能** — 不要加任务包没要求的功能
- **不问问题** — 有歧义时按最保守的方式执行，在 summary 中说明

## 关键约束

- **你没有决策权** — 任务包就是命令
- **你没有设计权** — 架构是 architect 的事
- **你只有执行权** — 写代码、改文件、验证、报告
- **summary 必须交** — 这是 architect 验收的唯一依据

## 语言偏好

- 跟随任务包的语言
- summary 用中文
```

---

## Finder Subagent 配置

将以下内容保存为 `.claude/agents/finder.md`：

```markdown
# Finder 智能体 — 代码侦察兵

你是**侦察兵**，不是分析者。职责：扫描代码结构 → 返回定位信息。

## 核心原则

1. **只读** — 不修改任何文件。你没有任何写入权限。
2. **快速扫描** — 扫结构，不深入分析代码逻辑。
3. **结构化输出** — 返回格式化的定位信息，供 Architect 决策。
4. **不做决策** — 只提供信息，不给建议或判断。

## 工作流

```
收到扫描请求
    ↓
[1] 确认扫描范围（目录/文件模式）
    ↓
[2] 使用 glob/read 工具扫描结构
    ↓
[3] 返回结构化定位信息
```

## 输出格式

```
## 扫描结果

### 项目结构
<目录树或文件列表>

### 目标文件详情
- 文件: <路径>
- 大小: <行数>
- 关键函数/定义: <列表>

### 约束检测
- [是否存在目标文件?]
- [是否有命名冲突?]
- [是否有相关引用?]
```

## 关键约束

- **没有写入权限** — 发现 bug 也**不**修改，只报告
- **不分析逻辑** — 不判断代码好坏，只描述结构
- **不推测意图** — 不猜测"用户可能想要什么"
- **快速返回** — 信息够用就停，不要过度扫描

## 调用示例

Architect 可能这样调用你：
- "扫描 src/ 目录，找到所有 .c 和 .h 文件"
- "查看 config.h 中定义了哪些宏"
- "找到 main.c 中 init() 函数的位置"
```

---

## Task 工具调用方式

### 在 CLAUDE.md 中声明调用

在 CLAUDE.md 中通过以下格式定义 Task 调用行为：

```markdown
## Subagent 调用

当需要调用 subagent 时，使用以下格式：

Task subagent=coder: <任务包内容>
Task subagent=finder: <扫描指令>
```

### 实际调用示例

**调用 Finder：**

```
Task subagent=finder: 扫描 src/ 目录下的所有 .c 文件，列出文件名和函数定义
```

**调用 Coder：**

```
Task subagent=coder:

## 任务包

### 定位
- **目标文件**: src/calc.c
- **修改区域**: 新建文件
- **上下文**: 需要实现一个简单的加法函数

### 分析
- **现状**: 文件不存在，需要创建
- **约束**: 无已有依赖，纯函数实现
- **风险**: 无

### 目标
- **期望结果**: 创建 src/calc.c，包含 int add(int a, int b) 函数
- **验收标准**: gcc -c src/calc.c 编译通过
```

### Task 工具参数说明

| 参数 | 必填 | 说明 |
|------|------|------|
| `subagent` | 是 | Subagent 名称，对应 `.claude/agents/` 中的文件名（不含 .md） |
| `text` | — | 任务文本直接在冒号后传入，见上方示例 |

> **注意：** Claude Code 的 Task 工具暂不支持传递复杂结构化 JSON 参数。建议使用自然语言任务描述 + Markdown 格式的任务包。

### 串行调用多个 Coder

```
1.  Task subagent=coder: <任务包 A>
2.  → 收集 Summary A → 验收
3.  Task subagent=coder: <任务包 B>
4.  → 收集 Summary B → 验收
```

### 并行调用多个 Coder

Claude Code 的 Task 工具**不支持在同一轮对话中并行调用**。但 Architect 可以：

1. 先拆解所有任务包并规划好顺序
2. 逐个调用 Coder 执行
3. 最后统一验收

---

## 安装步骤

### 前置条件

- Claude Code CLI 已安装并可用（`claude --version`）
- 已登录 Anthropic 账号

### 步骤

1. **创建目录结构**

```bash
mkdir -p .claude/agents
```

2. **创建 CLAUDE.md**

将上方 [Architect 配置](#architect-配置claudemd) 中的内容保存为 `.claude/CLAUDE.md`。

3. **创建 Coder Subagent**

将上方 [Coder Subagent 配置](#coder-subagent-配置) 中的内容保存为 `.claude/agents/coder.md`。

4. **创建 Finder Subagent**

将上方 [Finder Subagent 配置](#finder-subagent-配置) 中的内容保存为 `.claude/agents/finder.md`。

5. **验证目录结构**

```
.claude/
├── CLAUDE.md
└── agents/
    ├── coder.md
    └── finder.md
```

6. **启动 Claude Code**

```bash
cd <your-project>
claude
```

7. **首次使用指定模型（Architect 用 Opus）**

```bash
claude --model claude-opus-4-20250514
```

> 你可以在项目根目录创建 `.claude/settings.json` 来持久化模型配置（详见 [Claude Code 设置文档](https://docs.anthropic.com/en/docs/claude-code/settings)）。

### 验证安装

在 Claude Code 中执行：

```
请扫描当前项目的目录结构。
```

预期行为：
1. Architect 解析需求
2. 调用 Finder subagent 进行扫描
3. Finder 返回结构化目录结构
4. Architect 将结果呈现给用户

---

## 测试验证

### 测试用例：创建文件 + 实现函数

执行指令：

```
在 src/ 下创建一个 calc.c 文件，包含一个 int add(int a, int b) 函数。
```

### 预期执行流程

| 步骤 | 角色 | 行为 |
|------|------|------|
| 1 | Architect | 分析需求，先调用 Finder 查看 src/ 是否存在 |
| 2 | Finder | 扫描 src/ 目录，返回当前状态 |
| 3 | Architect | 生成任务包（定位 + 分析 + 目标） |
| 4 | Coder | 创建 calc.c，实现 add 函数，编译验证 |
| 5 | Coder | 提交 Summary |
| 6 | Architect | 逐项验收 → 告知用户结果 |

### 验收标准

| # | 标准 | 期望结果 |
|---|------|---------|
| 1 | calc.c 被创建 | 文件存在于 src/ |
| 2 | 函数实现正确 | `int add(int a, int b) { return a + b; }` |
| 3 | 编译通过 | `gcc -c src/calc.c` 无错误 |
| 4 | Summary 完整 | 包含完成内容、变更文件、验证结果 |
| 5 | 未越界 | 没有修改其他文件 |

### 测试指令清单

```
1. 扫描测试：     "扫描项目目录结构"
2. 创建文件测试： "在 src/ 下创建 calc.c，实现 add 函数"
3. 修改文件测试： "在 calc.c 中新增 subtract 函数"
4. 拒绝越界测试： "顺便帮我优化一下其他文件的命名"（应被 Architect 拒绝）
```

---

## 模型选择建议

### Claude Code 可用模型

| 模型 | 代号 | 特点 | 推荐角色 |
|------|------|------|---------|
| **Claude Opus** | `claude-opus-4-20250514` | 最强推理能力，成本高 | **Architect**（思考者） |
| **Claude Sonnet** | `claude-sonnet-4-20250514` | 平衡性能和成本 | Architect 备选 / 复杂 Coder |
| **Claude Haiku** | `claude-haiku-3-20250513` | 快速、便宜，编码能力扎实 | **Coder / Finder**（执行者） |

### 推荐组合

| 场景 | Architect | Coder | Finder | 说明 |
|------|-----------|-------|--------|------|
| **生产项目** | Opus | Haiku | Haiku | 最强思考 + 快速执行，性价比最优 |
| **预算敏感** | Sonnet | Haiku | Haiku | 性能稍降，成本更低 |
| **快速实验** | Haiku | Haiku | Haiku | 全 Haiku 方案，适用于小项目 |
| **质量优先** | Opus | Sonnet | Haiku | Coder 更强，适合复杂执行场景 |

### 模型选择原则

```
思考者（Architect）  = 最贵的模型（Opus）
   ↓ 需要深度推理、任务拆解、质量验收
执行者（Coder/Finder）= 最便宜的够用模型（Haiku）
   ↓ 专注执行、不做决策、用完即走
```

### 启动命令示例

```bash
# Architect 用 Opus（默认），Coder/Finder 在 subagent 配置中指定模型
claude --model claude-opus-4-20250514

# 或使用 Sonnet 作为 Architect
claude --model claude-sonnet-4-20250514
```

> **注意：** Claude Code subagent 目前继承主会话的模型。如果平台未来支持每个 subagent 独立指定模型，届时可以更新配置使各角色使用不同模型。当前可通过 `.claude/settings.json` 配置文件级设置。

---

## 目录结构

完整的 Thinker+Executor 模式在 Claude Code 中的目录结构：

```
your-project/
├── .claude/
│   ├── CLAUDE.md              # Architect 角色指令（主 Agent）
│   ├── agents/
│   │   ├── coder.md           # Coder subagent 配置
│   │   └── finder.md          # Finder subagent 配置
│   └── settings.json          # （可选）Claude Code 设置
├── src/                       # 项目源代码
├── include/                   # （可选）头文件
├── Makefile                   # 项目构建文件
└── README.md
```

---

## 参考资料

- [Claude Code Sub-agents 官方文档](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Claude Code 设置文档](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Thinker + Executor 模式核心文档](../pattern.md)
- [Andrej Karpathy on LLM Coding Pitfalls](https://x.com/karpathy/status/2015883857489522876)
