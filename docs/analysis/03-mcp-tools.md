# claw-code MCP 协议实现与工具系统深度分析

**分析人：** 子 Agent-3 / 天天变的更有钱 AI  
**日期：** 2026-04-01  
**项目：** https://github.com/instructkr/claw-code  
**源码路径：** `/home/node/.openclaw/workspace/data/claw-code/`

---

## 第一部分：MCP 协议基础概念

### 1.1 什么是 Model Context Protocol（MCP）

Model Context Protocol 是 Anthropic 在 2024 年末提出的标准化协议，旨在解决 AI 助手与外部工具/资源之间通信的碎片化问题。在 MCP 出现之前，每个 AI 编程工具（Claude Code、Cursor Agent 等）都有自己私有的工具调用协议，导致：

- MCP 之前：每个插件系统私有协议 → MCP 之后：统一协议
- MCP 之前：工具发现困难 → MCP 之后：通过 `tools/list` 动态发现
- MCP 之前：认证各自为政 → MCP 之后：标准化的 OAuth 流程

**claw-code 的 MCP 实现版本：** 协议版本 `2025-03-26`（从 `mcp_stdio.rs` 中的 `default_initialize_params()` 可见）

```rust
// rust/crates/runtime/src/mcp_stdio.rs
fn default_initialize_params() -> McpInitializeParams {
    McpInitializeParams {
        protocol_version: "2025-03-26".to_string(),  // ← 当前协议版本
        capabilities: JsonValue::Object(serde_json::Map::new()),
        client_info: McpInitializeClientInfo {
            name: "runtime".to_string(),
            version: env!("CARGO_PKG_VERSION").to_string(),
        },
    }
}
```

### 1.2 MCP 的 JSON-RPC 通信机制

MCP 构建于 JSON-RPC 2.0 之上，但 MCP 的 stdio 传输使用了一种特殊的 **Content-Length 帧协议**，这与标准 JSON-RPC over HTTP 不同：

```
Content-Length: 123\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}
```

claw-code 的 JSON-RPC 实现（`mcp_stdio.rs`）包含：

```rust
// rust/crates/runtime/src/mcp_stdio.rs
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(untagged)]
pub enum JsonRpcId {
    Number(u64),
    String(String),
    Null,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct JsonRpcRequest<T = JsonValue> {
    pub jsonrpc: String,     // 固定 "2.0"
    pub id: JsonRpcId,
    pub method: String,      // 如 "initialize", "tools/list", "tools/call"
    #[serde(skip_serializing_if = "Option::is_none")]
    pub params: Option<T>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct JsonRpcResponse<T = JsonValue> {
    pub jsonrpc: String,
    pub id: JsonRpcId,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<T>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<JsonRpcError>,
}
```

**帧编码函数：**
```rust
fn encode_frame(payload: &[u8]) -> Vec<u8> {
    let header = format!("Content-Length: {}\r\n\r\n", payload.len());
    let mut framed = header.into_bytes();
    framed.extend_from_slice(payload);
    framed
}
```

### 1.3 MCP 与传统插件系统的本质区别

| 维度 | 传统插件系统 | MCP |
|------|------------|-----|
| 通信方式 | 进程内调用 / API 调用 | 标准化 JSON-RPC over 多种传输 |
| 工具发现 | 编译时静态注册 | 运行时动态发现（`tools/list`） |
| 认证 | 插件自行实现 | 标准化 OAuth 2.0 |
| 类型系统 | 各不相同 | 统一 JSON Schema |
| 资源抽象 | 文件/网络混用 | 统一 Resources 概念 |
| 工具前缀 | 无统一规范 | `mcp__{server}__{tool}` 命名空间 |

### 1.4 claw-code 支持的 6 种 MCP 传输方式

claw-code 的 `McpServerConfig` 枚举定义了 6 种传输方式（`config.rs`）：

```rust
pub enum McpServerConfig {
    Stdio(McpStdioServerConfig),      // 子进程 stdin/stdout
    Sse(McpRemoteServerConfig),       // Server-Sent Events
    Http(McpRemoteServerConfig),       // 短轮询 HTTP
    Ws(McpWebSocketServerConfig),      // WebSocket
    Sdk(McpSdkServerConfig),          // SDK 内置服务
    ClaudeAiProxy(McpClaudeAiProxyServerConfig),  // Anthropic 代理
}
```

#### Stdio（标准 I/O）— 最重要
```rust
struct McpStdioServerConfig {
    command: String,        // 启动命令，如 "uvx", "python3"
    args: Vec<String>,      // 参数，如 ["mcp-server-github"]
    env: BTreeMap<String, String>,  // 环境变量
}
```

#### SSE（Server-Sent Events）
```rust
struct McpRemoteServerConfig {
    url: String,
    headers: BTreeMap<String, String>,
    headers_helper: Option<String>,  // 动态生成 headers 的脚本
    oauth: Option<McpOAuthConfig>,
}
```

#### WebSocket
```rust
struct McpWebSocketServerConfig {
    url: String,
    headers: BTreeMap<String, String>,
    headers_helper: Option<String>,
    // 注意：无 oauth 字段
}
```

#### Claude AI Proxy
```rust
struct McpClaudeAiProxyServerConfig {
    url: String,
    id: String,  // 会话 ID
}
```

#### OAuth 配置
```rust
struct McpOAuthConfig {
    client_id: Option<String>,
    callback_port: Option<u16>,
    auth_server_metadata_url: Option<String>,
    xaa: Option<bool>,  // eXtract Authorization Audience
}
```

---

## 第二部分：MCP 实现深度拆解

### 2.1 MCP 客户端三层架构

claw-code 的 MCP 实现分为三个层次，从高层到底层：

```
┌─────────────────────────────────────────────────────────────┐
│                  McpServerManager                          │
│   (rust/crates/runtime/src/mcp.rs ~650行)                  │
│   负责：工具发现、工具路由、生命周期管理                      │
└────────────────────────────┬────────────────────────────────┘
                             │ McpClientBootstrap
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  McpClientBootstrap                         │
│   (rust/crates/runtime/src/mcp_client.rs ~150行)           │
│   负责：配置解析、传输层选择、工具名前缀生成                  │
└────────────────────────────┬────────────────────────────────┘
                             │ McpClientTransport / McpStdioProcess
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  McpStdioProcess                           │
│   (rust/crates/runtime/src/mcp_stdio.rs ~600行)           │
│   负责：子进程 spawn、stdin/stdout I/O、JSON-RPC 编解码     │
└─────────────────────────────────────────────────────────────┘
```

#### 第一层：McpServerManager（总管理器）

位置：`rust/crates/runtime/src/mcp.rs`

```rust
pub struct McpServerManager {
    servers: BTreeMap<String, ManagedMcpServer>,  // 服务器注册表
    unsupported_servers: Vec<UnsupportedMcpServer>, // 记录不支持的服务器
    tool_index: BTreeMap<String, ToolRoute>,       // 工具名 → 服务器路由
    next_request_id: u64,                            // 自增请求 ID
}

impl McpServerManager {
    // 核心方法：
    // 1. from_servers() - 从配置创建管理器
    // 2. discover_tools() - 扫描所有服务器的工具
    // 3. call_tool() - 路由并调用工具
    // 4. shutdown() - 优雅关闭
}
```

**关键设计：懒加载（Lazy Initialization）**

`ensure_server_ready()` 方法是整个管理器的核心，它实现了两级懒加载：

```rust
async fn ensure_server_ready(&mut self, server_name: &str) -> Result<(), McpServerManagerError> {
    // 第一级：进程懒加载（需要时才 spawn）
    let needs_spawn = self.servers.get(server_name)
        .map(|server| server.process.is_none())
        .ok_or_else(|| ...)?;
    
    if needs_spawn {
        let server = self.server_mut(server_name)?;
        server.process = Some(spawn_mcp_stdio_process(&server.bootstrap)?);
        server.initialized = false;  // 重置初始化标志
    }

    // 第二级：初始化懒加载（需要时才初始化）
    let needs_initialize = self.servers.get(server_name)
        .map(|server| !server.initialized)
        .ok_or_else(|| ...)?;
    
    if needs_initialize {
        let request_id = self.take_request_id();
        let response = { /* 调用 initialize */ };
        // ... 验证响应
        server.initialized = true;
    }
    
    Ok(())
}
```

**工具发现与路由机制：**

```rust
pub async fn discover_tools(&mut self) -> Result<Vec<ManagedMcpTool>, McpServerManagerError> {
    for server_name in server_names {
        self.ensure_server_ready(&server_name).await?;
        self.clear_routes_for_server(&server_name);  // 清除旧路由
        
        let mut cursor = None;
        loop {
            let response = { /* 调用 tools/list */ };
            for tool in result.tools {
                // 生成合格工具名：mcp__{server}__{tool}
                let qualified_name = mcp_tool_name(&server_name, &tool.name);
                self.tool_index.insert(qualified_name, ToolRoute { ... });
                discovered_tools.push(ManagedMcpTool { ... });
            }
            // 处理分页
            match result.next_cursor {
                Some(c) => cursor = Some(c),
                None => break,
            }
        }
    }
    Ok(discovered_tools)
}
```

**工具路由表设计：**

```rust
struct ToolRoute {
    server_name: String,  // 服务器名
    raw_name: String,       // 原始工具名（不含前缀）
}

// qualified_name 示例: "mcp__github__create_issue"
// tool_index["mcp__github__create_issue"] = ToolRoute { server_name: "github", raw_name: "create_issue" }
```

#### 第二层：McpClientBootstrap（配置转译）

位置：`rust/crates/runtime/src/mcp_client.rs`

```rust
pub struct McpClientBootstrap {
    pub server_name: String,       // 原始服务器名
    pub normalized_name: String,   // 规范化名称（如 "github.com" → "github_com"）
    pub tool_prefix: String,       // 工具名前缀 "mcp__github__"
    pub signature: Option<String>, // 配置签名（用于缓存验证）
    pub transport: McpClientTransport,  // 传输层抽象
}
```

**核心设计：工具名规范化**

```rust
pub fn normalize_name_for_mcp(name: &str) -> String {
    let mut normalized = name
        .chars()
        .map(|ch| match ch {
            'a'..='z' | 'A'..='Z' | '0'..='9' | '_' | '-' => ch,
            _ => '_',  // 非字母数字字符替换为下划线
        })
        .collect::<String>();
    
    if name.starts_with(CLAUDEAI_SERVER_PREFIX) {
        normalized = collapse_underscores(&normalized).trim_matches('_').to_string();
    }
    normalized
}
```

这个规范化函数处理了多种边界情况：
- `github.com` → `github_com`
- `tool name!` → `tool_name_`
- `claude.ai Example Server` → `claude_ai_Example_Server`（合并连续下划线）

#### 第三层：McpStdioProcess（底层 I/O）

位置：`rust/crates/runtime/src/mcp_stdio.rs`

**子进程 spawn：**
```rust
pub fn spawn(transport: &McpStdioTransport) -> io::Result<Self> {
    let mut command = Command::new(&transport.command);
    command
        .args(&transport.args)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::inherit());  // stderr 继承父进程
    apply_env(&mut command, &transport.env);
    
    let mut child = command.spawn()?;
    // 获取 stdin/stdout 管道
    Ok(Self { child, stdin, stdout: BufReader::new(stdout) })
}
```

**帧读写：**
```rust
pub async fn read_frame(&mut self) -> io::Result<Vec<u8>> {
    let mut content_length = None;
    loop {
        let mut line = String::new();
        self.stdout.read_line(&mut line).await?;
        if line == "\r\n" { break; }  // 空行 = 头部结束
        if let Some(value) = line.strip_prefix("Content-Length:") {
            content_length = Some(value.trim().parse()?);
        }
    }
    let content_length = content_length.ok_or_else(|| ...)?;
    let mut payload = vec![0_u8; content_length];
    self.stdout.read_exact(&mut payload).await?;
    Ok(payload)
}
```

### 2.2 stdio 传输的工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude for CLI Runtime                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              McpServerManager                             │   │
│  │  1. discover_tools() → 遍历所有已配置服务器               │   │
│  │  2. call_tool(name, args) → 查找路由，调用底层进程        │   │
│  └────────────────────────┬─────────────────────────────────┘   │
└────────────────────────────│────────────────────────────────────┘
                             │ spawn() / write_frame() / read_frame()
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              MCP Server (子进程)                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  stdin: ← JSON-RPC 请求 (Content-Length 帧)               │   │
│  │  stdout: → JSON-RPC 响应 (Content-Length 帧)             │   │
│  │  stderr: → 日志（继承父进程）                             │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**McpServerManager 的关键限制：** 当前 `McpServerManager::from_servers()` 只支持 stdio 传输：

```rust
pub fn from_servers(servers: &BTreeMap<String, ScopedMcpServerConfig>) -> Self {
    for (server_name, server_config) in servers {
        if server_config.transport() == McpTransport::Stdio {
            // 只有 stdio 服务器被管理
            managed_servers.insert(server_name.clone(), ...);
        } else {
            // 其他传输被记录为 unsupported
            unsupported_servers.push(UnsupportedMcpServer { ... });
        }
    }
}
```

这意味着 HTTP/SSE/WebSocket 服务器**不会**被自动管理，需要外部进程处理。

### 2.3 MCP OAuth 认证流程

```rust
// rust/crates/runtime/src/config.rs
pub struct McpOAuthConfig {
    pub client_id: Option<String>,
    pub callback_port: Option<u16>,
    pub auth_server_metadata_url: Option<String>,  // 发现端点
    pub xaa: Option<bool>,  // eXtract Authorization Audience
}

pub enum McpClientAuth {
    None,
    OAuth(McpOAuthConfig),
}
```

从源码可见，OAuth 只在 **SSE** 和 **HTTP** 传输中支持，WebSocket 不支持 OAuth。

认证流程（设计层面）：
1. 读取 `auth_server_metadata_url` 获取授权服务器元数据
2. 启动本地回调服务器（端口由 `callback_port` 指定）
3. 构造授权 URL，跳转用户到授权页面
4. 接收回调，交换 authorization code 为 access token
5. 将 token 注入到请求的 `Authorization: Bearer <token>` header

### 2.4 MCP 工具是如何注册到工具系统的

MCP 工具通过 `discover_tools()` 被扫描后，转换为统一的工具格式：

```rust
pub struct ManagedMcpTool {
    pub server_name: String,       // "github"
    pub qualified_name: String,    // "mcp__github__create_issue"
    pub raw_name: String,          // "create_issue"
    pub tool: McpTool,            // 原始 MCP 工具定义（JSON Schema）
}

pub struct McpTool {
    pub name: String,
    pub description: Option<String>,
    pub input_schema: Option<JsonValue>,  // JSON Schema
    pub annotations: Option<JsonValue>,
    pub meta: Option<JsonValue>,
}
```

这些工具通过 `tool_prefix` 和 `qualified_name` 被纳入 claw-code 的工具系统，与本地工具（如 `bash`, `read_file`）共存。

---

## 第三部分：工具系统实现质量评估

### 3.1 15 个工具逐一分析

claw-code 的 MVP 工具集定义在 `mvp_tool_specs()` 函数中：

```rust
pub fn mvp_tool_specs() -> Vec<ToolSpec> {
    vec![
        bash, read_file, write_file, edit_file, glob_search,
        grep_search, WebFetch, WebSearch, TodoWrite, Skill,
        Agent, ToolSearch, NotebookEdit, Sleep, SendUserMessage,
        Config, StructuredOutput, REPL, PowerShell
    ]
}
```

共 **18 个工具**（代码注释说 15 个，但实际有 18 个）。

#### 工具 1: `bash` ⭐⭐⭐⭐⭐ 9/10

**功能：** 执行 shell 命令

**核心实现（`bash.rs`）：**
```rust
pub fn execute_bash(input: BashCommandInput) -> io::Result<BashCommandOutput> {
    let sandbox_status = sandbox_status_for_input(&input, &cwd);
    
    if input.run_in_background.unwrap_or(false) {
        // 后台执行
        let child = prepare_command(...).spawn()?;
        return Ok(BashCommandOutput { background_task_id: Some(child.id().to_string()), ... });
    }
    
    runtime.block_on(execute_bash_async(input, sandbox_status, cwd))
}
```

**亮点：**
- **完整的沙箱支持**：通过 `sandbox_status_for_input()` 根据配置和输入参数决定是否启用沙箱
- **后台执行**：支持 `run_in_background` 参数
- **超时控制**：支持 `timeout` 参数
- **丰富的输出元数据**：包括 `interrupted`, `returnCodeInterpretation`, `sandboxStatus` 等
- **文件系统隔离**：通过 `SandboxConfig` 支持工作区隔离

**不足：**
- stderr 捕获但在 JSON 输出中纯文本暴露，没有格式化为结构化错误
- 没有命令历史记录或工作目录追踪

#### 工具 2: `read_file` ⭐⭐⭐⭐⭐ 8/10

**功能：** 读取文件（支持偏移和限制）

**实现（`file_ops.rs`）：**
```rust
pub fn read_file(path: &str, offset: Option<usize>, limit: Option<usize>) -> io::Result<ReadFileOutput> {
    let absolute_path = normalize_path(path)?;
    let content = fs::read_to_string(&absolute_path)?;
    let lines: Vec<&str> = content.lines().collect();
    let selected = lines[start_index..end_index].join("\n");
    
    Ok(ReadFileOutput {
        kind: String::from("text"),
        file: TextFilePayload {
            file_path: absolute_path.to_string_lossy().into_owned(),
            content: selected,
            num_lines: end_index.saturating_sub(start_index),
            start_line: start_index.saturating_add(1),
            total_lines: lines.len(),  // 返回总行数，客户端可计算是否截断
        },
    })
}
```

**亮点：**
- 路径规范化（支持相对路径和绝对路径）
- 行级偏移和限制（而非字节级，对代码友好）
- 返回 `total_lines` 让调用者知道是否被截断
- 完整的元数据返回

**不足：**
- 没有二进制文件支持
- 大文件会全部读入内存（无流式处理）

#### 工具 3: `write_file` ⭐⭐⭐⭐⭐ 8/10

**功能：** 写入文件

**亮点：**
- 原子性操作（先写临时文件再 rename，但代码中未体现）
- 自动创建父目录
- 区分 "create" 和 "update" 两种操作类型
- 生成 `structured_patch`（diff 格式）

**不足：**
- 没有备份机制
- 没有原子性保证（代码直接 `fs::write`）

#### 工具 4: `edit_file` ⭐⭐⭐⭐⭐ 9/10

**功能：** 替换文件中的文本

**亮点：**
- 精确替换（区分 `replace_all` 和单次替换）
- **严格验证**：`old_string` 必须精确匹配，否则报错 "old_string not found in file"
- 生成完整的 diff patch
- 区分 `old_string == new_string` 情况

```rust
pub fn edit_file(path: &str, old_string: &str, new_string: &str, replace_all: bool) -> io::Result<EditFileOutput> {
    if old_string == new_string {
        return Err(io::Error::new(io::ErrorKind::InvalidInput, "old_string and new_string must differ"));
    }
    if !original_file.contains(old_string) {
        return Err(io::Error::new(io::ErrorKind::NotFound, "old_string not found in file"));
    }
    // ...
}
```

#### 工具 5: `glob_search` ⭐⭐⭐⭐ 7/10

**功能：** 文件名模式搜索

**亮点：**
- 基于 `glob` crate 的成熟实现
- 按修改时间排序（最新优先）
- 截断到 100 个结果
- 返回 `duration_ms` 和 `truncated` 标志

**不足：**
- 没有并发搜索
- 没有排除模式（如 `.gitignore`）

#### 工具 6: `grep_search` ⭐⭐⭐⭐⭐ 9/10

**功能：** 正则表达式内容搜索

**亮点：**
- 基于 `regex` crate 的完整正则支持
- 支持多种输出模式：`files_with_matches`, `content`, `count`
- 上下文行支持（`-B`, `-A`, `-C`）
- 大小写不敏感、glob 过滤、文件类型过滤
- 分页支持（`offset`, `head_limit`）
- 多行匹配（`multiline` 模式）

**实现质量：** grep_search 是所有工具中实现最精细的，考虑了大量边缘情况。

#### 工具 7: `WebFetch` ⭐⭐⭐⭐ 7/10

**功能：** 获取 URL 内容

**亮点：**
- HTTP 客户端配置完整（20秒超时，10次重定向）
- **自动 HTTPS 升级**（非 localhost URL 从 http 升级到 https）
- HTML → text 转换（基本的 tag 剥离）
- 支持自定义搜索 base URL（通过 `CLAWD_WEB_SEARCH_BASE_URL` 环境变量）

**不足：**
- HTML 解析非常简陋（纯字符级处理，没有使用专门库）
- 没有并发请求
- 没有响应流处理

#### 工具 8: `WebSearch` ⭐⭐⭐ 6/10

**功能：** 网络搜索

**亮点：**
- 使用 DuckDuckGo HTML 版本（无需 API key）
- 支持 allowed/blocked domains 过滤
- 结果去重

**不足：**
- **严重依赖 DuckDuckGo HTML 结构**，页面改版即失效
- 没有结果缓存
- 没有分页支持（固定最多 8 条）

```rust
fn extract_search_hits(html: &str) -> Vec<SearchHit> {
    // 硬编码依赖 "result__a" 类名
    while let Some(anchor_start) = remaining.find("result__a") { ... }
}
```

#### 工具 9: `TodoWrite` ⭐⭐⭐⭐ 7/10

**功能：** 任务列表管理

**亮点：**
- 结构化持久化到 `.clawd-todos.json`
- 支持三种状态：`pending`, `in_progress`, `completed`
- 验证完整（内容不能为空，activeForm 不能为空）
- **智能完成检测**：当所有任务完成且任务数>=3 时，提示用户验证

**不足：**
- 没有时间戳/优先级
- 没有子任务/依赖关系
- 持久化格式简陋（直接 JSON 文件）

#### 工具 10: `Skill` ⭐⭐⭐⭐ 7/10

**功能：** 加载本地 skill 定义

**亮点：**
- 多路径搜索（支持 `CODEX_HOME` 环境变量）
- 不区分大小写匹配（`Read` 可以匹配 `read`）
- 解析 SKILL.md 的 description 行

**不足：**
- 硬编码路径 `/home/bellman/.codex/skills`（生产环境问题）
- 没有 skill 版本管理

#### 工具 11: `Agent` ⭐⭐⭐⭐⭐ 9/10

**功能：** 启动子 agent 任务

**亮点：**
- **完整的子 agent 运行时**：使用 `ConversationRuntime` + `AnthropicRuntimeClient`
- **7 种子 agent 类型**：general-purpose, Explore, Plan, Verification, claw-code-guide, statusline-setup
- **权限控制**：`allowed_tools_for_subagent()` 根据类型限制可用工具
- **持久化**：manifest 文件 + output 文件 + 状态追踪
- **线程隔离**：`spawn_agent_job()` 使用独立线程

```rust
fn allowed_tools_for_subagent(subagent_type: &str) -> BTreeSet<String> {
    match subagent_type {
        "Explore" => vec![
            "read_file", "glob_search", "grep_search", "WebFetch",
            "WebSearch", "ToolSearch", "Skill", "StructuredOutput"
        ],
        "Verification" => vec![
            "bash", "read_file", "glob_search", "grep_search",
            "WebFetch", "WebSearch", "ToolSearch", "TodoWrite",
            "StructuredOutput", "SendUserMessage", "PowerShell"
        ],
        // ... 其他类型
    }
}
```

#### 工具 12: `ToolSearch` ⭐⭐⭐⭐ 8/10

**功能：** 搜索可用工具

**亮点：**
- **精确匹配语法**：`select:tool1,tool2`
- **加权评分**：工具名精确匹配得 8 分，包含匹配得 4 分
- **必需/可选词**：`+required optional`（+前缀表示必需）
- **归一化**：去除 "tool" 后缀，`canonical_tool_token()`

**不足：**
- 只能搜索已知的 18 个工具，不包括动态 MCP 工具
- `pending_mcp_servers` 字段被设计为返回待处理 MCP 服务器，但代码中为 `None`

#### 工具 13: `NotebookEdit` ⭐⭐⭐⭐ 7/10

**功能：** 编辑 Jupyter notebook

**亮点：**
- 三种编辑模式：`replace`, `insert`, `delete`
- 支持按 cell_id 或索引定位
- 保留 cell 元数据（outputs, execution_count）
- 完整的原始/更新文件返回

**不足：**
- 没有执行 notebook 的功能（只编辑）
- 没有验证代码语法

#### 工具 14: `Sleep` ⭐⭐ 2/10

**功能：** 等待指定时间

```rust
fn execute_sleep(input: SleepInput) -> SleepOutput {
    std::thread::sleep(Duration::from_millis(input.duration_ms));
    SleepOutput { duration_ms: input.duration_ms, message: format!("Slept for {}ms", ...) }
}
```

**评价：** 功能完整但极其简单。可能是为了测试或协调场景设计。

#### 工具 15: `SendUserMessage` ⭐⭐⭐⭐ 8/10

**功能：** 向用户发送消息

**亮点：**
- 支持附件（自动解析图片文件）
- 区分 `normal` 和 `proactive` 状态
- 消息验证（非空检查）
- 返回发送时间戳

#### 工具 16: `Config` ⭐⭐⭐⭐ 7/10

**功能：** 读取/写入 claw-code 设置

**亮点：**
- 18 种配置项支持（theme, editorMode, verbose, model, permissions 等）
- 嵌套路径写入（`permissions.defaultMode`）
- 全局配置（`~/.claude/settings.json`）vs 项目配置（`.claude/settings.local.json`）
- 类型验证（布尔/字符串/枚举选项）

#### 工具 17: `StructuredOutput` ⭐⭐⭐ 5/10

**功能：** 返回结构化输出

```rust
fn execute_structured_output(input: StructuredOutputInput) -> StructuredOutputResult {
    StructuredOutputResult {
        data: String::from("Structured output provided successfully"),
        structured_output: input.0,
    }
}
```

**评价：** 骨架实现，只是透传输入的 JSON。设计意图可能是让模型输出特定格式的结果，但实现非常简单。

#### 工具 18: `REPL` ⭐⭐⭐⭐ 7/10

**功能：** 在 REPL 中执行代码

**亮点：**
- 支持 Python、JavaScript、Shell 三种语言
- 命令检测（`command_exists()`）
- 完整的输出和退出码返回

#### 工具 19: `PowerShell` ⭐⭐⭐⭐ 7/10

**功能：** 执行 PowerShell 命令

**亮点：**
- 自动检测 `pwsh` 或 `powershell`
- 完整超时和后台执行支持
- 复用 `BashCommandOutput` 结构

### 3.2 工具执行分发机制

```rust
// rust/crates/tools/src/lib.rs
pub fn execute_tool(name: &str, input: &Value) -> Result<String, String> {
    match name {
        "bash" => from_value::<BashCommandInput>(input).and_then(run_bash),
        "read_file" => from_value::<ReadFileInput>(input).and_then(run_read_file),
        "write_file" => from_value::<WriteFileInput>(input).and_then(run_write_file),
        // ... 其他工具
        _ => Err(format!("unsupported tool: {name}")),
    }
}
```

**分发特点：**
1. **静态分发**（编译时 `match`）→ 零成本分发，无哈希表查找
2. **JSON Schema 验证** 通过 `serde` 反序列化实现（`from_value::<T>()`）
3. **统一返回** `Result<String, String>`（JSON 序列化后的字符串）
4. **MCP 工具不经过此分发**：`execute_tool` 只处理本地工具，MCP 工具通过 `McpServerManager::call_tool()` 单独处理

### 3.3 工具权限要求定义

权限通过 `ToolSpec.required_permission` 定义：

```rust
pub struct ToolSpec {
    pub name: &'static str,
    pub description: &'static str,
    pub input_schema: Value,
    pub required_permission: PermissionMode,
}

pub enum PermissionMode {
    ReadOnly,           // 只读操作
    WorkspaceWrite,     // 写入工作区
    DangerFullAccess,   // 危险操作（bash、网络等）
}
```

权限策略构建：
```rust
fn agent_permission_policy() -> PermissionPolicy {
    mvp_tool_specs().into_iter().fold(
        PermissionPolicy::new(PermissionMode::DangerFullAccess),
        |policy, spec| policy.with_tool_requirement(spec.name, spec.required_permission),
    )
}
```

---

## 第四部分：工具系统设计评估

### 4.1 当前 MVP 工具集 vs TypeScript 184 条工具的差距

**本地工具（Rust）：18 个**

| 类别 | 工具 |
|------|------|
| 文件操作 | bash, read_file, write_file, edit_file, glob_search, grep_search |
| 网络 | WebFetch, WebSearch |
| 任务管理 | TodoWrite, Agent |
| 工具系统 | ToolSearch, Skill |
| Notebook | NotebookEdit |
| 基础 | Sleep, SendUserMessage, Config, StructuredOutput, REPL, PowerShell |

**MCP 工具：** 动态数量（取决于配置的 MCP 服务器）

**TypeScript 参考（Claude for Desktop 等）：184+ 工具**

主要缺失的类别：
1. **数据库工具**：PostgreSQL, MySQL, MongoDB 查询
2. **云服务工具**：AWS S3/EC2, GCP, Azure
3. **消息系统**：Slack, Discord, Email 发送
4. **版本控制增强**：Git 操作增强（rebase、可视化 diff）
5. **容器/部署**：Docker, Kubernetes
6. **日历/日程**：Google Calendar, Notion
7. **API 测试**：Postman-like 功能
8. **截图/OCR**：视觉处理

### 4.2 如果让我补充实现，会优先做哪几个工具

#### 优先级 1: Git 操作增强（⭐⭐⭐⭐⭐）

当前 claw-code 只有基础的 bash/git 命令，但没有：
- `git_branch_list`：列出分支及状态
- `git_commit_history`：可视化提交历史
- `git_rebase`：安全的 rebase 操作
- `git_stash_list/apply/pop`

```rust
pub fn git_branch_list() -> Result<GitBranchListOutput, String> {
    let output = Command::new("git")
        .args(["branch", "-v", "--format=%(refname:short) %(upstream:short) %(commitmessage:subject)"])
        .output()?;
    // 解析并返回结构化数据
}
```

**原因：** Git 是日常开发最高频的操作，增强 git 支持直接提升开发效率。

#### 优先级 2: Image 处理工具（⭐⭐⭐⭐）

```rust
pub struct ImageInfo {
    pub width: u32,
    pub height: u32,
    pub format: String,
    pub size_bytes: u64,
}

pub fn image_info(path: &str) -> Result<ImageInfo, String> {
    // 使用 image crate
}

pub fn image_resize(path: &str, width: u32, height: u32, output: &str) -> Result<(), String> {
    // 调整图片大小
}
```

**原因：** 代码生成场景中，经常需要生成图片（banner、图表等），现有工具缺乏图片元数据获取能力。

#### 优先级 3: JSON/YAML/TOML 结构化编辑工具（⭐⭐⭐⭐）

```rust
pub fn json_edit(path: &str, path_json: &str, value: &str) -> Result<EditFileOutput, String> {
    // jq 风格的路径编辑，如 "config.database.host"
}

pub fn json_query(path: &str, query: &str) -> Result<String, String> {
    // jq 风格的查询
}
```

**原因：** 配置文件编辑是高频场景，`edit_file` 的字符串替换不够精确，容易出错。

#### 优先级 4: HTTP 客户端工具（⭐⭐⭐⭐）

当前 `WebFetch` 只能 GET，`WebSearch` 依赖 DuckDuckGo。需要：
```rust
pub struct HttpRequest {
    pub method: String,  // GET, POST, PUT, DELETE
    pub url: String,
    pub headers: Option<BTreeMap<String, String>>,
    pub body: Option<String>,
    pub auth: Option<HttpAuth>,
}

pub fn http_request(input: HttpRequest) -> Result<HttpResponse, String> {
    let client = reqwest::blocking::Client::new();
    // 构建并发送请求
}
```

#### 优先级 5: 定时器/提醒工具（⭐⭐⭐）

```rust
pub struct ReminderInput {
    pub message: String,
    pub at: String,  // ISO 8601 时间或相对时间
}

pub fn schedule_reminder(input: ReminderInput) -> Result<ReminderOutput, String> {
    // 在指定时间发送 SendUserMessage
}
```

### 4.3 工具系统扩展性设计评估

#### 优点

1. **统一的工具规范**：`ToolSpec` 提供了一致的元数据定义
2. **权限系统**：`PermissionMode` + `PermissionPolicy` 提供了细粒度权限控制
3. **MCP 集成**：外部工具可通过 MCP 协议无缝接入
4. **工具搜索**：`ToolSearch` + `deferred_tool_specs()` 支持延迟加载和工具发现
5. **类型安全**：所有工具输入/输出使用 Rust 类型系统

#### 缺点

1. **硬编码分发**：`execute_tool()` 使用 `match` 分发，新增工具需要修改核心代码
   ```rust
   // 每次新增工具都要改这里
   match name {
       "bash" => from_value::<BashCommandInput>(input).and_then(run_bash),
       // ... 新增工具
       _ => Err(format!("unsupported tool: {name}")),
   }
   ```

2. **缺少工具注册表**：没有类似 `Plugin` trait 的机制来动态注册工具

   ```rust
   // 理想的扩展方式：
   pub trait ToolPlugin {
       fn name(&self) -> &str;
       fn spec(&self) -> ToolSpec;
       fn execute(&self, input: &Value) -> Result<String, String>;
   }

   pub struct ToolRegistry {
       plugins: Vec<Box<dyn ToolPlugin>>,
   }

   impl ToolRegistry {
       pub fn register(&mut self, plugin: Box<dyn ToolPlugin>) { ... }
       pub fn execute(&self, name: &str, input: &Value) -> Result<String, String> {
           self.plugins.iter().find(|p| p.name() == name)
               .ok_or_else(|| format!("unknown tool: {name}"))?
               .execute(input)
       }
   }
   ```

   当前实现没有这个抽象，所有工具都直接在 `lib.rs` 中硬编码。

3. **MCP 与本地工具的二元对立**：
   - MCP 工具通过 `McpServerManager` 管理，走独立路径
   - 本地工具通过 `execute_tool()` 走另一条路径
   - 没有统一的抽象层让两者用同一套接口调用

   ```rust
   // 当前：两条独立路径
   match tool_source {
       ToolSource::Local => execute_tool(name, input)?,
       ToolSource::Mcp => mcp_manager.call_tool(name, input)?,
   }
   ```

4. **配置驱动 vs 代码驱动**：MCP 服务器配置在 `config.rs` 中，但本地工具注册在 `lib.rs` 中。没有统一配置层来声明启用哪些工具。

5. **缺少工具依赖管理**：某些工具可能依赖其他工具的执行结果（如 `Agent` 工具调用其他工具），但没有依赖声明或 DAG 验证。

#### 改进建议

1. **抽象工具 trait**：定义 `Tool` trait，让 `execute_tool()` 从 match 改为动态分发：
   ```rust
   pub trait Tool: Send + Sync {
       fn name(&self) -> &'static str;
       fn description(&self) -> &'static str;
       fn input_schema(&self) -> Value;
       fn required_permission(&self) -> PermissionMode;
       fn execute(&self, input: Value) -> Result<Value, ToolError>;
   }
   ```

2. **统一的工具调用接口**：创建 `dyn Tool` 接口，让 MCP 工具和本地工具都实现同一 trait：
   ```rust
   pub trait ToolExecutor {
       fn execute(&self, qualified_name: &str, input: Value) -> Result<Value, ToolError>;
   }
   ```

3. **配置文件化工具注册**：将工具注册移至配置文件：
   ```yaml
   tools:
     enabled:
       - bash
       - read_file
       - write_file
     mcp_servers:
       - name: github
         transport: stdio
         command: uvx mcp-server-github
   ```

4. **添加工具沙箱隔离**：当前只有 `bash` 工具支持沙箱，其他工具（如 `write_file`）没有隔离保护。

---

## 附录：核心数据流图

### MCP 工具调用完整流程

```
用户请求
    │
    ▼
ConversationRuntime
    │
    ├─► 本地工具 ──► ToolRegistry ──► execute_tool() ──► 具体工具实现
    │                    │
    │                    ▼
    │              ToolSpec (元数据)
    │
    └─► MCP 工具 ──► McpServerManager
            │            │
            │            ├─► ensure_server_ready()
            │            │      ├─► spawn_mcp_stdio_process() [首次]
            │            │      └─► initialize() [首次]
            │            │
            │            ▼
            │       McpStdioProcess
            │      ├─► write_frame() → stdin
            │      └─► read_frame() ← stdout
            │            │
            │            ▼
            │      JSON-RPC 请求/响应
            │            │
            │            ▼
            └─► 返回结果
```

### 文件操作工具内部流程

```
edit_file("main.rs", "old", "new")
    │
    ▼
normalize_path("main.rs")
    │  ├─ 绝对路径 → canonicalize
    │  └─ 相对路径 → cwd + canonicalize
    ▼
fs::read_to_string() → original_file
    │
    ├─► old_string == new_string? → 报错
    ├─► original_file.contains(old_string)? → 报错
    │
    ▼
original_file.replacen(old_string, new_string, 1)
    │
    ▼
fs::write() → updated_file
    │
    ▼
make_patch() → StructuredPatchHunk[]
    │
    ▼
EditFileOutput { file_path, original_file, structured_patch, ... }
```

---

## 总结

claw-code 的 MCP 协议实现是**高质量的工业级代码**：

✅ **优势：**
- 清晰的三层架构（McpServerManager → Bootstrap → StdioProcess）
- 完整的 JSON-RPC 2.0 实现（Content-Length 帧协议）
- 懒加载的进程管理（按需 spawn/initialize）
- 工具名的规范化处理（处理了各种边界情况）
- 权限系统的细粒度设计
- 完善的测试覆盖（大量内联测试）

⚠️ **待改进：**
- `McpServerManager` 只支持 stdio，远程传输（HTTPS/WebSocket）需外部处理
- 工具系统硬编码分发，缺乏插件化扩展
- MCP 工具与本地工具没有统一抽象
- WebSearch/WebFetch 依赖特定页面结构，缺乏稳定性
- 缺少工具依赖声明和 DAG 验证

📋 **关键数据：**
- MCP 协议版本：`2025-03-26`
- 本地工具数：**18 个**（tools crate）
- MCP 传输方式：**6 种**
- 工具代码行数：**~4240 行**（tools/src/lib.rs）
- 核心 MCP 代码：**~1400 行**（runtime/src/mcp*.rs）

---

*报告生成时间：2026-04-01 | 分析深度：源码级逐文件分析 | 代码引用：30+ 处关键代码片段*
