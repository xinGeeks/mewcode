# MewCode

终端里的 AI 编程助手。基于 [Textual](https://textual.textualize.io/) 构建 TUI，同时支持 Anthropic、OpenAI 和 OpenAI 兼容协议（DeepSeek、Kimi、GLM、MiniMax 等）的模型接口。

除了交互式的终端界面外，MewCode 还提供非交互式命令行模式、浏览器远程模式，以及子智能体、多智能体团队、Skill、Hook、记忆、Git Worktree、OS 级沙箱等能力。

## 特性

- **流式 TUI** —— 流式输出、思考过程展示、工具调用折叠、权限确认弹窗、斜杠命令注册表
- **多模型接入** —— `anthropic` / `openai` / `openai-compat` 三种协议，可配置多个 provider
- **工具体系** —— 文件读写编辑、bash、glob、grep、任务管理、worktree、skill、MCP 调用
- **子智能体** —— 内置 `Explore`、`Plan`、`Verification`、`general-purpose` 四种，支持自定义
- **多智能体团队** —— 邮箱通信、共享任务列表，队友可跑在独立窗格（tmux / iTerm2）或进程内
- **权限体系** —— 四种权限模式、规则匹配、危险命令检测，另有 bwrap / seatbelt 两种 OS 级沙箱后端
- **Hooks 引擎** —— 15 个生命周期事件，可执行 shell、HTTP 请求或复用模型
- **记忆** —— 跨会话召回、会话历史整合、项目级与用户级双层存储
- **非交互模式** —— `-p` 直接执行提示词，`--output-format stream-json` 输出 NDJSON 事件流
- **远程模式** —— `--remote` 启动 WebSocket 服务，用浏览器访问

## 环境要求

- Python >= 3.11
- [uv](https://docs.astral.sh/uv/)（推荐）或 pip

## 安装

使用 uv（项目自带 `uv.lock`）：

```bash
uv sync
uv run mewcode
```

或者用 pip 以可编辑模式安装：

```bash
pip install -e .
mewcode
```

## 快速开始

1. 在项目下创建配置文件 `.mewcode/config.yaml`：

   ```bash
   mkdir -p .mewcode
   ```

   最小内容：

   ```yaml
   providers:
     - name: anthropic-official
       protocol: anthropic
       base_url: https://api.anthropic.com
       api_key: "sk-ant-..."
       model: claude-sonnet-4-20250514
   ```

2. 启动：

   ```bash
   mewcode
   ```

`api_key` 留空时，会按协议自动回落到环境变量 `ANTHROPIC_API_KEY` 或 `OPENAI_API_KEY`。MCP 服务器的 `env` 和 `headers` 支持写 `${ENV_VAR}`，连接时展开。

`.mewcode/` 已在 `.gitignore` 中，配置和密钥不会被提交。

## 命令行用法

```bash
mewcode                            # 启动交互式 TUI
mewcode --mode plan                # 指定权限模式（覆盖配置文件）
mewcode -p "解释一下 src/main.py"  # 非交互式执行，结果打到 stdout
mewcode -p "..." --output-format stream-json   # 输出 NDJSON 事件流
mewcode --remote                   # 启动远程模式，浏览器访问 http://localhost:18888
```

`stream-json` 输出的事件类型：`assistant`、`thinking`、`tool_use`、`tool_result`、`usage`、`turn_complete`、`result`、`error`、`compact`、`retry`。

### 终端快捷键

| 按键 | 作用 |
| --- | --- |
| `Ctrl+C` | 退出 |
| `Esc` | 取消当前请求 |
| `Shift+Tab` | 循环切换权限模式 |
| `Ctrl+O` | 折叠 / 展开工具调用详情 |

## 配置

配置按以下顺序查找并**逐层合并**（后层覆盖前层）：

1. `~/.mewcode/config.yaml` —— 用户全局
2. `<项目>/.mewcode/config.yaml` —— 项目级
3. `<项目>/.mewcode/config.local.yaml` —— 本地覆盖（建议 gitignore）

完整的配置项：

```yaml
providers:                      # 必填，至少一个
  - name: anthropic-official
    protocol: anthropic         # anthropic | openai | openai-compat
    base_url: https://api.anthropic.com
    api_key: "sk-ant-..."
    model: claude-sonnet-4-20250514
    thinking: true              # 开启扩展思考
    context_window: 200000      # 0 表示自动推导
    max_output_tokens: 64000    # 0 表示按 thinking 自动决定

permission_mode: default        # default | acceptEdits | plan | bypassPermissions

mcp_servers:                    # MCP 服务器列表
  - name: context7
    command: npx                # stdio 方式
    args: ["-y", "@upstash/context7-mcp"]
  - name: my-http-mcp           # HTTP 方式（Streamable HTTP）
    url: https://example.com/mcp
    headers:
      Authorization: "Bearer ${MY_TOKEN}"

hooks:                          # 见下文「Hooks」
  - id: fmt-on-write
    event: post_tool_use
    if: tool == WriteFile
    action:
      type: command
      command: ruff format .

enable_fork: true                       # 是否允许 fork 模式的 skill / 子智能体
enable_verification_agent: false        # 是否注册 verification 子智能体
enable_coordinator_mode: false          # Lead 只调度不写代码

teammate_mode: ""               # 强制队友进程内运行；留空则按当前终端自动检测

worktree:
  symlink_directories: ["node_modules", ".venv", "vendor"]
  stale_cleanup_interval: 3600
  stale_cutoff_hours: 24

sandbox:
  enabled: false                # 启用 OS 级沙箱
  auto_allow: false             # 沙箱兜底：自动放行命令
  network_enabled: false        # 沙箱内允许网络访问
```

### context window 推导

模型的上下文窗口按四层回退依次解析：

1. 配置里显式写的 `context_window`（永远优先）
2. 从 provider 的 `/v1/models` 端点自动拉取（仅 `anthropic` 协议会走这一层）
3. 内置的「模型名子串 → 窗口」映射表
4. 保守默认值（`claude` → 200000，其他 → 128000）

## 斜杠命令

| 命令 | 说明 |
| --- | --- |
| `/help` (`/h`, `/?`) | 显示帮助信息 |
| `/status` (`/s`) | 显示状态信息 |
| `/clear` | 清除对话历史 |
| `/compact` (`/c`) | 压缩上下文 |
| `/plan` (`/p`) | 切换到 Plan 模式 |
| `/session` | 会话管理 |
| `/memory` | 记忆管理 |
| `/mcp` | 显示 MCP 服务器状态 |
| `/skill` (`/skills`) | 技能包管理：`list` / `info <name>` / `reload` |
| `/sandbox` | 沙箱管理 |
| `/rewind` | 回溯到之前的检查点 |
| `/tasks` (`/task`) | 后台任务管理 |
| `/trace` (`/tree`) | 查看 Agent 父子追踪树 |
| `/worktree` (`/wt`) | 管理 Git Worktree |

每个已加载的 Skill 还会自动注册一个同名斜杠命令。

在 `.mewcode/commands/` 下放 `*.md` 文件即可新增自定义命令，文件名即命令名（子目录用冒号拼接命名空间，如 `git/log.md` → `/git:log`）。可选的 YAML frontmatter 支持 `description`、`argument-hint`、`aliases`，正文里的 `$ARGUMENTS` 会被替换成用户输入的参数。

## 工具

Lead 的内置工具：`ReadFile`、`WriteFile`、`EditFile`、`Bash`、`Glob`、`Grep`、`Agent`、`AskUserQuestion`、`TaskStop`、`TeamCreate` / `TeamDelete`、`SendMessage`、`EnterWorktree` / `ExitWorktree`、`LoadSkill` / `InstallSkill`、`ExitPlanMode`、`SyntheticOutput`，以及 MCP 工具检索与分发（`ToolSearch` / `mcp_call`）。

团队协作工具 `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` 只发给队员，Lead 侧用 `/tasks` 查看后台任务。

MCP 工具的加载方式按 schema 总量自动选择：总量在 context window 的 10% 以内时走 `eager`，全部注入；超出后，官方 Anthropic 端点走 `native`（工具留在 `tools[]` 里打 `defer_loading` 标记，交给服务端决定是否展示），其他端点走 `dispatch`（延迟工具整个藏起来，靠 `ToolSearch` / `mcp_call` 按需捞取）。也可用环境变量强制指定。

## 权限系统

四种权限模式决定读写和命令是否需要确认：

| 模式 | 读 | 写 | 执行命令 |
| --- | --- | --- | --- |
| `default` | 自动放行 | 询问 | 询问 |
| `acceptEdits` | 自动放行 | 自动放行 | 询问 |
| `plan` | 自动放行 | 询问 | 询问 |
| `bypassPermissions` | 自动放行 | 自动放行 | 自动放行 |

### 权限规则

规则文件是一组 `rule` / `effect` 条目，effect 取 `allow`、`deny`、`ask`。三份文件按 `~/.mewcode/permissions.yaml` → `.mewcode/permissions.yaml` → `.mewcode/permissions.local.yaml` 顺序加载：

```yaml
- rule: "Bash(git status*)"
  effect: allow
- rule: "ReadFile(./secrets/*)"
  effect: deny
- rule: "mcp_call(linear__*)"
  effect: ask
```

规则语法是 `工具名(匹配模式)`，模式用 fnmatch 匹配工具的对应参数（`Bash` 匹配命令，`ReadFile` / `WriteFile` / `EditFile` 匹配文件路径，`Glob` / `Grep` 匹配 pattern，`mcp_call` 匹配 `server__tool`）。裁决优先级为 **deny > ask > allow**，规则写在哪一层都不影响结果。

### OS 级沙箱

路径级检查之外，还有一层内核级隔离：macOS 用 `sandbox-exec`（Seatbelt），Linux 用 `bubblewrap`（bwrap）。即使命令绕过路径检查，内核也会阻止越权写入。Linux 上需要安装 `bwrap`。

## 子智能体

模型通过 `Agent` 工具派生子智能体，`subagent_type` 指定类型（区分大小写，需与定义里的 `name` 一致）。内置四种：

| 类型 | 说明 |
| --- | --- |
| `Explore` | 只读搜索专家，用 Glob / Grep / ReadFile 定位代码 |
| `Plan` | 架构规划，只读探索后输出分步实现计划 |
| `Verification` | 验证专家，尝试打破实现找出隐藏 bug |
| `general-purpose` | 通用子 Agent，拥有全部工具 |

`Verification` 默认不注册，需要在配置里打开 `enable_verification_agent: true`。

自定义智能体用 Markdown 定义，放在 `.mewcode/agents/`（项目级）或 `~/.mewcode/agents/`（用户级）：

```markdown
---
name: Reviewer
description: 审查代码变更，找出问题
disallowedTools:
  - WriteFile
model: haiku
maxTurns: 15
permissionMode: default
isolation: worktree
---

你是一个代码审查专家……
```

`name` 和 `description` 必填；可选字段还有 `tools`（白名单）、`disallowedTools`（黑名单）、`model`（默认 `inherit`）、`maxTurns`（默认 200）、`permissionMode`、`background`、`isolation`（`worktree` 表示在独立 worktree 里跑）。项目级定义覆盖用户级，用户级覆盖内置。

## Skills

Skill 是可复用的提示词包，放在 `.mewcode/skills/`（项目级）或 `~/.mewcode/skills/`（用户级）。技能名要匹配 `^[a-z][a-z0-9-]*$`。两种格式：

- `技能名/SKILL.md` —— frontmatter 写 `name` / `description`，正文是提示词
- `技能名/skill.yaml` + `技能名/prompt.md` —— frontmatter 写 `name` / `description` / `mode` / `model` / `context`

`mode` 取 `inline`（在当前对话里展开）或 `fork`（开一个继承上下文的子进程），`context` 取 `full` / `recent` / `none`，`model` 可指定专用模型。

模型看到匹配的 skill 描述后会调用 `LoadSkill` 激活。`/skill reload` 可热重载。

`InstallSkill` 支持从 GitHub 安装，支持的 URL 形式：

- `https://www.skills.sh/<owner>/<repo>/<skill-name>`
- `https://github.com/<owner>/<repo>/tree/<ref>/<subpath>`
- `https://raw.githubusercontent.com/<owner>/<repo>/<ref>/<subpath>/SKILL.md`

## 多智能体团队

团队模式把复杂任务拆给多个 agent 并行推进：

1. `TeamCreate` 建团队
2. `Agent` 工具带 `team_name` + `name` 拉起队员（不带 `team_name` 则是阻塞式的一次性子智能体）
3. 队员通过 `SendMessage` 按名字互相通信，Lead 在每轮开头收到系统提醒
4. 队员干完活把结果发给 `lead`，然后进入空闲状态
5. Lead 汇总所有结果

队员之间还有一张共享任务表（`TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate`）。团队状态存在 `~/.mewcode/teams/<团队名>/`。

队友跑在哪里由后端自动决定：`teammate_mode` 设为 `in-process` 或处于非交互模式时一律进程内运行；否则检测当前是否身处 tmux（`$TMUX`）或 iTerm2（`$ITERM_SESSION_ID`）会话，是就开独立窗格跑独立进程（`mewcode --teammate --team-name <t> --agent-name <n>`），都不是就回退到进程内。Windows 始终进程内运行。

`enable_coordinator_mode: true` 时 Lead 只负责调度，不直接改代码。

## Hooks

Hook 在生命周期节点上触发自定义动作。共 15 个事件：

`session_start`、`session_end`、`turn_start`、`turn_end`、`pre_tool_use`、`post_tool_use`、`pre_send`、`post_receive`、`startup`、`shutdown`、`error`、`compact`、`permission_request`、`file_change`、`command_execute`

动作类型有四种：`command`（执行 shell）、`prompt`（交给模型）、`http`（发请求）、`agent`。

```yaml
hooks:
  - id: block-secrets
    event: pre_tool_use
    if: tool == WriteFile
    reject: true
    action:
      type: command
      command: "grep -q SECRET '$FILE_PATH'"
  - id: audit
    event: post_tool_use
    once: true
    action:
      type: http
      url: https://example.com/audit
      method: POST
      body: '{"tool": "$TOOL_NAME"}'
```

`reject: true` 只允许用在 `pre_tool_use` 上：条件一旦命中就无条件拦截这次工具调用，命令的输出作为拒绝理由回传给模型（命令自身的退出码只影响日志，不影响是否拦截）。`if` 支持 `tool`、`event`、`args.<字段>` 条件组合，命令和消息里的 `$EVENT`、`$TOOL_NAME`、`$FILE_PATH`、`$TOOL_ARGS.<字段>` 会被替换成实际值。

## 记忆

两层存储：

- 项目级 `<项目>/.mewcode/memory/`
- 用户级 `~/.mewcode/memory/`（`user` / `feedback` 类型跟随用户，`project` / `reference` 类型留在项目里）

记忆文件是带 frontmatter 的 Markdown，`MEMORY.md` 作为索引入口。系统会在会话中自动召回相关记忆，并在会话结束后把对话整合成长期记忆。

另外，指令文件会被自动发现并拼进系统提示，顺序为：`~/.mewcode/MEWCODE.md` → `~/.mewcode/AGENTS.md` → 从 git root 到工作目录的 `MEWCODE.md` / `AGENTS.md` / `.mewcode/MEWCODE.md` → `MEWCODE.local.md`。这些文件支持 `include` 指令。

## Git Worktree

`/worktree` 可以在独立 worktree 里干活，避免污染主分支。worktree 建在 `.mewcode/worktrees/` 下，`node_modules`、`.venv`、`vendor` 等目录会软链过去而不是复制。`EnterWorktree` / `ExitWorktree` 工具供模型在会话中切换。

## MCP

在配置里加 `mcp_servers` 即可接入。stdio 方式写 `command` + `args`，HTTP 方式写 `url`（走 Streamable HTTP 传输）。`/mcp` 查看连接状态。

## 远程模式

```bash
mewcode --remote
```

在 `0.0.0.0:18888` 上同时提供静态页面和 WebSocket 服务，浏览器访问 `http://localhost:18888` 即可使用完整的 TUI 能力（流式输出、权限确认、斜杠命令）。

## 目录约定

```
<项目>/.mewcode/
├── config.yaml            # 项目级配置
├── config.local.yaml      # 本地覆盖
├── permissions.yaml       # 项目级权限规则
├── permissions.local.yaml # 本地权限规则（界面上确认过的规则会写到这里）
├── agents/                # 自定义子智能体
├── skills/                # Skill 技能包
├── commands/              # 自定义斜杠命令
├── memory/                # 项目级记忆
├── sessions/              # 会话历史
├── worktrees/             # Git worktree
└── debug.log              # 运行日志
```

用户级目录 `~/.mewcode/` 结构类似，另有 `teams/`（团队状态）和 `MEWCODE.md` / `AGENTS.md`（全局指令）。

## 项目结构

```
mewcode/
├── __main__.py       # CLI 入口
├── app.py            # Textual TUI
├── agent.py          # Agent 主循环、事件流
├── client.py         # LLM 客户端（Anthropic / OpenAI / OpenAI 兼容）
├── config.py         # 配置加载与合并
├── validator.py      # 配置校验
├── context/          # 上下文管理、压缩、工具结果预算
├── agents/           # 子智能体定义与加载
├── teams/            # 多智能体团队
├── skills/           # Skill 技能包
├── hooks/            # Hooks 引擎
├── memory/           # 记忆召回与整合
├── permissions/      # 权限检查、规则、危险命令
├── sandbox/          # OS 级沙箱（bwrap / seatbelt）
├── worktree/         # Git worktree 管理
├── mcp/              # MCP 客户端与加载策略
├── commands/         # 斜杠命令注册表与处理器
├── tools/            # 工具实现
└── filehistory/      # 文件历史与回溯
```

## 开发

```bash
uv sync                        # 安装依赖
uv run pytest                  # 运行测试
uv run pytest tests/test_agent.py -v
```

测试覆盖 agent 主循环、权限、hooks、团队协作、记忆、worktree 以及各类工具。

## 许可证

未指定。
