# ClaudeCodeSaveToken
This is a set of strategies for saving tokens.[这是一套节省token的策略]灵感来源于https://www.anthropic.com/engineering/multi-agent-research-system

# Claude Code 模型/角色体系说明

给佬友们同步这套配置用的说明文档，包含：opusplan 模型路由的来源、使用方法，以及配套的子代理角色体系。

## 1. opusplan 是什么

`opusplan` 是 Claude Code CLI **内置**的一个模型别名（alias），不是某个配置文件里发明出来的东西——
我实测过：这个字符串直接硬编码在 `claude.exe` 可执行文件里（`grep -l opusplan` 能在二进制里搜到），
说明它是官方原生支持的预设，跟 `opus`、`sonnet`、`haiku` 这些普通模型别名是同一层级的东西，只是它不指向单一模型，而是指向一条**路由规则**。

规则内容（执行 `/model opusplan` 时 CLI 自己给出的说明）：

> Opus in plan mode, else Sonnet

即：

| 会话所处状态 | 实际调用的模型 |
|---|---|
| Plan 模式（规划阶段，尚未落地改代码） | Opus（当前为 `claude-opus-4-8`） |
| 非 Plan 模式（退出 Plan 后正式执行、写代码、跑命令） | Sonnet（当前为 `claude-sonnet-5`） |

设计意图：规划/架构决策这类高价值推理用最强模型，日常执行类任务用性价比更高的模型，自动切换，不需要手动来回改 `/model`。

## 2. 怎么使用

**设置：**
```
/model opusplan
```
执行后会写入 `~/.claude/settings.json`：
```json
{
  "model": "opusplan"
}
```
（写一次即永久生效，新开的会话都会用这条规则，不用每次都敲一遍。）

**查看当前设置：**
```
/model
```
不带参数，会显示当前生效的模型/别名。

**切回单一模型（不再分角色）：**
```
/model opus     # 全程 Opus
/model sonnet   # 全程 Sonnet
```

**如何进入/退出 Plan 模式：**
- 进入：让 Claude 先规划、不动手改代码时（例如复杂需求先出方案），CLI 会自动进入 Plan 模式，或可手动触发。
- 退出：方案确认后退出 Plan 模式开始执行（`ExitPlanMode`），之后的操作就自动切回 Sonnet。

**其他关联设置（可选）：**
```json
"effortLevel": "xhigh"
```
控制推理力度（reasoning effort），跟 opusplan 是两个独立维度，可以一起抄。

## 3. 配套的子代理角色体系

opusplan 只管"主会话"用什么模型。除此之外，本机还配置了 4 个独立的子代理角色（通过 `Agent` 工具调用，`subagent_type` 传对应名字），
每个角色的模型是**写死**在自己的定义文件里的，不受 opusplan 规则影响：

定义文件目录：`~/.claude/agents/*.md`

| 角色 (subagent_type) | 模型 | 权限 | 用途 |
|---|---|---|---|
| `Explore` | `haiku` | 只读：Read/Grep/Glob | 快速代码库搜索，不写文件 |
| `implementer` | `sonnet`（effort: medium） | Read/Write/Edit/Bash/Grep/Glob | 写新代码、实现功能 |
| `reviewer` | `sonnet` | 只读 + Bash | 代码审查，只分析不改 |
| `test-runner` | `sonnet` | Read/Bash/Grep/Glob | 跑测试、复现失败用例、验证修复 |

每个文件的格式（frontmatter + prompt）：
```markdown
---
name: implementer
description: 代码实现专家。需要写新代码、实现功能、搭建模块时使用...
tools: Read, Write, Edit, Bash, Grep, Glob
model: sonnet
effort: medium
---

你是一名资深软件工程师，专注干净、正确的实现。
...
```

调用方式（在对话里）：直接说"用 implementer 写这个功能"之类的，或者 Claude 自己判断任务类型后主动调用。

## 4. 给同事的部署步骤

**必须：**
1. 把整个 `~/.claude/agents/` 目录拷贝到同事的 `~/.claude/agents/` 下。
2. 让同事执行一次 `/model opusplan`（或手动在他的 `settings.json` 里加 `"model": "opusplan"`）。

**可选：**
3. 如果要效果级别一致，加一行 `"effortLevel": "xhigh"`。
4. 如果同事也想要同款状态栏（statusLine），需要额外拷贝 `~/.claude/statusline/statusline.ps1`，并把 `settings.json` 里 `statusLine.command` 中的路径从 `C:/Users/ZhuHongji/...` 改成他自己的用户路径。
5. `~/.claude/CLAUDE.md`（个人沟通风格设定，比如默认中文、简洁风格）是个人偏好，跟角色/模型体系无关，按需给，不强制。

**最简版（只要角色体系能跑起来）：**
```
拷贝 ~/.claude/agents/ 整个文件夹
执行 /model opusplan
