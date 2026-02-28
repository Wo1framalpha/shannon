# Shannon Architecture Deep Dive
# Shannon 架构深度剖析

## System Overview
## 系统概述

Shannon is a production-grade, AI-powered penetration testing framework that combines white-box source code analysis with black-box dynamic exploitation to discover and validate security vulnerabilities in web applications.

<!-- 中文说明：Shannon 是一个生产级的、由 AI 驱动的渗透测试框架，它将白盒源代码分析与黑盒动态漏洞利用相结合，用于发现和验证 Web 应用程序中的安全漏洞。-->

## High-Level Architecture
## 高层架构

<!--
中文说明：Shannon 采用四层架构设计：
1. 用户界面层 - CLI 命令行工具、Temporal Web UI 和审计日志
2. 编排层 - 使用 Temporal 进行工作流定义、活动执行和工作池管理
3. 执行层 - Claude Agent SDK（AI 引擎）、Playwright MCP（浏览器自动化）和 Git 检查点管理器
4. 工具层 - 外部安全工具（nmap、subfinder、whatweb）和源代码分析
-->

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                           │
│                         用户界面层                                │
│  CLI (shannon script) │ Temporal Web UI │ Audit Logs            │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                    Orchestration Layer                           │
│                    编排层                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐         │
│  │  Workflow   │  │  Activities  │  │  Worker Pool   │         │
│  │  Definition │──│  Execution   │──│  (Temporal)    │         │
│  │  工作流定义  │  │  活动执行     │  │  工作池        │         │
│  └─────────────┘  └──────────────┘  └────────────────┘         │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                      Execution Layer                             │
│                      执行层                                       │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐        │
│  │ Claude Agent │  │ Playwright │  │  Git Checkpoint  │        │
│  │     SDK      │  │    MCP     │  │     Manager      │        │
│  │  AI 引擎     │  │  浏览器自动化│  │  Git 检查点管理器 │        │
│  └──────────────┘  └────────────┘  └──────────────────┘        │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                       Tool Layer                                 │
│                       工具层                                      │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌──────────────┐      │
│  │  nmap   │  │subfinder │  │whatweb │  │ Source Code  │      │
│  │ 端口扫描 │  │ 子域名发现│  │ Web指纹 │  │  源代码分析   │      │
│  └─────────┘  └──────────┘  └────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

## Execution Flow
## 执行流程

### Complete Pipeline
### 完整流水线

<!--
中文说明：Shannon 的执行流程分为 5 个阶段：
1. 预侦察（Pre-Reconnaissance）- 顺序执行，运行外部工具扫描
2. 侦察（Reconnaissance）- 顺序执行，分析攻击面
3. 漏洞分析（Vulnerability Analysis）- 5 个 agent 并行执行
4. 漏洞利用（Exploitation）- 5 个 agent 并行执行（条件性）
5. 报告生成（Reporting）- 顺序执行，汇总所有发现
-->

```
┌──────────────────────────────────────────────────────────────────┐
│                         SHANNON PIPELINE                          │
│                         Shannon 流水线                             │
└──────────────────────────────────────────────────────────────────┘

Phase 1: PRE-RECONNAISSANCE (Sequential)
阶段 1: 预侦察（顺序执行）
┌────────────────────────────────────────────┐
│  pre-recon agent                           │
│  - Run external tools (nmap, subfinder)    │
│  - Analyze source code structure           │
│  - Identify tech stack                     │
│  - Generate attack surface map             │
└────────────────┬───────────────────────────┘
                 │
Phase 2: RECONNAISSANCE (Sequential)
┌────────────────▼───────────────────────────┐
│  recon agent                               │
│  - Analyze pre-recon findings              │
│  - Browser-based exploration               │
│  - Map all entry points                    │
│  - Identify auth mechanisms                │
└────────────────┬───────────────────────────┘
                 │
Phase 3-4: VULNERABILITY + EXPLOITATION (Pipelined Parallel)
                 │
    ┌────────────┴────────────┐
    │    Parallel Pipelines    │
    └────────────┬────────────┘
                 │
    ┌────────────┴────────────┬──────────────┬──────────────┬──────────────┐
    │                         │              │              │              │
┌───▼────┐                ┌───▼────┐     ┌───▼────┐    ┌───▼────┐    ┌───▼────┐
│injection│               │  xss   │     │  auth  │    │ authz  │    │  ssrf  │
│  vuln  │               │  vuln  │     │  vuln  │    │  vuln  │    │  vuln  │
└───┬────┘               └───┬────┘     └───┬────┘    └───┬────┘    └───┬────┘
    │                        │              │              │              │
┌───▼─────────┐         ┌───▼──────────┐┌──▼──────────┐┌──▼──────────┐┌──▼──────────┐
│Check Queue  │         │Check Queue   ││Check Queue  ││Check Queue  ││Check Queue  │
│>0 vulns?    │         │>0 vulns?     ││>0 vulns?    ││>0 vulns?    ││>0 vulns?    │
└───┬─────────┘         └───┬──────────┘└──┬──────────┘└──┬──────────┘└──┬──────────┘
    │Yes                    │Yes            │Yes            │Yes            │Yes
┌───▼────┐               ┌──▼────┐      ┌──▼────┐      ┌──▼────┐      ┌──▼────┐
│injection│              │  xss   │     │  auth  │     │ authz  │     │  ssrf  │
│ exploit │              │ exploit│     │ exploit│     │ exploit│     │ exploit│
└───┬────┘              └───┬────┘     └───┬────┘     └───┬────┘     └───┬────┘
    │                        │              │              │              │
    └────────────┬───────────┴──────────────┴──────────────┴──────────────┘
                 │
Phase 5: REPORTING (Sequential)
┌────────────────▼───────────────────────────┐
│  Assemble exploitation evidence            │
└────────────────┬───────────────────────────┘
                 │
┌────────────────▼───────────────────────────┐
│  report agent                              │
│  - Add executive summary                   │
│  - Clean up artifacts                      │
│  - Provide reproducible PoCs               │
└────────────────┬───────────────────────────┘
                 │
┌────────────────▼───────────────────────────┐
│  Inject model metadata                     │
└────────────────────────────────────────────┘
```

## Temporal Workflow Architecture
## Temporal 工作流架构

<!--
中文说明：Temporal 是 Shannon 的核心编排引擎，提供以下关键特性：
1. 状态机管理 - 跟踪工作流从初始化到完成的整个生命周期
2. 自动重试机制 - 区分可重试错误（账单、限流）和不可重试错误（认证、配置）
3. 崩溃恢复 - Worker 进程崩溃后自动恢复工作流状态
4. 可查询进度 - 实时查询当前执行阶段和完成的 agent
-->

### State Machine
### 状态机

<!--
中文说明：工作流状态转换：
- INITIAL（初始）→ RUNNING（运行中）→ COMPLETED（已完成）
- RUNNING 状态下如果遇到可重试错误，会在 5-30 分钟后自动重试
- 如果遇到不可恢复的错误，直接转到 FAILED（失败）状态
-->

```
┌──────────────┐
│   INITIAL    │
└──────┬───────┘
       │
       │ Start workflow
       ▼
┌──────────────┐
│   RUNNING    │◄─────┐
└──────┬───────┘      │
       │              │
       │ Execute      │ Retry
       │ phase        │ (5-30 min)
       │              │
       ├──────────────┘
       │
       │ All phases complete
       ▼
┌──────────────┐
│  COMPLETED   │
└──────────────┘

       │ Unrecoverable error
       ▼
┌──────────────┐
│   FAILED     │
└──────────────┘
```

### Retry Strategy

```
Error Classification Tree
├─ Retryable Errors
│  ├─ BillingError (spending cap)
│  │  └─ Retry: 5min → 10min → 20min → 30min (max)
│  ├─ RateLimitError (429)
│  │  └─ Retry: 5min → 10min → 20min → 30min (max)
│  └─ TransientError (5xx, network)
│     └─ Retry: 5min → 10min → 20min → 30min (max)
│
└─ Non-Retryable Errors
   ├─ AuthenticationError
   ├─ PermissionError
   ├─ ConfigurationError
   ├─ InvalidRequestError
   └─ ExecutionLimitError
```

## Agent Activity Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│              Agent Activity Execution Flow                   │
└─────────────────────────────────────────────────────────────┘

1. INITIALIZATION
   ┌──────────────────────────────────────┐
   │ Load config (if provided)            │
   │ Initialize audit session             │
   │ Load prompt template                 │
   └──────────────┬───────────────────────┘
                  │
2. CHECKPOINT
   ┌──────────────▼───────────────────────┐
   │ git stash push -m "checkpoint:..."   │
   └──────────────┬───────────────────────┘
                  │
3. EXECUTION
   ┌──────────────▼───────────────────────┐
   │ runClaudePrompt()                    │
   │ - Start heartbeat loop (2s)          │
   │ - Execute AI agent                   │
   │ - Stream messages                    │
   │ - Log tools and responses            │
   └──────────────┬───────────────────────┘
                  │
                  ├─────── Success ──────┐
                  │                      │
4a. VALIDATION                           │
   ┌──────────────▼───────────────────┐  │
   │ validateAgentOutput()            │  │
   │ - Check deliverable files exist  │  │
   │ - Verify structure/content       │  │
   └──────────────┬───────────────────┘  │
                  │                      │
                  ├── Pass ────┐         │
                  │            │         │
5a. COMMIT                     │         │
   ┌──────────────▼───────┐   │         │
   │ git stash drop       │   │         │
   │ git add . && commit  │   │         │
   └──────────────────────┘   │         │
                               │         │
                  ├── Fail ────┤         │
                  │            │         │
4b. ERROR HANDLING              │  ◄─────┘
   ┌──────────────▼───────────────────┐
   │ Classify error for Temporal      │
   │ - Retryable? → throw for retry   │
   │ - Non-retryable? → fail workflow │
   └──────────────┬───────────────────┘
                  │
5b. ROLLBACK
   ┌──────────────▼───────────────────┐
   │ git stash pop                    │
   │ Clean workspace                  │
   └──────────────────────────────────┘
```

## Claude Agent SDK Integration

### Message Stream Processing

```
┌────────────────────────────────────────────────────────────┐
│              Claude Agent SDK Message Loop                  │
└────────────────────────────────────────────────────────────┘

query({ prompt, options })
    │
    │ Stream messages
    ▼
┌────────────────────────┐
│ for await (message)    │
└───────┬────────────────┘
        │
        ├─ type: 'system_init'
        │  └─ Capture model name
        │
        ├─ type: 'assistant'
        │  ├─ Increment turn count
        │  └─ Log to audit
        │
        ├─ type: 'tool_use'
        │  ├─ Log tool name/params
        │  └─ Update progress indicator
        │
        ├─ type: 'tool_result'
        │  └─ Log result/duration
        │
        ├─ type: 'usage'
        │  └─ Accumulate cost
        │
        ├─ type: 'error'
        │  ├─ Detect API errors
        │  └─ Throw for retry
        │
        └─ type: 'complete'
           └─ Return result
```

### MCP Server Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Server Setup                          │
└─────────────────────────────────────────────────────────────┘

Agent requires browser automation?
    │
    ├─ Yes
    │  └─ Assign Playwright MCP instance
    │     ├─ Unique user data dir: /tmp/{agent-name}
    │     ├─ Isolated session (--isolated flag)
    │     └─ Docker: use system Chromium
    │         Local: use Playwright bundled browsers
    │
    └─ No
       └─ Shannon Helper MCP only

┌────────────────────────────────────────────────────────────┐
│ Shannon Helper MCP                                          │
│ - generate_totp(secret) → TOTP code                        │
│ - get_config() → Configuration context                     │
│ - Custom utility functions                                 │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ Playwright MCP (per-agent instance)                        │
│ - browser_navigate(url)                                    │
│ - browser_click(element)                                   │
│ - browser_type(element, text)                              │
│ - browser_snapshot() → Accessibility tree                  │
│ - browser_evaluate(js) → Execute JavaScript                │
└────────────────────────────────────────────────────────────┘
```

## Git Checkpoint System

```
┌────────────────────────────────────────────────────────────┐
│                   Git Checkpoint Flow                       │
└────────────────────────────────────────────────────────────┘

Agent attempt starts
    │
    ▼
┌──────────────────────────────────────┐
│ git status                           │
│ Check for uncommitted changes        │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ git stash push                       │
│   -m "checkpoint:agent:attempt-N"    │
│   --include-untracked                │
└──────────────┬───────────────────────┘
               │
               │ Execute agent
               │
               ▼
        ┌──────┴───────┐
        │              │
    Success         Failure
        │              │
        ▼              ▼
┌──────────────┐  ┌─────────────┐
│git stash drop│  │git stash pop│
│(remove chkpt)│  │(restore)    │
└──────┬───────┘  └──────┬──────┘
       │                 │
       ▼                 ▼
┌──────────────┐  ┌─────────────┐
│ git add .    │  │ Clean staged│
│ git commit   │  │ changes     │
└──────────────┘  └─────────────┘
```

## Audit System Architecture

### Directory Structure

```
audit-logs/
└── example.com_shannon-1234567890/
    │
    ├── session.json              # Atomic updates (crash-safe)
    │   {
    │     "id": "shannon-1234567890",
    │     "webUrl": "https://example.com",
    │     "startTime": 1234567890000,
    │     "status": "running",
    │     "phases": {
    │       "pre-recon": { "status": "completed", ... },
    │       "recon": { "status": "running", ... }
    │     },
    │     "agents": {
    │       "pre-recon": {
    │         "attempts": [
    │           { "number": 1, "duration_ms": 60000, "cost_usd": 0.52 }
    │         ]
    │       }
    │     },
    │     "summary": {
    │       "totalCostUsd": 1.30,
    │       "totalDurationMs": 150000
    │     }
    │   }
    │
    ├── workflow.log              # Append-only (crash-safe)
    │   [2025-02-28 10:00:00] Phase started: pre-recon
    │   [2025-02-28 10:00:05] Agent started: pre-recon (attempt 1)
    │   [2025-02-28 10:01:05] Agent completed: pre-recon (cost: $0.52)
    │   [2025-02-28 10:01:05] Phase completed: pre-recon
    │   ...
    │
    ├── agents/                   # Per-agent logs
    │   ├── pre-recon.log
    │   │   [Turn 1] Tool: Read file src/main.ts
    │   │   [Turn 2] Tool: Bash command nmap ...
    │   │   [Turn 3] Response: Identified Express.js app...
    │   │
    │   ├── recon.log
    │   └── ...
    │
    ├── prompts/                  # Reproducibility
    │   ├── pre-recon-code.txt    # Exact prompt used
    │   ├── recon.txt
    │   └── ...
    │
    └── deliverables/             # Agent outputs
        ├── pre_recon_report.md
        ├── recon_report.md
        ├── injection_vulnerability_queue.json
        ├── injection_exploitation_evidence.md
        └── comprehensive_security_assessment_report.md
```

### Crash-Safe Logging

```
┌────────────────────────────────────────────────────────────┐
│              Crash-Safe Audit Implementation                │
└────────────────────────────────────────────────────────────┘

APPEND-ONLY LOGGING (survives kill -9)
    │
    ▼
┌─────────────────────────────────────┐
│ Event occurs (tool use, response)   │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ Format as JSON line                 │
│ const line = JSON.stringify(event)  │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ fs.appendFile(logPath, line + '\n') │
│ Immediate flush to disk             │
└─────────────────────────────────────┘

ATOMIC SESSION UPDATES (no partial writes)
    │
    ▼
┌─────────────────────────────────────┐
│ Session state changes               │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ Write to temporary file             │
│ fs.writeFile(session.json.tmp, ...) │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ Atomic rename (OS-level operation)  │
│ fs.rename(tmp, session.json)        │
└─────────────────────────────────────┘

CONCURRENCY SAFETY (parallel agents)
    │
    ▼
┌─────────────────────────────────────┐
│ SessionMutex: Per-session lock      │
│ Only one agent updates session.json │
│ at a time                            │
└─────────────────────────────────────┘
```

## Configuration System

### Configuration Flow

```
┌────────────────────────────────────────────────────────────┐
│              Configuration Loading & Distribution           │
└────────────────────────────────────────────────────────────┘

1. USER PROVIDES CONFIG
   ┌────────────────────────────────┐
   │ my-app-config.yaml             │
   │ - authentication (form/SSO)    │
   │ - credentials                  │
   │ - TOTP secret                  │
   │ - login flow steps             │
   │ - rules (avoid/focus)          │
   └──────────────┬─────────────────┘
                  │
2. PARSE & VALIDATE
   ┌──────────────▼─────────────────┐
   │ parseConfig(configPath)        │
   │ - Load YAML                    │
   │ - Validate against JSON Schema │
   │ - Throw ConfigError on invalid │
   └──────────────┬─────────────────┘
                  │
3. DISTRIBUTE TO AGENTS
   ┌──────────────▼─────────────────┐
   │ distributeConfig(config)       │
   │                                │
   │ All agents get:                │
   │ - authentication               │
   │                                │
   │ Vuln/Exploit agents get:       │
   │ - rules (avoid/focus paths)    │
   │                                │
   │ Specific agents get:           │
   │ - custom_instructions          │
   └──────────────┬─────────────────┘
                  │
4. INJECT INTO PROMPTS
   ┌──────────────▼─────────────────┐
   │ loadPrompt(name, vars, config) │
   │ - Replace {{CONFIG_CONTEXT}}   │
   │ - Replace {{LOGIN_INSTRUCTIONS}}│
   │ - Inject auth details           │
   └────────────────────────────────┘
```

### Authentication Configuration

```yaml
# Form-based authentication example
authentication:
  login_type: form
  login_url: "https://app.com/login"
  credentials:
    username: "pentester@example.com"
    password: "Test123!"
    totp_secret: "JBSWY3DPEHPK3PXP"  # Base32 secret

  login_flow:
    - "Navigate to {{login_url}}"
    - "Type {{username}} into the email input field"
    - "Type {{password}} into the password input field"
    - "If 2FA prompt appears:"
    - "  Call generate_totp tool with secret: {{totp_secret}}"
    - "  Type the generated code into the 2FA input"
    - "Click the 'Sign In' button"

  success_condition:
    type: url_contains
    value: "/dashboard"

# SSO authentication example
authentication:
  login_type: sso
  login_url: "https://app.com/login"
  sso_provider: "google"
  credentials:
    email: "pentester@example.com"
    password: "GooglePassword123!"
    totp_secret: "ABCD1234..."  # Google account TOTP

  login_flow:
    - "Click 'Sign in with Google' button"
    - "In Google popup, type {{email}}"
    - "Click Next"
    - "Type {{password}}"
    - "Click Sign In"
    - "If 2FA required, use generate_totp tool"
    - "Wait for redirect back to application"

  success_condition:
    type: element_visible
    value: "user profile dropdown"
```

## Prompt Template System

### Template Loading Pipeline

```
┌────────────────────────────────────────────────────────────┐
│               Prompt Template Processing                    │
└────────────────────────────────────────────────────────────┘

1. SELECT TEMPLATE
   Agent name → Prompt file mapping
   ┌──────────────────────────────────┐
   │ 'pre-recon' → 'pre-recon-code.txt'│
   │ 'injection-vuln' → 'vuln-injection.txt'│
   │ 'injection-exploit' → 'exploit-injection.txt'│
   └──────────────┬───────────────────┘
                  │
2. LOAD TEMPLATE
   ┌──────────────▼───────────────────┐
   │ fs.readFile(promptPath)          │
   └──────────────┬───────────────────┘
                  │
3. VARIABLE SUBSTITUTION
   ┌──────────────▼───────────────────┐
   │ Replace placeholders:            │
   │ - {{TARGET_URL}} → "https://..." │
   │ - {{REPO_PATH}} → "/path/to/..."│
   │ - {{DELIVERABLE_PATH}} → "..."  │
   └──────────────┬───────────────────┘
                  │
4. INJECT SHARED PARTIALS
   ┌──────────────▼───────────────────┐
   │ Include shared content:          │
   │ - login-instructions.txt         │
   │ - deliverable-format.txt         │
   └──────────────┬───────────────────┘
                  │
5. INJECT CONFIG CONTEXT
   ┌──────────────▼───────────────────┐
   │ If config provided:              │
   │ - Authentication details         │
   │ - Login flow instructions        │
   │ - Rules (avoid/focus paths)      │
   └──────────────┬───────────────────┘
                  │
6. TESTING MODE?
   ┌──────────────▼───────────────────┐
   │ If PIPELINE_TESTING=true:        │
   │ - Use minimal prompt             │
   │ - Skip long context              │
   │ - Fast iteration                 │
   └──────────────┬───────────────────┘
                  │
7. RETURN FINAL PROMPT
   └─────────────────────────────────→
```

## Performance Optimization

### Parallelization Strategy

```
TRADITIONAL SEQUENTIAL EXECUTION
═════════════════════════════════════════════════════════════
pre-recon → recon → vuln1 → vuln2 → vuln3 → vuln4 → vuln5
                    ↓
                    exploit1 → exploit2 → exploit3 → exploit4 → exploit5
                    ↓
                    report
Total time: ~2.5 hours
Cost: $50

SHANNON PIPELINED PARALLEL EXECUTION
═════════════════════════════════════════════════════════════
pre-recon → recon → ┌─ vuln1 → exploit1 ─┐
                    ├─ vuln2 → exploit2 ─┤
                    ├─ vuln3 → exploit3 ─┤─→ report
                    ├─ vuln4 → exploit4 ─┤
                    └─ vuln5 → exploit5 ─┘
                    (all 5 pipelines run in parallel)
Total time: ~1 hour (60% faster)
Cost: $50 (same, but faster)

KEY BENEFITS:
- Each exploit starts immediately after its vuln completes
- No synchronization barrier between phases
- 5x speedup with same resource usage
- Graceful failure handling (one pipeline fails, others continue)
```

### Resource Management

```
┌────────────────────────────────────────────────────────────┐
│                  Resource Allocation                        │
└────────────────────────────────────────────────────────────┘

DOCKER CONTAINER LIMITS
├─ Shared Memory: 2GB (for Chromium/Playwright)
├─ IPC: host mode (better performance)
└─ Security: seccomp:unconfined (allow ptrace for debugging)

TEMPORAL HEARTBEAT
├─ Interval: 2 seconds
├─ Timeout: 10 minutes (production) / 5 minutes (testing)
└─ Purpose: Prevent false "dead worker" detection during:
   - Long file reads
   - Expensive git operations
   - Heavy AI inference
   - Browser rendering

CONCURRENT ACTIVITIES
├─ Max parallel: 5 (vuln/exploit pipelines)
├─ Heartbeat prevents timeout during resource contention
└─ Each activity has independent progress tracking
```

## Docker Architecture

### Service Dependencies

```
┌────────────────────────────────────────────────────────────┐
│                   Docker Service Graph                      │
└────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │  temporal   │
                    │  (server)   │
                    └──────┬──────┘
                           │ gRPC: 7233
                           │ Web UI: 8233
                    ┌──────▼──────┐
                    │  depends_on │
                    │  + healthy  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   worker    │
                    │  (Shannon)  │
                    └─────────────┘
                    Volumes:
                    - prompts (ro)
                    - audit-logs (rw)
                    - target-repo (rw)
                    - benchmarks (ro)

Optional router service (--profile router):
                    ┌─────────────┐
                    │   router    │
                    │ (multi-model)│
                    └─────────────┘
                    Port: 3456
                    Routes: worker → router → OpenAI/OpenRouter
```

### Volume Mounts

```
┌────────────────────────────────────────────────────────────┐
│                     Volume Strategy                         │
└────────────────────────────────────────────────────────────┘

HOST                                 CONTAINER
════════════════════════════════════════════════════════════════

./prompts/                     →     /app/prompts/
├─ pre-recon-code.txt                (read-only, templates)
├─ recon.txt
└─ vuln-*.txt

./audit-logs/                  →     /app/audit-logs/
└─ {session}/                        (read-write, session data)
   ├─ session.json
   ├─ workflow.log
   └─ agents/

${TARGET_REPO}                 →     /target-repo/
(user-provided path)                 (read-write, analyzed app)

${OUTPUT_DIR}                  →     /app/output/
(optional custom path)               (read-write, deliverables)

${BENCHMARKS_BASE}             →     /benchmarks/
(optional, for testing)              (read-only, test apps)

Temporal workflow state        →     temporal-data (named volume)
(persisted across restarts)
```

## CLI Interface

### Command Structure

```
┌────────────────────────────────────────────────────────────┐
│                   Shannon CLI Commands                      │
└────────────────────────────────────────────────────────────┘

./shannon start URL=<url> REPO=<path> [OPTIONS]
├─ Validates: URL, REPO, API keys
├─ Ensures: Docker containers running
├─ Submits: Workflow to Temporal
└─ Returns: Workflow ID for tracking

./shannon logs ID=<workflow-id>
├─ Auto-discovers: workflow.log location
├─ Tails: Real-time log output
└─ Shows: Phase transitions, agent progress

./shannon query ID=<workflow-id>
├─ Queries: Temporal workflow state
├─ Returns: Current phase, completed agents, metrics
└─ Format: JSON progress summary

./shannon stop [CLEAN=true]
├─ Stops: All Docker containers
├─ Preserves: Workflow data (default)
└─ Cleans: Removes volumes if CLEAN=true

./shannon help
└─ Shows: Usage and examples
```

### Workflow ID Format

```
Format: {hostname}_shannon-{timestamp}

Examples:
- example.com_shannon-1709123456789
- juice-shop.local_shannon-1709123456790
- localhost_shannon-1709123456791

Benefits:
✓ Human-readable
✓ Sortable by time
✓ Includes target identifier
✓ Unique across sessions
✓ Easy to grep in logs
```

## Error Handling
## 错误处理

<!--
中文说明：Shannon 实现了智能的错误分类和恢复机制：
1. 可重试错误（Retryable Errors）- 临时性错误，Temporal 会自动重试
   - BillingError（账单错误）- API 花费上限，等待 5-30 分钟后重试
   - RateLimitError（限流错误）- 请求过多，使用指数退避算法重试
   - TransientError（临时错误）- 服务器端问题，自动重试
   - NetworkError（网络错误）- 连接超时或 DNS 失败

2. 不可重试错误（Non-Retryable Errors）- 永久性错误，工作流立即失败
   - AuthenticationError（认证错误）- API 密钥无效
   - PermissionError（权限错误）- 权限不足
   - ConfigurationError（配置错误）- 配置文件无效
   - InvalidRequestError（请求错误）- 请求格式错误
-->

### Error Classification
### 错误分类

```
┌────────────────────────────────────────────────────────────┐
│               Error Taxonomy & Recovery                     │
└────────────────────────────────────────────────────────────┘

RETRYABLE ERRORS (Temporal will retry with backoff)
═══════════════════════════════════════════════════════════════
BillingError
├─ Cause: API spending cap reached
├─ Detection: "spending", "cap", "limit" in response
├─ Strategy: Wait 5-30 minutes for cap reset
└─ Recovery: Automatic (Temporal retry)

RateLimitError (429)
├─ Cause: Too many requests
├─ Detection: HTTP 429 status
├─ Strategy: Exponential backoff
└─ Recovery: Automatic (Temporal retry)

TransientError (5xx)
├─ Cause: Server-side issues
├─ Detection: HTTP 500-599 status
├─ Strategy: Retry with backoff
└─ Recovery: Automatic (Temporal retry)

NetworkError
├─ Cause: Connection timeout, DNS failure
├─ Detection: ECONNREFUSED, ETIMEDOUT
├─ Strategy: Retry with backoff
└─ Recovery: Automatic (Temporal retry)

OutputValidationError (limited retries)
├─ Cause: Agent didn't save required files
├─ Detection: Missing deliverable files
├─ Strategy: Retry up to 3 times (unlikely to self-heal)
└─ Recovery: Automatic (Temporal retry, max 3)

NON-RETRYABLE ERRORS (Workflow fails immediately)
═══════════════════════════════════════════════════════════════
AuthenticationError
├─ Cause: Invalid API key
├─ Detection: HTTP 401, "invalid_api_key"
└─ Recovery: User must fix API key

PermissionError
├─ Cause: Insufficient permissions
├─ Detection: HTTP 403, file access denied
└─ Recovery: User must grant permissions

ConfigurationError
├─ Cause: Invalid YAML, schema validation failed
├─ Detection: Parse error, ajv validation error
└─ Recovery: User must fix config file

InvalidRequestError
├─ Cause: Malformed request to API
├─ Detection: HTTP 400, parameter errors
└─ Recovery: Fix code (likely a bug)

ExecutionLimitError
├─ Cause: Max turns/time exceeded
├─ Detection: Reached maxTurns (10,000)
└─ Recovery: Likely infinite loop, needs investigation
```

### Recovery Flow

```
Agent execution fails
    │
    ▼
┌─────────────────────────────┐
│ Classify error              │
│ classifyErrorForTemporal()  │
└────────────┬────────────────┘
             │
    ┌────────┴────────┐
    │                 │
Retryable       Non-retryable
    │                 │
    ▼                 ▼
┌────────────┐   ┌──────────────┐
│ Rollback   │   │ Rollback     │
│ git stash  │   │ git stash pop│
│ pop        │   └──────┬───────┘
└─────┬──────┘          │
      │                 ▼
      │          ┌──────────────┐
      │          │ Fail workflow│
      │          │ immediately  │
      │          └──────────────┘
      │
      ▼
┌─────────────────────────────┐
│ Wait for retry interval     │
│ - Initial: 5 minutes        │
│ - Maximum: 30 minutes       │
│ - Backoff: 2x multiplier    │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│ Retry agent from clean      │
│ checkpoint                  │
└─────────────────────────────┘
```

## Benchmarking

### XBOW Benchmark Results

```
┌────────────────────────────────────────────────────────────┐
│           Shannon Lite Performance (XBOW)                   │
└────────────────────────────────────────────────────────────┘

Overall Success Rate: 96.15% (hint-free, source-aware)

Test Application Results:
═════════════════════════════════════════════════════════════

OWASP Juice Shop
├─ Vulnerabilities Found: 20+
├─ Critical Exploits:
│  ├─ SQL Injection → Full database exfiltration
│  ├─ Auth Bypass → Complete authentication bypass
│  ├─ Privilege Escalation → Admin account creation
│  └─ IDOR → Access any user's private data
└─ False Positives: 0 (XSS defenses correctly identified)

c{api}tal API (Checkmarx)
├─ Vulnerabilities Found: 15
├─ Critical Exploits:
│  ├─ Command Injection → Root-level access via debug endpoint
│  ├─ Auth Bypass → Legacy v1 API endpoint exploitation
│  ├─ Mass Assignment → User to admin privilege escalation
│  └─ Path Traversal → File system access
└─ False Positives: 0

OWASP crAPI
├─ Vulnerabilities Found: 15+
├─ Critical Exploits:
│  ├─ JWT Attacks → Algorithm confusion, alg:none, kid injection
│  ├─ SQL Injection → PostgreSQL database compromise
│  ├─ SSRF → Internal token forwarding to external service
│  └─ Rate Limiting Bypass → Brute force attack
└─ False Positives: 0

Key Metrics:
════════════════════════════════════════════════════════════
Average Time per Test: 1-1.5 hours
Average Cost per Test: $50 USD (Claude 4.5 Sonnet)
Accuracy: 96.15% (high precision, low false positives)
```

## Future Roadmap

### Planned Features

```
┌────────────────────────────────────────────────────────────┐
│                   Development Roadmap                       │
└────────────────────────────────────────────────────────────┘

PHASE 1: Additional Vulnerability Types (Q2 2025)
═══════════════════════════════════════════════════════════════
□ XML External Entity (XXE) Injection
□ Server-Side Template Injection (SSTI)
□ Insecure Deserialization
□ File Upload Vulnerabilities
□ WebSocket Security Issues

PHASE 2: Shannon Pro Features (Q2-Q3 2025)
═══════════════════════════════════════════════════════════════
□ LLM-based Data Flow Analysis Engine
   ├─ Inspired by LLMDFA paper (arxiv.org/abs/2402.10754)
   ├─ Graph-based whole-codebase analysis
   └─ Deeper vulnerability detection

□ CI/CD Integration
   ├─ GitHub Actions workflow
   ├─ GitLab CI pipeline
   └─ Jenkins plugin

□ Advanced Reporting
   ├─ OWASP Top 10 compliance mapping
   ├─ CVE/CWE classification
   └─ Risk scoring matrix

PHASE 3: Enterprise Features (Q3-Q4 2025)
═══════════════════════════════════════════════════════════════
□ Multi-tenant Architecture
□ RBAC & Access Controls
□ Compliance Automation (SOC 2, HIPAA)
□ Custom Rule Engine
□ Integration with Keygraph Platform

PHASE 4: Community & Ecosystem (Ongoing)
═══════════════════════════════════════════════════════════════
□ Plugin System for Custom Agents
□ Community-contributed Vulnerability Patterns
□ Integration with Security Tools (Burp, ZAP)
□ Multi-language Support (beyond Node.js)
```

## Key Takeaways
## 核心要点

<!--
中文说明：Shannon 的核心创新和特点总结：
1. 技术创新 - "No Exploit, No Report"方法论、流水线并行化、白盒+黑盒混合分析
2. 生产就绪 - 容器化部署、配置管理、错误处理、审计日志、成本和性能指标
3. 使用场景 - 作为持续的"红队"与开发"蓝队"配合，在每次构建时发现漏洞
-->

### Technical Innovation
### 技术创新

1. **"No Exploit, No Report" Methodology**
   - Only reports vulnerabilities that can be successfully exploited
   - Dramatically reduces false positives
   - Provides reproducible proof-of-concepts

2. **Pipelined Parallel Execution**
   - 5x speedup over sequential execution
   - No synchronization barrier between phases
   - Graceful failure handling

3. **White-box + Black-box Hybrid**
   - Source code analysis guides exploitation attempts
   - Browser automation validates real-world exploitability
   - High accuracy with low false positives

4. **Temporal-based Orchestration**
   - Production-grade reliability
   - Automatic crash recovery
   - Queryable real-time progress

5. **Crash-Safe Audit System**
   - Survives kill -9 and crashes
   - Complete reproducibility
   - Forensic-grade logs

### Production Readiness

✓ Containerized deployment
✓ Configuration management
✓ Error handling & retry logic
✓ Audit & compliance logging
✓ Cost & performance metrics
✓ Modular & extensible architecture

### Use Case

Shannon acts as a continuous "Red Team" to your development "Blue Team":
- Traditional: 1 pentest per year → 364 days of unknown vulnerabilities
- Shannon: Pentest every build → Ship with confidence

**Result:** Close the security gap between rapid development and manual security testing.
