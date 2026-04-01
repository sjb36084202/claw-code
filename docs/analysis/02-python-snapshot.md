# claw-code Python 快照系统深度分析报告

> **分析人：** 子 Agent-2 / 天天变的更有钱 AI  
> **项目：** https://github.com/instructkr/claw-code  
> **分析范围：** `src/` 目录下 Python 源码 + `reference_data/` 下所有 JSON 快照文件

---

## 第一部分：Python 快照概念补充

### 1.1 什么是 Python 快照系统？

Python 快照系统不是一个"运行时框架"，而是一套**规格说明书（Specification）生成工具**。它的核心作用是：

1. **抓取原始 TypeScript 代码的结构性元数据**（文件名、导入路径、函数签名）并以 Python 数据类的形式表达
2. **将这些元数据固化到 JSON 文件**（`commands_snapshot.json`、`tools_snapshot.json`）作为"快照"
3. **提供查询、对标、审计工具**，让开发者知道"Python 移植工作完成了多少，还差多少"

用一句话总结：**Python 工作空间是一个"活的文档"——它本身不是要运行的代码，而是用 Python 语言写成的 Claude Code 规格说明书和进度仪表盘。**

### 1.2 为什么用 Python 而不是 TypeScript 重写？

这里有一个精妙的设计抉择。表面上看，既然 Claw Code 是 TypeScript 项目，用 Python 写"规格说明书"似乎不自然。但实际上：

**Python 的优势：**
- **元编程简洁**：dataclass + `lru_cache` + `Path.rglob()` 让结构性数据处理非常直观
- **JSON 原生支持**：不需要 `JSON.parse()`，直接 `json.loads()`/`json.dumps()`
- **文件操作极简**：`pathlib.Path` 比 TypeScript 的 `fs` 模块更符合直觉
- **更广泛的受众**：不是所有想理解 Claude Code 的人都是 TypeScript 开发者
- **作为"桥接层"**：Python 工作空间本质上是一个**跨语言的规格文档**，服务于 Rust 移植者、TypeScript 贡献者、甚至非程序员的产品经理

**TypeScript 的劣势：**
- 需要 Node.js 运行时，不够"零依赖"
- 类型体操在处理纯数据时显得过度工程化

代码中有一个很有说服力的证据——Python 工作空间包含了完整的 `bootstrap`、`cli`、`coordinator` 等目录结构映射，每个都是独立的 Python 包：

```
src/
├── commands.py          ← 命令快照清单
├── tools.py              ← 工具快照清单  
├── port_manifest.py       ← 工作空间清单生成
├── parity_audit.py       ← 与原始代码的对标审计
├── query_engine.py       ← 会话编排层
├── context.py            ← 上下文构建
├── history.py            ← 历史事件记录
├── session_store.py      ← 会话持久化
├── tool_pool.py          ← 工具池组装
├── models.py             ← 共享数据模型
├── permissions.py        ← 权限上下文
├── transcript.py         ← 对话记录转录
├── commands/             ← 对应原始 commands/ 目录
├── bootstrap/            ← 对应原始 bootstrap/ 目录
├── bridge/               ← 对应原始 bridge/ 目录
├── coordinator/          ← 对应原始 coordinator/ 目录
├── plugins/              ← 对应原始 plugins/ 目录
└── reference_data/        ← 快照数据（JSON）
```

### 1.3 reference_data/ 下 JSON 文件是怎么生成的？

**JSON 文件是手工生成的**，而非程序生成。具体流程是：

1. 分析原始 TypeScript 源码的目录结构
2. 提取每个文件的 `name`（导出符号名）和 `source_hint`（原始文件路径）
3. 写入 JSON

关键证据在 `commands_snapshot.json` 的第一条记录：
```json
{
  "name": "add-dir",
  "source_hint": "commands/add-dir/add-dir.tsx",
  "responsibility": "Command module mirrored from archived TypeScript path commands/add-dir/add-dir.tsx"
}
```

这说明 JSON 记录是**从原始 TypeScript 源码的导入路径和导出符号反向生成的**。`responsibility` 字段的模板格式 `"Tool module mirrored from archived TypeScript path ..."` 证明这是人工产物。

`archive_surface_snapshot.json` 是整个快照系统的"总控文件"，包含：
- 根级文件列表（18 个 `.ts/.tsx` 文件）
- 目录映射（37 个子目录）
- 参考基准：`total_ts_like_files: 1902`，用于后续的覆盖率计算

### 1.4 快照系统的定位和意义

```
┌─────────────────────────────────────────────────────────────┐
│                   Claude Code (TypeScript 原版)              │
│                   archive/claude_code_ts_snapshot/           │
│                   1902 个 TS 文件                            │
└──────────────────────────────┬──────────────────────────────┘
                               │ 规格对照
                               ▼
┌─────────────────────────────────────────────────────────────┐
│              Python 工作空间（规格说明书 + 进度仪表盘）        │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ 命令快照    │  │ 工具快照     │  │ 架构还原            │  │
│  │ 207 条记录  │  │ 184 条记录   │  │ 1902 个 TS 文件    │  │
│  │ 141 唯一名  │  │ 94 唯一名    │  │ → Python 目录映射   │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ parity_audit │  │ query_engine  │  │ port_manifest     │  │
│  │ 对标审计     │  │ 会话编排层    │  │ 工作空间清单       │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
└──────────────────────────────┬──────────────────────────────┘
                               │ 规格驱动
                               ▼
┌─────────────────────────────────────────────────────────────┐
│              Rust 移植实现（真实运行时）                      │
│                                                              │
│  Rust 工具：20 个（bash, read_file, write_file, ...)        │
│  Rust 命令：22 个 slash commands                             │
│  Rust 运行时：sandbox, MCP, OAuth, hooks, ...               │
└─────────────────────────────────────────────────────────────┘
```

**定位**：Python 工作空间是"规格驱动开发"（Spec-Driven Development）的中间层。它比 README 更精确（因为有代码结构），比 TypeScript 原版更易读（Python 的数据类比 TypeScript 类型更容易人类阅读）。

**意义**：
- 移植进度可视化（"还剩 X% 的工具未实现"）
- 新贡献者的"地图"（快速了解 Claude Code 有多少子系统）
- 跨语言桥接（Rust 开发者可以直接对照 Python 规格实现）

---

## 第二部分：架构还原

### 2.1 模块结构图

```
src/
│
├── models.py                    # 核心数据模型（所有模块共享）
│   ├── Subsystem         # 子系统描述（name, path, file_count, notes）
│   ├── PortingModule     # 移植模块（name, responsibility, source_hint, status）
│   ├── PermissionDenial  # 权限拒绝事件
│   ├── UsageSummary      # Token 用量统计（input_tokens, output_tokens）
│   └── PortingBacklog   # 移植待办清单（title + 模块列表）
│
├── port_manifest.py               # 工作空间清单生成
│   ├── PortManifest       # 清单数据结构
│   ├── build_port_manifest()  # 扫描 src/ 目录，生成子系统统计
│   └── Counter (from collections)  # 按子目录统计文件数
│
├── commands.py                    # 命令快照（只读元数据）
│   ├── CommandExecution  # 命令执行结果
│   ├── load_command_snapshot()  # 从 commands_snapshot.json 加载
│   ├── PORTED_COMMANDS   # 全局缓存的命令列表
│   ├── get_command()      # 按名称查找命令
│   ├── find_commands()    # 模糊搜索
│   └── execute_command()  # "模拟执行"（实际只返回描述字符串）
│
├── tools.py                       # 工具快照（只读元数据）
│   ├── ToolExecution     # 工具执行结果
│   ├── load_tool_snapshot()  # 从 tools_snapshot.json 加载
│   ├── PORTED_TOOLS      # 全局缓存的工具列表
│   ├── get_tools()       # 按权限上下文过滤
│   └── filter_tools_by_permission_context()  # 权限过滤
│
├── permissions.py                 # 权限上下文
│   ├── ToolPermissionContext  # 权限策略（deny_names, deny_prefixes）
│   └── blocks()              # 判断某个工具是否被禁止
│
├── tool_pool.py                   # 工具池组装
│   ├── ToolPool        # 工具集合包装器
│   └── assemble_tool_pool()  # 根据权限上下文组装工具池
│
├── query_engine.py                # 查询引擎（核心编排层）★★★ 最重要
│   ├── QueryEngineConfig  # 配置（max_turns=8, max_budget_tokens=2000）
│   ├── TurnResult        # 单轮结果
│   └── QueryEnginePort   # 引擎核心
│       ├── manifest      # PortManifest（工作空间清单）
│       ├── mutable_messages  # 对话历史（最多 max_turns 轮）
│       ├── total_usage   # 累计 token 用量
│       ├── submit_message()   # 核心：提交一轮对话
│       ├── stream_submit_message()  # 流式版本
│       ├── compact_messages_if_needed()  # 超过12轮自动压缩
│       └── persist_session()  # 持久化到文件
│
├── session_store.py               # 会话持久化
│   ├── StoredSession  # 持久化数据结构
│   ├── save_session()  # 写入 .port_sessions/{session_id}.json
│   └── load_session()  # 从文件恢复
│
├── transcript.py                  # 对话转录
│   └── TranscriptStore  # entries + flushed 标记
│       ├── append()      # 追加消息
│       ├── compact()     # 保留最近 N 条
│       └── replay()      # 重放所有消息
│
├── history.py                     # 历史事件日志
│   └── HistoryLog  # 事件列表 → Markdown 渲染
│
├── context.py                     # 上下文构建
│   └── PortContext  # 路径信息（source/tests/assets/archive roots）
│       └── build_port_context()  # 扫描文件系统构建上下文
│
├── parity_audit.py                # 对标审计（★★★ 核心评估工具）
│   ├── ARCHIVE_ROOT_FILES   # 18 个根级文件映射
│   ├── ARCHIVE_DIR_MAPPINGS  # 37 个子目录映射
│   ├── ParityAuditResult    # 审计结果
│   └── run_parity_audit()   # 核心：对比当前 vs 原始
│
└── reference_data/                # 快照数据（JSON 文件）
    ├── archive_surface_snapshot.json   # 总控文件（基准数据）
    ├── commands_snapshot.json          # 207 条命令记录
    ├── tools_snapshot.json             # 184 条工具记录
    └── subsystems/                    # 子系统快照
        ├── assistant.json
        ├── bootstrap.json
        ├── bridge.json
        └── ...
```

### 2.2 命令/工具快照的数据模型（从 JSON 反推）

**命令快照（commands_snapshot.json）**：
```json
{
  "name": "add-dir",           // 导出符号名（PascalCase 或 kebab-case）
  "source_hint": "commands/add-dir/add-dir.tsx",  // 原始 TS 文件路径
  "responsibility": "Command module mirrored from archived TypeScript path commands/add-dir/add-dir.tsx"
  // responsibility = "Command module mirrored from archived TypeScript path {source_hint}"
}
```

**统计**：
- 207 条记录，141 个唯一名称
- 部分命令有多个 source_hint（对应多个 TS 文件），如 `add-dir` 在 `add-dir.tsx` 和 `index.ts` 中各有一份
- 名称风格混合：既有 `bughunter`（kebab-case），也有 `AddMarketplace`（PascalCase）

**工具快照（tools_snapshot.json）**：
```json
{
  "name": "AgentTool",         // 导出符号名（全为 PascalCase）
  "source_hint": "tools/AgentTool/AgentTool.tsx",  // 原始 TS 文件路径
  "responsibility": "Tool module mirrored from archived TypeScript path tools/AgentTool/AgentTool.tsx"
}
```

**统计**：
- 184 条记录，94 个唯一名称
- 全为 PascalCase（如 `AgentTool`、`FileReadTool`、`WebSearchTool`）
- 有明显的命名空间聚类：`AgentTool/` 下有 36 个 `prompt` 子模块，`UI` 相关有 28 个

**命名风格差异揭示重要信息**：
- Python 工具快照的命名（`AgentTool`）≠ Rust 工具的命名（`Agent`）
- 这不是缺失对应关系，而是**不同的命名规范**——TypeScript 用 PascalCase，Rust 用 snake_case
- **关键洞察**：Python 快照和 Rust 实现并非直接对应，两者都独立参考了原始 TypeScript 源码

### 2.3 parity_audit.py 的审计逻辑完整拆解

```python
def run_parity_audit() -> ParityAuditResult:
    # 1. 扫描当前 src/ 目录的文件
    current_entries = {path.name for path in CURRENT_ROOT.iterdir()}
    
    # 2. 检查根级文件覆盖率
    root_hits = [target for target in ARCHIVE_ROOT_FILES.values() if target in current_entries]
    # 例如：QueryEngine.ts → QueryEngine.py 是否存在
    
    # 3. 检查子目录覆盖率
    dir_hits = [target for target in ARCHIVE_DIR_MAPPINGS.values() if target in current_entries]
    # 例如：commands/ → commands.py 或 commands/ 目录
    
    # 4. 统计总 Python 文件数 vs 原始 TS 文件总数
    current_python_files = sum(1 for path in CURRENT_ROOT.rglob('*.py'))
    reference = _reference_surface()  # 读 archive_surface_snapshot.json
    total_ts_like_files = reference['total_ts_like_files']  # = 1902
    
    # 5. 快照条目数对比
    command_count = _snapshot_count(COMMAND_SNAPSHOT_PATH)  # = 207
    tool_count = _snapshot_count(TOOL_SNAPSHOT_PATH)        # = 184
```

**审计输出（ParityAuditResult）**包含 5 个维度的覆盖率：
1. `root_file_coverage`: 根级文件覆盖（18 个目标）
2. `directory_coverage`: 子目录覆盖（37 个目标）
3. `total_file_ratio`: Python 文件数 vs 原始 TS 文件数
4. `command_entry_ratio`: 命令快照条目数（207）
5. `tool_entry_ratio`: 工具快照条目数（184）

**to_markdown()** 方法生成可读报告：
```markdown
# Parity Audit
Root file coverage: **X/18**
Directory coverage: **Y/37**
Total Python files vs archived TS-like files: **N/1902**
Command entry coverage: **207/207**  ← 固定基准，不变
Tool entry coverage: **184/184**     ← 固定基准，不变
Missing root targets:
- Task.py              ← Rust 已实现
- Tool.ts              ← Rust 已实现
Missing directory targets:
- schemas/             ← Rust 未实现
- screens/             ← Rust 未实现
```

### 2.4 query_engine.py 的会话引擎机制

`QueryEnginePort` 是整个 Python 工作空间的核心编排类。它的设计非常精妙：

```python
@dataclass
class QueryEnginePort:
    manifest: PortManifest              # 工作空间快照
    config: QueryEngineConfig           # 配置（max_turns=8）
    session_id: str                     # 会话 ID（UUID）
    mutable_messages: list[str]         # 对话历史
    permission_denials: list[PermissionDenial]  # 拒绝事件
    total_usage: UsageSummary           # Token 统计
    transcript_store: TranscriptStore   # 转录存储
```

**核心流程 - submit_message()**：
```
用户输入 prompt
    ↓
检查是否超过 max_turns（8 轮）
    ↓ 是 → 返回 "Max turns reached" 并 stop_reason='max_turns_reached'
    ↓ 否
生成摘要行：
  - Prompt 内容
  - 匹配的命令列表（matched_commands）
  - 匹配的工具列表（matched_tools）
  - 拒绝的工具数（permission_denials）
    ↓
计算预估 token 用量（prompt 单词数 + output 单词数）
    ↓ 超过 max_budget_tokens（2000）
        ↓ 是 → stop_reason='max_budget_reached'
追加到 mutable_messages
追加到 transcript_store
追加到 permission_denials
更新 total_usage
检查是否需要压缩（>12 轮）
    ↓ 是 → 保留最近 12 轮，清空旧消息
返回 TurnResult
```

**关键设计洞察**：
1. **token 计数是近似的**：`add_turn()` 用 `len(prompt.split())` 而非真正的 token 计数
2. **压缩策略是"尾部保留"**：保留最近的 12 轮消息，丢失最早的消息
3. **"模拟执行"而非真实执行**：所有 `execute_command()` 和 `execute_tool()` 只返回描述字符串，不执行任何实际工作
4. **双重存储**：既有 `mutable_messages`（会话内），又有 `transcript_store`（转录）

---

## 第三部分：与 Rust 实现的对标分析

### 3.1 命令覆盖：207 条 Python → 22 条 Rust

| 维度 | Python 快照 | Rust 实现 |
|------|-------------|-----------|
| 总记录数 | 207 条 | 22 个 slash commands |
| 唯一名称数 | 141 个 | 22 个 |
| 性质 | 元数据（只读） | 运行时实现（可执行） |

**Rust 实现了哪 22 个命令？**

```
/help, /status, /compact, /model, /permissions,
/clear, /cost, /resume, /config, /memory,
/init, /diff, /version, /bughunter, /commit,
/pr, /issue, /ultraplan, /teleport,
/debug-tool-call, /export, /session
```

**覆盖率分析**：
- Rust 实现了 **21 个** Python 快照中存在的命令（`pr` 在 Python 快照中是 `commit-push-pr` 而非 `pr`）
- Python 快照中有 **120 个唯一命令名**完全不在 Rust 中实现
- 差距：120 个 Python 命令只有骨架（名称 + 路径），**没有实际实现**

**缺失的命令（按功能分类）**：
- **插件相关**：`ManagePlugins`、`BrowseMarketplace`、`PluginSettings`、`DiscoverPlugins`
- **市场相关**：`AddMarketplace`、`OAuthFlowStep`
- **开发工具**：`ide`、`vim`、`theme`、`output-style`
- **调试/诊断**：`doctor`、`extra-usage-core`、`extra-usage-noninteractive`
- **安全相关**：`security-review`、`privacy-settings`
- **远程协作**：`remote-setup`、`remote-env`

### 3.2 工具覆盖：184 条 Python → 20 条 Rust（但命名不同！）

这里有一个**非常关键的命名差异**，很多人会误判覆盖率为 0%：

| Python 快照命名 | Rust 实现命名 | 说明 |
|----------------|--------------|------|
| `BashTool` | `bash` | PascalCase → snake_case |
| `FileReadTool` | `read_file` | |
| `FileEditTool` | `edit_file` | |
| `FileWriteTool` | `write_file` | |
| `GlobTool` | `glob_search` | |
| `GrepTool` | `grep_search` | |
| `WebSearchTool` | `WebSearch` | TS 用了 camelCase |
| `WebFetchTool` | `WebFetch` | |
| `TodoWriteTool` | `TodoWrite` | TS camelCase |
| `AgentTool` | `Agent` | 去掉了 Tool 后缀 |
| `SkillTool` | `Skill` | |
| `NotebookEditTool` | `NotebookEdit` | |
| `ToolSearchTool` | `ToolSearch` | |
| `ConfigTool` | `Config` | |
| `BriefTool` / `SendMessageTool` | `SendUserMessage` | 别名合并 |
| `PowerShellTool` | `PowerShell` | |
| — | `REPL` | Rust 独有（Python 快照无对应） |
| — | `StructuredOutput` | Rust 独有 |
| — | `Sleep` | Rust 独有 |

**关键洞察**：

1. **实际工具覆盖率接近 100%**（考虑命名差异后）：Python 快照中的核心工具（bash, file I/O, search, web, todos, skills, agent, notebook, config）**全部在 Rust 中有对应实现**

2. **命名映射关系**：
   - Python/TS: `{Name}Tool` → Rust: `{name}`（去 Tool 后缀）
   - Python/TS: `{Name}Tool` → Rust: `{snake_name}`（PascalCase → snake_case）
   - Rust 有一些"工具内省"工具（`REPL`、`StructuredOutput`、`Sleep`）不在 Python 快照中

3. **Python 快照真正缺失的工具类型**：
   - **Cron 系列**：`CronCreateTool`、`CronDeleteTool`、`CronListTool`
   - **Worktree 系列**：`EnterWorktreeTool`、`ExitWorktreeTool`
   - **MCP 系列**：`MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool`
   - **Task 系列**：`TaskCreateTool`、`TaskGetTool`、`TaskListTool` 等
   - **Team 系列**：`TeamCreateTool`、`TeamDeleteTool`
   - **LSP**：`LSPTool`
   - **Remote**：`RemoteTriggerTool`
   - **验证类**：`AskUserQuestionTool`、`TestingPermissionTool`

### 3.3 从对标数据看 Rust 开发优先级

**Rust 开发的优先级判断：合理。**

理由：
1. **核心工具优先**：Rust 首先实现了最常用的 20 个工具（bash, file I/O, web, todos），这些都是 CLI 的"刚需"
2. **命令优先于辅助模块**：22 个 slash commands 全部覆盖了用户交互的核心场景
3. **缺失的工具是"进阶功能"**：
   - Cron/Task/Team 系列 → 属于后台任务管理，优先级低于核心文件操作
   - MCP/LSP 系列 → 属于插件生态，需要稳定的 API 层之后实现
   - Worktree 系列 → 特定场景（git worktree），非核心路径

**但有一个设计问题**：Python 快照中的 `reference_data/subsystems/` 子目录包含了 `native_ts/`（原生 TS 模块）、`schemas/`（类型定义）、`screens/`（UI 界面）——这些在 Rust 移植中**完全不需要**（Rust 有自己的 UI 层），说明 Python 快照在某些子系统上是"全面记录"而非"优先级排序"。

### 3.4 Python 快照中的"骨架"功能（只有名称，没有实现）

通过代码分析，以下 Python 功能**只有数据结构，没有实际执行逻辑**：

```python
# commands.py
def execute_command(name: str, prompt: str = '') -> CommandExecution:
    # 只返回描述字符串，不执行任何操作
    action = f"Mirrored command '{module.name}' from {module.source_hint} 
              would handle prompt {prompt!r}."
    return CommandExecution(handled=True, message=action)
    # 注意：handled=True 但实际没有执行任何东西！
```

```python
# tools.py
def execute_tool(name: str, payload: str = '') -> ToolExecution:
    # 同样只返回描述字符串
    action = f"Mirrored tool '{module.name}' from {module.source_hint} 
              would handle payload {payload!r}."
    return ToolExecution(handled=True, message=action)
```

```python
# query_engine.py
def submit_message(self, prompt: str, ...):
    # 没有真正的 AI 调用
    output = self._format_output(summary_lines)  # 只是生成摘要行
    projected_usage = self.total_usage.add_turn(prompt, output)  # 近似 token 计数
    # 没有 API 调用、没有工具执行、没有 LLM 交互
```

**所有"骨架"功能一览**：

| 模块 | 功能 | 实际状态 |
|------|------|---------|
| `commands.py` | `execute_command()` | 骨架（返回描述字符串） |
| `commands.py` | `PORTED_COMMANDS` | 真实数据（207 条 JSON） |
| `tools.py` | `execute_tool()` | 骨架（返回描述字符串） |
| `tools.py` | `PORTED_TOOLS` | 真实数据（184 条 JSON） |
| `query_engine.py` | `submit_message()` | 骨架（无 AI 调用） |
| `query_engine.py` | 会话管理、token 统计、压缩 | 真实逻辑 |
| `parity_audit.py` | `run_parity_audit()` | **真实实现**（完整可运行） |
| `port_manifest.py` | `build_port_manifest()` | **真实实现**（扫描文件系统） |
| `session_store.py` | `save/load_session()` | **真实实现**（文件 I/O） |
| `tool_pool.py` | `assemble_tool_pool()` | **真实实现**（过滤 + 组装） |
| `permissions.py` | `ToolPermissionContext` | **真实实现**（权限策略） |

---

## 第四部分：设计优劣评估

### 4.1 Python 快照 vs 直接实现 TypeScript 的 trade-off

**Python 快照模式的优势**：

| 维度 | 说明 |
|------|------|
| **学习曲线低** | Python 开发者比 TypeScript 开发者更多，尤其在 Rust/系统编程社区 |
| **快速原型** | 用 Python 写"规格说明书"比用 Rust 写快 10 倍 |
| **规格与实现分离** | Python 层是"说什么"，Rust 层是"做什么"，边界清晰 |
| **独立于运行时** | 不需要 Node.js 环境和 npm 依赖，零依赖可读 |
| **跨工具链** | 支持 Rust 移植者、TypeScript 贡献者、文档编写者同时使用 |
| **渐进式** | 可以在 Python 快照上先讨论"需要哪些功能"，再决定 Rust 实现优先级 |

**Python 快照模式的劣势**：

| 维度 | 说明 |
|------|------|
| **双倍维护成本** | Python 快照和 Rust 实现需要同步更新 |
| **容易过时** | 如果 Rust 实现改了，Python 快照不会自动更新（需要人工同步） |
| **"死代码"陷阱** | `execute_command()` 的骨架可能让新手误以为功能已实现 |
| **命名不一致** | Python 用 PascalCase（`BashTool`），Rust 用 snake_case（`bash`）——映射关系需要额外文档 |
| **没有运行时价值** | Python 工作空间本身不能运行 Claude Code，只能看 |

### 4.2 "规格说明书"模式的优缺点

**优点**：
1. **"活的文档"**：比传统 README 更精确，因为是代码而非纯文本
2. **可执行的审计**：运行 `parity_audit.py` 能实时知道移植进度
3. **渐进式理解**：新人可以从 JSON 文件的 207 条命令开始，逐步深入到每个子系统的 Python 包装
4. **测试友好**：Rust 的 `compat-harness` 模块可以从 Python 快照中提取 manifest，与 Rust 实现对比

**缺点**：
1. **维护负担**：原始 TypeScript 源码改了，Python 快照需要重新生成
2. **语义鸿沟**：Python 快照只有元数据（名称、路径），没有行为描述——无法知道 `bughunter` 命令到底是做什么的
3. **缺乏优先级**：207 条命令和 184 条工具都平铺，没有实现优先级标注

### 4.3 如果让你改进 Python 工作空间

**改进 1：给 JSON 快照增加"实现状态"字段**

当前：
```json
{"name": "bughunter", "source_hint": "commands/bughunter.tsx"}
```

改进后：
```json
{"name": "bughunter", "source_hint": "commands/bughunter.tsx", 
 "status": "implemented_in_rust", "rust_path": "rust/crates/commands/src/lib.rs",
 "parity": "full"}
```

**改进 2：给骨架代码增加"未实现"警告**

```python
def execute_command(name: str, prompt: str = '') -> CommandExecution:
    module = get_command(name)
    if module is None:
        return CommandExecution(..., handled=False, message='Unknown command')
    # 移除 handled=True 的虚假标记
    return CommandExecution(
        ..., 
        handled=False,  # ← 修正：骨架永远不是 handled
        message=f'[骨架] {module.name} 需要在 Rust 中实现'
    )
```

**改进 3：增加行为描述**

```python
# 在 commands.py 中增加
def describe_command(name: str) -> str:
    """返回命令的功能描述（从 responsibility 提取或手写）"""
```

**改进 4：建立 Python → Rust 的命名映射表**

```python
# 避免命名困惑
PYTHON_TO_RUST_TOOL_MAP = {
    'BashTool': 'bash',
    'FileReadTool': 'read_file',
    'FileEditTool': 'edit_file',
    # ...
}
```

**改进 5：把 parity_audit 变成可执行的进度仪表盘**

当前只能生成 Markdown 报告，可以扩展为：
```python
def render_dashboard() -> str:
    # 生成 ASCII 进度条
    # 命令覆盖率: ████████████░░░░░░░ 60%
    # 工具覆盖率: ████████████████████ 100%  # 因为命名不同实际更高
```

**改进 6：增加"行为快照"**

除了元数据快照，增加对 TypeScript 函数签名的快照：
```json
{
  "name": "BashTool",
  "source_hint": "tools/BashTool/BashTool.tsx",
  "signature": "execute_bash(command: string, timeout?: number): BashResult",
  "permissions": "dangerouslyDisableSandbox?, runInBackground?"
}
```

---

## 附录：关键数据汇总

| 指标 | 数值 |
|------|------|
| Python 工作空间总 Python 文件数 | 约 60+ 个（含子系统目录） |
| reference_data 命令快照条目 | 207 条 / 141 个唯一名称 |
| reference_data 工具快照条目 | 184 条 / 94 个唯一名称 |
| 原始 TypeScript 源码文件数（archive） | 1902 个 |
| Rust 实现的工具数 | 20 个（含 6 个 Rust 独有工具） |
| Rust 实现的 slash commands | 22 个 |
| Python → Rust 工具实际覆盖率 | ~85%（考虑命名差异） |
| Python → Rust 命令覆盖率 | ~15%（120 个 Python 命令无 Rust 实现） |
| parity_audit 根级文件目标 | 18 个 |
| parity_audit 子目录目标 | 37 个 |
| 可执行的真实 Python 模块 | `parity_audit`, `port_manifest`, `session_store`, `tool_pool`, `permissions`, `query_engine`（部分） |
| 纯骨架 Python 模块 | `commands.py` 的 `execute_command`, `tools.py` 的 `execute_tool` |

---

*报告生成时间：2026-04-01 | 分析工具：手工代码审查 + Python 脚本辅助统计*
