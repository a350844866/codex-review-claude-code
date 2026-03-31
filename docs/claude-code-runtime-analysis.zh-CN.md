# Claude Code 运行逻辑拆解

## 0. 文档范围

这份文档基于 `instructkr/claude-code` 当前仓库状态做分析，而不是基于 Anthropic 官方发布的 Claude Code 源码。

必须先说明一个关键事实：

- 这个仓库当前主分支保存的是一个 **Python porting workspace**
- 它不是原始 Claude Code 的完整 TypeScript 运行时
- 它保存了大量 **结构性证据**，包括：
  - 顶层入口文件映射
  - 子系统名与 sample file
  - 207 条命令镜像条目
  - 184 条工具镜像条目
  - 归档快照统计：`1902` 个 TS-like 文件

因此，本文分成两层来讲：

1. 这个仓库当前的 Python 代码究竟怎么运行
2. 根据镜像元数据，原始 Claude Code 很可能是怎样组织和执行的

凡是基于文件名、目录名、sample file、命令/工具清单推导出来的部分，我都会明确标记为“推断”。

---

## 1. 一句话结论

如果只看当前仓库，可执行逻辑其实很薄：

- `src/main.py` 是 CLI 入口
- `src/commands.py` 和 `src/tools.py` 负责加载镜像 JSON
- `src/runtime.py` 负责把用户 prompt 粗粒度路由到“可能相关的命令/工具”
- `src/query_engine.py` 和 `src/port_manifest.py` 负责把当前 Python 工作区渲染成摘要
- 大量子目录只是占位包，它们暴露的是“原始 Claude Code 某个子系统存在过、规模大概多大、有哪些 sample file”

但如果把这些元数据拼起来，原始 Claude Code 的形态已经很清晰：

- 它是一个 **终端交互式 agent harness**
- 入口在 `entrypoints/cli.tsx`
- UI 层高度依赖 React/Ink 风格的终端组件树
- 用户侧有大量 `/commands`
- 模型侧有独立 `tools/*` 能力面
- 工具有权限系统、沙箱模式、计划模式、worktree 模式、子代理/团队代理、MCP、插件、远程会话、记忆系统、分析埋点和状态管理

换句话说，Claude Code 不是一个“单次问答 CLI”，而是一个：

`终端 UI + 状态机 + LLM 回路 + 工具编排层 + 权限治理层 + 多代理系统 + 远程/插件/MCP 扩展层`

---

## 2. 当前仓库的真实代码结构

### 2.1 当前可执行 Python 主体

当前 `src/` 里真正有执行逻辑的主要是这些文件：

- `src/main.py`
- `src/port_manifest.py`
- `src/query_engine.py`
- `src/runtime.py`
- `src/commands.py`
- `src/tools.py`
- `src/parity_audit.py`
- 以及少量简单数据结构或占位辅助文件

当前 Python 文件总数是 `54`。

### 2.2 当前仓库的本质

当前仓库更像一个“Claude Code 结构索引器”，而不是“Claude Code 完整复刻版”：

- `reference_data/archive_surface_snapshot.json` 记录原始快照顶层文件/目录与总量
- `reference_data/commands_snapshot.json` 记录命令面
- `reference_data/tools_snapshot.json` 记录工具面
- `reference_data/subsystems/*.json` 记录各子系统的规模和 sample file

所以，当前仓库实际上干的是两件事：

1. 把归档快照做成一个可查询的数据集
2. 用一个很薄的 Python CLI 暴露这个数据集

---

## 3. 当前 Python Workspace 的运行逻辑

这一节只描述“这个仓库今天怎么跑”。

### 3.1 启动入口

入口是 `python3 -m src.main ...`。

`src/main.py` 用 `argparse` 注册这些子命令：

- `summary`
- `manifest`
- `parity-audit`
- `subsystems`
- `commands`
- `tools`
- `route`
- `show-command`
- `show-tool`

这意味着当前仓库没有真正的 agent loop，没有真实的对话循环，也没有真实的模型调用。它只是一个查询与摘要 CLI。

### 3.2 启动后第一步：构建 manifest

`main()` 一进来就调用 `build_port_manifest()`。

`src/port_manifest.py` 的逻辑很直白：

- 扫描 `src/` 下所有 `.py`
- 按顶层模块名或文件名做计数
- 为已知文件附加说明文字
- 产出 `PortManifest`

这个 manifest 是当前 Python 工作区的“目录级视图”。

### 3.3 `summary` 的执行路径

当执行：

```bash
python3 -m src.main summary
```

调用链是：

1. `src.main.main()`
2. `build_port_manifest()`
3. `QueryEnginePort(manifest).render_summary()`
4. `build_command_backlog()`
5. `build_tool_backlog()`
6. 渲染 Markdown 文本并输出

这里的 “QueryEngine” 不是 LLM 推理引擎，而是一个“摘要渲染器”。

### 3.4 `commands` / `tools` 的执行路径

当执行：

```bash
python3 -m src.main commands --limit 20
python3 -m src.main tools --limit 20
```

调用链分别是：

- `render_command_index()`
- `render_tool_index()`

它们都会先走各自的 snapshot loader：

- `src.commands.load_command_snapshot()`
- `src.tools.load_tool_snapshot()`

这两个 loader 的共同点：

- 从 JSON 读取镜像数据
- 转成 `PortingModule`
- 用 `@lru_cache(maxsize=1)` 缓存

所以当前仓库没有数据库，没有网络服务，没有动态注册中心。数据源就是本地 JSON。

### 3.5 `route` 的执行路径

当前仓库中最接近“运行时路由器”的是 `src/runtime.py`。

执行：

```bash
python3 -m src.main route 'review MCP tool'
```

调用链是：

1. `src.main.main()`
2. `PortRuntime().route_prompt(prompt, limit=...)`
3. `_collect_matches()` 分别对 command/tool 清单打分
4. `_score()` 按 token 是否命中 `name/source_hint/responsibility` 累加得分
5. 优先保留每种 kind 的一个代表项
6. 再按得分与名称排序补齐剩余结果

这不是智能调度器，只是一个基于关键字的启发式检索器。

### 3.6 子系统包的运行逻辑

像这些目录：

- `src/assistant`
- `src/bridge`
- `src/cli`
- `src/components`
- `src/hooks`
- `src/services`
- `src/utils`

当前都只有一个 `__init__.py` 占位文件。它们的共同模式是：

1. 读取 `reference_data/subsystems/<name>.json`
2. 暴露：
   - `ARCHIVE_NAME`
   - `MODULE_COUNT`
   - `SAMPLE_FILES`
   - `PORTING_NOTE`

也就是说，这些包现在不承载业务逻辑，只承载元数据。

### 3.7 `parity-audit` 的运行逻辑

`src/parity_audit.py` 的职责是比较：

- 当前 Python workspace
- 参考的 TypeScript archive surface snapshot

它做的事情包括：

- 检查顶层根文件映射是否存在
- 检查顶层目录映射是否存在
- 统计当前 Python 文件数
- 对照归档参考里的总 TS-like 文件数
- 对照命令/工具镜像条目数

这里要注意：

- 它依赖 `reference_data/archive_surface_snapshot.json`
- 如果本地真的放了 `archive/claude_code_ts_snapshot/src`，会做更完整的存在性判断
- 但即便 archive 不在，参考统计仍然存在，因此“结构对照”仍可进行

### 3.8 测试覆盖范围

`tests/test_porting_workspace.py` 当前主要覆盖的是：

- manifest 能否构建
- summary CLI 能否跑
- parity audit 能否跑
- 命令/工具条目数是否足够
- 几个元数据占位包是否正确暴露 sample file
- route / show-command / show-tool CLI 是否能跑

测试证明当前仓库的重点是“元数据可读性”和“CLI 查询能力”，不是“真实 agent 行为”。

---

## 4. 当前代码审查结果

### 4.1 已确认并修复的问题

我在审查时发现一个真实缺陷：

- 原来的 `src/task.py` 是自引用导入：
  - `from .task import PortingTask`
- 这会导致：
  - `from src.task import PortingTask` 失败
  - `import src.tasks` 失败

这个问题不会被原测试发现，因为原测试没有覆盖 `src.task` / `src.tasks` 导入路径。

我已经将其修复为真正的 dataclass 定义，并新增测试覆盖 `default_tasks()`。

### 4.2 其他观察

没发现第二个同等级阻塞问题，但有几个边界值得知道：

- 当前 `find_commands()` / `find_tools()` 的过滤维度只看 `name` 和 `source_hint`
- `route_prompt()` 的匹配维度更宽，还会看 `responsibility`
- 这意味着“列表查询”和“路由查询”的召回标准并不完全一致

这不一定是 bug，但属于行为不完全统一。

---

## 5. 从镜像数据反推出的 Claude Code 总体架构

这一节开始是 **基于元数据的架构推断**。

推断依据包括：

- 顶层文件名
- 子系统目录名
- 子系统 sample file
- command/tool family 名称
- 状态、hooks、services、entrypoints 等目录分工

### 5.1 顶层轮廓

归档快照的顶层根文件有这些核心项：

- `main.tsx`
- `QueryEngine.ts`
- `Task.ts`
- `Tool.ts`
- `commands.ts`
- `context.ts`
- `history.ts`
- `ink.ts`
- `interactiveHelpers.tsx`
- `replLauncher.tsx`
- `setup.ts`
- `tasks.ts`
- `tools.ts`

这套命名基本说明原始系统具备：

- 终端 React/Ink UI
- REPL 入口
- Query / Tool / Task 三类核心抽象
- 独立命令系统
- 独立工具系统
- 上下文、历史和初始化模块

这和现代 coding agent 的标准形态高度一致。

### 5.2 分层图

可以把推断出来的 Claude Code 分成下面几层：

```text
entrypoints/cli.tsx
  -> screens/REPL.tsx / screens/ResumeConversation.tsx
    -> components/App.tsx + 大量终端 UI 组件
      -> state/* + hooks/*
        -> commands/*   (用户命令面)
        -> tools/*      (模型工具面)
          -> services/api/* / services/* / utils/*
            -> shell / fs / lsp / web / mcp / remote / plugins / skills
              -> permission / sandbox / analytics / memory / bridge / server
```

这个分层几乎说明 Claude Code 是一个“长会话 agent 平台”，而不是一个单函数 CLI。

---

## 6. 原始 Claude Code 的启动流程推断

### 6.1 入口层

`entrypoints/cli.tsx` 很可能是用户执行 `claude` 或类似二进制后进入的第一站。

配套文件还包括：

- `entrypoints/init.ts`
- `entrypoints/mcp.ts`
- `entrypoints/sandboxTypes.ts`
- `entrypoints/sdk/*`

据此可以推断启动阶段至少会做这些事：

1. 初始化环境与配置
2. 注册 CLI 传输与输出机制
3. 加载沙箱模式定义
4. 初始化 MCP 相关入口
5. 装配 SDK schema / control schema

### 6.2 界面装配层

以下文件非常关键：

- `screens/REPL.tsx`
- `screens/ResumeConversation.tsx`
- `components/App.tsx`
- `components/BaseTextInput.tsx`

这说明主产品形态是：

- 终端内 REPL 对话界面
- 支持恢复历史会话
- 通过组件树管理输入框、状态条、提示框、通知、工具结果、进度行

### 6.3 全局状态层

`state` 子系统包含：

- `AppState.tsx`
- `AppStateStore.ts`
- `store.ts`
- `selectors.ts`
- `onChangeAppState.ts`

这非常像一个集中式 UI + 会话状态容器。合理推断运行时至少维护：

- 当前对话/会话状态
- 正在运行的 agent / 子 agent 状态
- 工具调用状态
- 权限审批状态
- UI 展示状态
- 远程/桥接/插件连接状态

---

## 7. 用户输入到模型输出的主回路推断

### 7.1 用户输入进入系统

推断路径：

1. 用户在 REPL 输入文本
2. 输入先经过组件层和 hooks 层
3. 识别这是普通自然语言输入还是 slash command
4. 更新 AppState
5. 决定进入命令处理流程或模型回路

证据包括：

- `components/BaseTextInput.tsx`
- `components/ContextSuggestions.tsx`
- `hooks/fileSuggestions.ts`
- `hooks/unifiedSuggestions.ts`
- `types/textInputTypes.ts`

### 7.2 如果是 slash command

`commands/*` 很明显是用户命令面。典型命令包括：

- `review`
- `plan`
- `resume`
- `rewind`
- `context`
- `permissions`
- `model`
- `mcp`
- `plugin`
- `session`
- `stats`
- `status`
- `usage`
- `theme`
- `voice`
- `vim`

所以 slash command 的职责很可能是：

- 改变会话模式
- 改变模型或推理强度
- 查看/修改上下文
- 配置权限和沙箱
- 管理插件/MCP/远程能力
- 触发专门工作流，如 review、autofix-pr、security-review

### 7.3 如果是普通 prompt

推断路径是：

1. 构造系统提示词与会话上下文
2. 发送到 Claude API 客户端
3. 接收模型增量输出
4. 如果模型请求工具调用，则切换到工具执行分支
5. 工具结果回写对话上下文
6. 再次喂给模型，直到回合完成

证据包括：

- `services/api/claude.ts`
- `services/api/client.ts`
- `constants/prompts.ts`
- `constants/systemPromptSections.ts`
- `constants/xml.ts`
- `constants/turnCompletionVerbs.ts`
- `history.ts`
- `assistant/sessionHistory.ts`

这是一条典型的 tool-calling agent loop。

---

## 8. Command 系统的职责推断

### 8.1 Command 不是 Tool

这是 Claude Code 架构里非常关键的分层。

`commands/*` 是给“用户”直接触发的交互命令。

`tools/*` 是给“模型”在对话中调用的执行能力。

两者区别：

- command 由用户显式触发
- tool 由模型在 agent loop 中隐式调度

### 8.2 Command 面显示出的产品能力

从 207 条镜像命令、141 个唯一命令名可以看出，Claude Code 的 command 面非常厚。

重要族群包括：

- 会话管理：`session` `resume` `rewind` `rename`
- 上下文管理：`context` `compact` `memory` `files`
- 代码工作流：`review` `diff` `commit` `commit-push-pr` `autofix-pr`
- 计划与推理：`plan` `ultraplan` `effort`
- 工具与权限：`permissions` `sandbox-toggle` `hooks`
- 扩展系统：`mcp` `plugin` `reload-plugins` `skills`
- 环境与远程：`remote-setup` `remote-env` `bridge`
- 产品配置：`model` `output-style` `rate-limit-options` `privacy-settings`

这说明 command 层承担的是“操作面板 + 运维控制面 + 工作流入口”的角色。

---

## 9. Tool 系统的职责推断

### 9.1 Tool 面是 Claude Code 的核心执行层

从镜像数据看，工具族群非常完整，几乎就是一个 agent OS。

主要工具家族及镜像文件数量：

- `AgentTool`：20
- `BashTool`：18
- `PowerShellTool`：14
- `FileEditTool`：6
- `LSPTool`：6
- `BriefTool`：5
- `ConfigTool`：5
- `FileReadTool`：5
- `ScheduleCronTool`：5
- `WebFetchTool`：5

这说明模型不是只能“说”，而是能实际操作环境。

### 9.2 基础执行工具

基础能力包括：

- `BashTool`
- `PowerShellTool`
- `FileReadTool`
- `FileWriteTool`
- `FileEditTool`
- `GlobTool`
- `GrepTool`
- `NotebookEditTool`
- `LSPTool`

这组成了 coding agent 的基础工具体系：

- 读文件
- 写文件
- 局部编辑
- 搜索代码
- 查符号
- 运行 shell
- 处理 notebook

### 9.3 流程控制工具

还有一批明显不是“文件操作”，而是“运行时模式切换”的工具：

- `EnterPlanModeTool`
- `ExitPlanModeV2Tool`
- `EnterWorktreeTool`
- `ExitWorktreeTool`
- `TodoWriteTool`
- `TaskCreateTool`
- `TaskListTool`
- `TaskGetTool`
- `TaskUpdateTool`
- `TaskStopTool`

这意味着 Claude Code 内部存在明确的工作模式切换：

- 计划模式
- 执行模式
- 独立 worktree 模式
- 任务清单/任务生命周期管理

也就是说，它不是“随便输出下一句”，而是有内建 workflow discipline。

### 9.4 人机协作工具

明显体现交互式 agent 特征的工具有：

- `AskUserQuestionTool`
- `BriefTool`
- `SendMessageTool`
- `SyntheticOutputTool`

可推断 Claude Code 支持：

- 主动向用户追问
- 生成结构化摘要
- 向其他实体或代理发消息
- 生成某些合成输出供 UI/流程消费

### 9.5 多代理工具

最重要的一组证据来自：

- `AgentTool`
- `forkSubagent`
- `resumeAgent`
- `runAgent`
- `TeamCreateTool`
- `TeamDeleteTool`
- `spawnMultiAgent`
- `components/CoordinatorAgentStatus.tsx`
- `hooks/toolPermission/handlers/swarmWorkerHandler.ts`
- `coordinator/coordinatorMode.ts`

这几乎可以直接下结论：

- Claude Code 内部存在多代理/子代理机制
- 至少支持 fork、resume、run 子 agent
- 存在 team / swarm / coordinator 概念
- UI 有专门的协调器状态展示

所以它不只是单代理 coding assistant，而是一个可扩展到代理编排的系统。

### 9.6 MCP 与外部资源工具

相关工具包括：

- `MCPTool`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- `McpAuthTool`

相关命令包括：

- `mcp`
- `addCommand`
- `xaaIdpCommand`

这说明 Claude Code 的 MCP 能力是比较正式的，不只是“兼容一下协议”，而是包含：

- 资源枚举
- 资源读取
- 认证
- 命令面管理

### 9.7 Web 与远程工具

相关工具包括：

- `WebFetchTool`
- `WebSearchTool`
- `RemoteTriggerTool`

相关子系统包括：

- `remote/*`
- `server/*`
- `upstreamproxy/*`
- `bridge/*`

这说明 Claude Code 很可能支持：

- 外网检索/抓取
- 远程触发
- 远程 session 管理
- 代理桥接和直连模式

---

## 10. 权限、沙箱与安全治理推断

这是这个架构里另一个很重的部分。

### 10.1 直接证据

文件名里明确出现了：

- `hooks/toolPermission/PermissionContext.ts`
- `hooks/toolPermission/handlers/coordinatorHandler.ts`
- `hooks/toolPermission/handlers/interactiveHandler.ts`
- `hooks/toolPermission/handlers/swarmWorkerHandler.ts`
- `hooks/toolPermission/permissionLogging.ts`
- `tools/BashTool/bashPermissions.ts`
- `tools/BashTool/bashSecurity.ts`
- `tools/BashTool/destructiveCommandWarning.ts`
- `tools/BashTool/readOnlyValidation.ts`
- `tools/BashTool/shouldUseSandbox.ts`
- `tools/PowerShellTool/powershellPermissions.ts`
- `tools/PowerShellTool/powershellSecurity.ts`
- `entrypoints/sandboxTypes.ts`
- `commands/sandbox-toggle/*`

### 10.2 推断出的权限模型

Claude Code 很可能不是单一权限开关，而是按上下文分层：

- 交互模式审批
- coordinator 模式审批
- swarm worker 模式审批
- shell / PowerShell 各自的安全策略
- destructive command 单独警告
- read-only 模式校验
- sandbox 选择逻辑

这说明权限系统不是附属功能，而是整个运行时的核心治理层。

### 10.3 可能的执行链

模型请求调用 `BashTool` 时，合理推断会经过：

1. 工具参数校验
2. 路径合法性检查
3. 命令语义分析
4. 是否 destructive 判定
5. 当前模式权限判定
6. 是否进入 sandbox 判定
7. 必要时 UI 弹框或审批
8. 执行命令
9. 结果格式化回传
10. 权限/审计日志记录

这套结构与现代 coding agent 的高风险工具防护高度一致。

---

## 11. 上下文、记忆和会话恢复推断

### 11.1 会话历史

直接证据：

- `assistant/sessionHistory.ts`
- `history.ts`
- `screens/ResumeConversation.tsx`
- `commands/resume/*`
- `commands/rewind/*`
- `commands/rename/generateSessionName.ts`

这说明 Claude Code 有明确的会话对象：

- 可恢复
- 可回滚
- 可重命名
- 有历史管理

### 11.2 Session Memory

直接证据：

- `services/SessionMemory/sessionMemory.ts`
- `services/SessionMemory/sessionMemoryUtils.ts`
- `services/SessionMemory/prompts.ts`

这说明系统并不只依赖“当前上下文窗口”，还可能对会话级记忆做：

- 提炼
- 存储
- 注入 prompt
- 供后续检索

### 11.3 Memdir

`memdir/*` 这组文件很有价值：

- `findRelevantMemories.ts`
- `memoryScan.ts`
- `memoryAge.ts`
- `teamMemPaths.ts`
- `teamMemPrompts.ts`

从命名判断，Claude Code 很可能还有一个文件化或目录化的 memory layer：

- 扫描记忆条目
- 按年龄排序或衰减
- 选择相关记忆注入当前回合
- 团队代理之间也可能共享或分离记忆路径

这说明它的上下文工程不是纯 prompt 拼接，而是带检索和生命周期管理的。

---

## 12. 插件、技能与扩展系统推断

### 12.1 Plugin

证据非常强：

- `commands/plugin/*` 家族有 17 个镜像文件
- 包括 marketplace、settings、trust warning、validate、options flow
- `plugins/builtinPlugins.ts`
- `plugins/bundled/index.ts`

这表明插件系统不是玩具，而是正式产品能力，至少包含：

- 插件发现
- 插件安装/管理
- market 管理
- 插件配置
- 插件信任提示
- 插件校验

### 12.2 Skill

相关文件包括：

- `SkillTool`
- `skills/loadSkillsDir.ts`
- `skills/bundled/*`
- `skills/mcpSkillBuilders.ts`
- `commands/skills/*`

这里很像“将特定行为模式、prompt 模板或工作流打包成技能”的设计。

从 bundled skill 名称看，系统内置了：

- `remember`
- `verify`
- `simplify`
- `batch`
- `updateConfig`
- `scheduleRemoteAgents`

因此技能层很可能位于：

- prompt 组装层之上
- tool 调用策略层之下

它提供的是“专家模式模板”而不是基础 I/O 能力。

---

## 13. 远程、桥接与多端接入推断

这是这个系统和普通本地 CLI 拉开差距的地方。

### 13.1 Remote 子系统

文件包括：

- `remote/RemoteSessionManager.ts`
- `remote/SessionsWebSocket.ts`
- `remote/remotePermissionBridge.ts`
- `remote/sdkMessageAdapter.ts`

说明至少存在：

- 远程 session 管理器
- WebSocket 通道
- 权限桥接
- SDK 消息适配

### 13.2 Bridge 子系统

`bridge` 有 31 个模块，sample file 包括：

- `bridgeMain.ts`
- `bridgeMessaging.ts`
- `bridgePermissionCallbacks.ts`
- `createSession.ts`
- `inboundMessages.ts`
- `initReplBridge.ts`
- `remoteBridgeCore.ts`
- `replBridge.ts`

这基本可以推断：

- Claude Code 支持把本地 REPL 和外部通道桥接起来
- 桥接通道可能支持消息透传、附件传入、权限回调、会话创建和状态同步

### 13.3 CLI 传输层

`cli/transports/*` 包含：

- `HybridTransport`
- `SSETransport`
- `WebSocketTransport`
- `SerialBatchEventUploader`
- `WorkerStateUploader`

这表明即便是 CLI 入口，内部也不是单纯 stdout/stderr，而是设计了更抽象的事件传输层。

合理推断用途包括：

- 本地终端渲染
- 远端控制台或 IDE 集成
- 事件上传
- worker 状态同步

---

## 14. UI、Hooks 与终端体验层推断

### 14.1 组件规模

`components` 子系统有 `389` 个模块，远高于“一个简单 CLI”应有的复杂度。

这说明 Claude Code 的终端体验很重。

### 14.2 可见的 UI 能力

从 sample file 能直接看出的能力：

- 自动更新
- Bash 模式进度
- Bridge 对话框
- API Key 审批
- Context 可视化
- 紧凑摘要
- 快捷键提示
- OAuth 控制台流程
- 各类通知与状态框

这表明 Claude Code 不是“只打印文本”，而是有完整的 TUI 产品设计。

### 14.3 Hooks 层

`hooks` 有 `104` 个模块，重要线索包括：

- 通知 hooks
- MCP 连通性通知
- LSP 初始化通知
- 插件安装状态通知
- 模型迁移通知
- tool permission hooks

这说明 UI 与状态变化之间通过 hooks 高度解耦，产品行为不是硬编码在单点逻辑里。

---

## 15. API、分析与产品运营层推断

### 15.1 API 层

`services/api/*` 包括：

- `bootstrap.ts`
- `claude.ts`
- `client.ts`
- `adminRequests.ts`
- `dumpPrompts.ts`
- `errors.ts`

因此可以推断：

- Claude Code 有专门的 API client 封装
- prompt 构造与导出可能是可调试的
- 有 bootstrap 阶段的远端初始化
- 错误处理有独立抽象

### 15.2 Analytics 层

相关文件包括：

- `analytics/datadog.ts`
- `analytics/firstPartyEventLogger.ts`
- `analytics/growthbook.ts`
- `analytics/sink.ts`
- `analytics/sinkKillswitch.ts`

说明产品内置：

- 埋点上报
- 实验/feature flag
- 第一方事件记录
- sink 开关与熔断

这不是黑客脚本，而是产品化系统。

---

## 16. 一个更贴近真实的 Claude Code 回合时序图

下面是我根据元数据反推出的单轮运行序列。

### 16.1 交互轮次

1. 用户在 REPL 输入消息或 slash command
2. UI 组件层接收输入并更新 `AppState`
3. hooks 生成建议、提示、通知或上下文增强
4. 如果是 slash command，路由到 `commands/*`
5. 如果是普通消息，进入 prompt 组装流程
6. 系统从：
   - 会话历史
   - session memory
   - memdir
   - 当前工作目录上下文
   - 技能/模式设定
   中提取上下文
7. `services/api/claude.ts` 或 `client.ts` 发起模型调用
8. 模型流式返回文本或 tool call
9. 若触发 tool：
   - 先做 schema / mode / permission / sandbox 校验
   - 再实际执行 shell / file / web / mcp / team / task 等操作
10. 工具输出进入 history / state / UI
11. 如有必要，工具结果再回灌模型
12. 回合完成后更新：
   - 会话历史
   - 记忆
   - 任务状态
   - analytics
   - 远程/桥接状态

### 16.2 多代理轮次

如果模型选择的是 `AgentTool` 或 team 相关工具，序列还会扩展：

1. 主代理创建子代理或团队
2. coordinator 进入编排模式
3. worker 获得各自上下文/权限
4. worker 使用子集工具执行任务
5. 结果通过消息/状态通道回传
6. 主代理汇总并继续主回路

这类结构和“单次 ChatCompletion”已经完全不是一个复杂度。

---

## 17. 当前仓库对理解 Claude Code 的真正价值

虽然它不是原始源码，但它仍然很有价值，因为它保留了三类关键信息：

### 17.1 产品表面积

通过 `commands_snapshot.json` 和 `tools_snapshot.json`，可以知道这个系统对外暴露了哪些能力。

### 17.2 内部模块边界

通过 `subsystems/*.json`，可以看出哪些模块是大头：

- `utils`: 564
- `components`: 389
- `services`: 130
- `hooks`: 104

这说明原始 Claude Code 的复杂度主要集中在：

- 基础设施工具箱
- 终端 UI
- 服务层逻辑
- 状态响应与权限逻辑

### 17.3 核心设计哲学

从结构上看，Claude Code 的核心哲学不是“让模型自动写代码”这么简单，而是：

- 用命令系统给用户强控制
- 用工具系统给模型强执行
- 用权限系统把高风险动作拦住
- 用状态/UI/hooks 把长会话体验做好
- 用 memory/skills/plugins/MCP 把系统变成可扩展平台

---

## 18. 不能过度下结论的地方

由于当前仓库没有保留原始 TypeScript 实现体，以下结论不能说死：

- 具体 prompt 模板内容
- API 请求/响应 schema 的全部细节
- 精确的 tool call 协议格式
- 子代理调度算法
- 真正的上下文裁剪和压缩策略
- 每个 command 的最终用户语义
- sandbox 的完整实现细节

也就是说，本文能高可信回答的是：

- 系统有哪些层
- 每层大概负责什么
- 主回路如何流转

但不能高可信逐行复原：

- 原始实现代码
- 每个函数的细节行为

---

## 19. 最终结论

基于当前仓库，Claude Code 可以被理解成一个五层复合系统：

1. **终端产品层**
   - `entrypoints` `screens` `components` `hooks` `state`
2. **用户控制层**
   - `commands/*`
3. **模型执行层**
   - `tools/*`
4. **基础设施与治理层**
   - `services/*` `utils/*` `constants/*` `permissions` `sandbox`
5. **扩展与分布式能力层**
   - `plugins` `skills` `MCP` `remote` `bridge` `team/subagent`

它的核心运行逻辑不是“用户提问 -> 模型回答”，而是：

`用户输入 -> 命令或对话路由 -> 上下文/状态装配 -> 模型调用 -> 工具执行 -> 权限审查 -> 状态/历史更新 -> 继续下一轮`

如果再加上子代理、MCP、remote、plugin，它就进一步变成：

`一个带 UI、状态机、权限系统和可扩展执行面的 agent runtime`

这就是我基于当前仓库代码和镜像元数据，对 Claude Code 运行逻辑做出的最完整且尽量保守的拆解。

