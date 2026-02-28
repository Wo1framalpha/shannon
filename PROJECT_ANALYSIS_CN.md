# Shannon 项目工作原理深度分析

## 项目概述

Shannon 是一个基于 AI 的自动化渗透测试工具，专门用于 Web 应用的安全测试。它结合了白盒源代码分析和黑盒动态漏洞利用技术，能够自动发现并验证应用程序中的安全漏洞。

**核心特点：**
- 完全自动化的渗透测试流程
- 基于 Claude Agent SDK 的 AI 驱动分析
- 使用 Temporal 进行工作流编排
- 支持多种 OWASP 漏洞类型检测
- 提供可复现的漏洞利用证明

## 技术架构

### 1. 核心技术栈

```
┌─────────────────────────────────────────────────────────┐
│                    Shannon 架构                          │
├─────────────────────────────────────────────────────────┤
│  应用层                                                  │
│  - CLI 接口 (shannon 脚本)                               │
│  - 配置管理 (YAML + JSON Schema 验证)                    │
│  - 审计系统 (崩溃安全的日志记录)                          │
├─────────────────────────────────────────────────────────┤
│  编排层                                                  │
│  - Temporal Workflows (工作流定义)                       │
│  - Temporal Activities (活动实现)                        │
│  - Temporal Worker (工作进程)                            │
├─────────────────────────────────────────────────────────┤
│  执行层                                                  │
│  - Claude Agent SDK (AI 推理引擎)                        │
│  - Playwright MCP (浏览器自动化)                         │
│  - Shannon Helper MCP (辅助工具)                         │
│  - Git 检查点管理                                        │
├─────────────────────────────────────────────────────────┤
│  工具层                                                  │
│  - nmap (端口扫描)                                       │
│  - subfinder (子域名发现)                                │
│  - whatweb (Web 技术指纹识别)                            │
└─────────────────────────────────────────────────────────┘
```

### 2. 依赖关系

**主要依赖：**
- `@anthropic-ai/claude-agent-sdk` - AI 代理执行引擎
- `@temporalio/*` - 工作流编排框架
- `@playwright/mcp` - 浏览器自动化 MCP 服务器
- `ajv` - JSON Schema 验证
- `js-yaml` - YAML 配置解析
- `zx` - Shell 脚本执行

## 工作流程详解

### 阶段 1: 预侦察 (Pre-Reconnaissance)

**执行的 Agent:** `pre-recon`

**主要任务：**
1. 运行外部安全工具扫描
   - `nmap` - 网络端口扫描
   - `subfinder` - 子域名枚举
   - `whatweb` - Web 技术栈识别
2. 分析源代码结构
3. 识别应用技术栈
4. 生成初步的攻击面报告

**输出文件：** `deliverables/pre_recon_report.md`

**实现位置：** `src/temporal/activities.ts:309` (runPreReconAgent)

### 阶段 2: 侦察 (Reconnaissance)

**执行的 Agent:** `recon`

**主要任务：**
1. 分析预侦察阶段的发现
2. 使用浏览器自动化探索应用
3. 映射所有入口点和 API 端点
4. 识别身份验证机制
5. 构建完整的攻击面地图

**输出文件：** `deliverables/recon_report.md`

**实现位置：** `src/temporal/activities.ts:313` (runReconAgent)

### 阶段 3: 漏洞分析 (Vulnerability Analysis)

**并行执行的 5 个 Agent：**

1. **injection-vuln** - SQL 注入、命令注入分析
   - 数据流分析：追踪用户输入到危险函数
   - 识别不安全的数据库查询
   - 检测命令执行漏洞

2. **xss-vuln** - 跨站脚本漏洞分析
   - 追踪用户输入到输出点
   - 识别不安全的 HTML 渲染
   - 检测 DOM XSS 漏洞

3. **auth-vuln** - 身份验证漏洞分析
   - JWT 令牌安全性分析
   - 会话管理漏洞
   - 密码重置漏洞

4. **authz-vuln** - 授权漏洞分析
   - IDOR (不安全的直接对象引用)
   - 权限提升漏洞
   - 访问控制绕过

5. **ssrf-vuln** - 服务器端请求伪造分析
   - URL 参数注入点
   - 内部服务访问
   - 云元数据服务访问

**输出文件（每个 Agent 一个）：**
- `deliverables/injection_vulnerability_queue.json`
- `deliverables/xss_vulnerability_queue.json`
- `deliverables/auth_vulnerability_queue.json`
- `deliverables/authz_vulnerability_queue.json`
- `deliverables/ssrf_vulnerability_queue.json`

**实现位置：**
- `src/temporal/workflows.ts:198-226` (并行流水线)
- `src/temporal/activities.ts:317-334` (各个 vuln agent)

### 阶段 4: 漏洞利用 (Exploitation)

**条件执行机制：**

在执行每个漏洞利用 Agent 之前，系统会检查对应的漏洞队列：

```typescript
// src/queue-validation.ts
const decision = await checkExploitationQueue(activityInput, vulnType);
if (decision.shouldExploit) {
  // 只有发现漏洞时才运行利用 Agent
  exploitMetrics = await runExploitAgent();
}
```

**并行执行的 5 个 Exploit Agent：**

1. **injection-exploit** - 实际执行注入攻击
2. **xss-exploit** - 实际执行 XSS 攻击
3. **auth-exploit** - 实际执行认证绕过
4. **authz-exploit** - 实际执行授权绕过
5. **ssrf-exploit** - 实际执行 SSRF 攻击

**重要原则：** "No Exploit, No Report"
- 只有成功执行的漏洞利用才会被包含在最终报告中
- 这个原则大大降低了误报率

**输出文件（每个 Agent 一个）：**
- `deliverables/injection_exploitation_evidence.md`
- `deliverables/xss_exploitation_evidence.md`
- `deliverables/auth_exploitation_evidence.md`
- `deliverables/authz_exploitation_evidence.md`
- `deliverables/ssrf_exploitation_evidence.md`

**实现位置：**
- `src/temporal/workflows.ts:198-226` (并行流水线)
- `src/temporal/activities.ts:337-354` (各个 exploit agent)

### 阶段 5: 报告生成 (Reporting)

**执行的 Agent:** `report`

**主要任务：**
1. 汇总所有成功的漏洞利用证据
2. 生成执行摘要
3. 提供可复现的 PoC (概念验证)
4. 评估业务影响
5. 提供修复建议

**报告生成流程：**

```typescript
// 1. 汇总所有利用证据文件
await assembleReportActivity(activityInput);

// 2. AI Agent 添加执行摘要和清理
state.agentMetrics['report'] = await a.runReportAgent(activityInput);

// 3. 注入模型元数据
await injectReportMetadataActivity(activityInput);
```

**输出文件：** `deliverables/comprehensive_security_assessment_report.md`

**实现位置：**
- `src/temporal/workflows.ts:268-283` (报告阶段)
- `src/phases/reporting.ts` (报告汇总逻辑)

## Temporal 工作流编排

### 1. 工作流定义

**文件：** `src/temporal/workflows.ts`

**核心函数：** `pentestPipelineWorkflow()`

**工作流状态：**

```typescript
interface PipelineState {
  status: 'running' | 'completed' | 'failed';
  currentPhase: string | null;
  currentAgent: string | null;
  completedAgents: AgentName[];
  failedAgent: string | null;
  error: string | null;
  startTime: number;
  agentMetrics: Record<string, AgentMetrics>;
  summary: PipelineSummary | null;
}
```

**并行化策略：**

Shannon 使用流水线并行化来加速执行：

```typescript
// 流水线：vuln agent → 队列检查 → 条件性 exploit agent
// 5 个流水线并行运行，互不等待

const pipelineResults = await Promise.allSettled([
  runVulnExploitPipeline('injection', ...),
  runVulnExploitPipeline('xss', ...),
  runVulnExploitPipeline('auth', ...),
  runVulnExploitPipeline('ssrf', ...),
  runVulnExploitPipeline('authz', ...)
]);
```

这种设计使得：
- 每个 exploit agent 在其对应的 vuln agent 完成后立即开始
- 不需要等待所有 vuln agents 完成
- 速度提升约 5 倍

### 2. 活动实现

**文件：** `src/temporal/activities.ts`

**核心函数：** `runAgentActivity()`

**活动生命周期：**

```
1. 加载配置文件（如果提供）
2. 初始化审计会话
3. 加载 AI 提示模板
4. 创建 Git 检查点
5. 执行 AI Agent (单次尝试)
6. 验证输出
7. 成功时提交 Git，失败时回滚
8. 错误分类（用于 Temporal 重试）
```

**心跳机制：**

```typescript
// 每 2 秒发送心跳，通知 Temporal 工作进程仍在运行
const heartbeatInterval = setInterval(() => {
  heartbeat({
    agent: agentName,
    elapsedSeconds: elapsed,
    attempt: attemptNumber
  });
}, HEARTBEAT_INTERVAL_MS);
```

### 3. 重试策略

**生产环境重试配置：**

```typescript
const PRODUCTION_RETRY = {
  initialInterval: '5 minutes',      // 初始重试间隔
  maximumInterval: '30 minutes',     // 最大重试间隔
  backoffCoefficient: 2,             // 指数退避系数
  maximumAttempts: 50,               // 最大重试次数
  nonRetryableErrorTypes: [          // 不可重试的错误类型
    'AuthenticationError',
    'PermissionError',
    'InvalidRequestError',
    // ... 更多
  ]
};
```

**错误分类：**

```typescript
// src/error-handling.ts
export function classifyErrorForTemporal(error: unknown) {
  // 可重试：账单错误、429 限流、5xx 错误
  if (isBillingError(error)) {
    return { retryable: true, type: 'BillingError' };
  }

  // 不可重试：认证失败、配置错误
  if (isAuthError(error)) {
    return { retryable: false, type: 'AuthenticationError' };
  }

  // ... 更多分类
}
```

### 4. 崩溃恢复

Temporal 的关键优势是崩溃恢复能力：

**场景 1：Worker 进程崩溃**
```
时间 T0: Worker 运行 injection-vuln agent
时间 T1: Worker 崩溃（内存不足、网络断开等）
时间 T2: Worker 重启
时间 T3: Temporal 自动从 T1 恢复工作流
       → 重新运行 injection-vuln agent
       → 继续后续阶段
```

**场景 2：花费限额错误**
```
时间 T0: Agent 遇到 API 花费限额
时间 T1: Temporal 将工作流暂停 5 分钟（初始重试间隔）
时间 T2: 5 分钟后自动重试
时间 T3: 如果仍失败，等待 10 分钟（指数退避）
时间 T4: 继续重试，直到成功或达到最大尝试次数（50）
```

## Claude Agent SDK 集成

### 1. Agent 执行引擎

**文件：** `src/ai/claude-executor.ts`

**核心函数：** `runClaudePrompt()`

**SDK 配置：**

```typescript
const options = {
  model: 'claude-sonnet-4-5-20250929',  // 模型名称
  maxTurns: 10_000,                      // 最大对话轮数
  cwd: sourceDir,                        // 工作目录（源代码位置）
  permissionMode: 'bypassPermissions',   // 跳过权限检查（自动化测试）
  mcpServers: {                          // MCP 服务器配置
    'shannon-helper': helperServer,      // 辅助工具 MCP
    'playwright-browser': playwrightMcp  // 浏览器自动化 MCP
  }
};
```

### 2. MCP 服务器集成

**Shannon Helper MCP** (`mcp-server/`):
- TOTP 生成（双因素认证）
- 配置文件访问
- 自定义辅助函数

**Playwright MCP** (`@playwright/mcp`):
- 浏览器自动化
- 页面导航
- 表单填写
- JavaScript 执行
- 截图和快照

**Docker 环境下的 Chromium 配置：**

```typescript
// Docker 使用系统 Chromium
if (isDocker) {
  mcpArgs.push('--executable-path', '/usr/bin/chromium-browser');
  mcpArgs.push('--browser', 'chromium');
}
```

### 3. 消息处理

**文件：** `src/ai/message-handlers.ts`

**消息流处理：**

```typescript
for await (const message of query({ prompt, options })) {
  switch (message.type) {
    case 'assistant':
      // AI 响应消息
      turnCount++;
      break;

    case 'tool_use':
      // AI 调用工具
      auditLogger.logToolUse(message);
      break;

    case 'complete':
      // 完成
      return { result, cost };

    case 'error':
      // 错误
      throw new Error(message.error);
  }
}
```

### 4. 花费上限防护

**多层防护机制：**

```typescript
// 防护 1: 消息处理器中的检测
if (detectApiError(message)) {
  apiErrorDetected = true;
}

// 防护 2: 完成后的检测（低轮数 + $0 成本）
if (turnCount <= 2 && totalCost === 0) {
  const looksLikeBillingError = /spending|cap|limit/i.test(result);
  if (looksLikeBillingError) {
    throw new PentestError('Spending cap reached', 'billing', true);
  }
}

// 防护 3: 活动层的检测
if (result.success && result.turns <= 2 && result.cost === 0) {
  // 回滚并重试
}
```

## Git 检查点管理

**文件：** `src/utils/git-manager.ts`

**工作流：**

```
尝试开始
  ↓
创建检查点: git stash push -m "checkpoint:${agent}:attempt-${N}"
  ↓
执行 Agent
  ↓
成功？
  ├─ 是 → git stash drop (丢弃检查点)
  │        git add . && git commit -m "${agent} completed"
  │
  └─ 否 → git stash pop (恢复检查点)
           清理工作区
```

**好处：**
- 每次尝试都有干净的起点
- 失败不会污染工作区
- 可以追踪成功的更改历史

## 审计与指标系统

### 1. 会话管理

**文件：** `src/audit/audit-session.ts`

**类：** `AuditSession`

**目录结构：**

```
audit-logs/
└── {hostname}_{sessionId}/
    ├── session.json          # 会话元数据和指标
    ├── workflow.log          # 统一的工作流日志
    ├── agents/
    │   ├── pre-recon.log
    │   ├── recon.log
    │   ├── injection-vuln.log
    │   └── ...
    ├── prompts/
    │   ├── pre-recon-code.txt
    │   ├── recon.txt
    │   └── ...
    └── deliverables/
        └── comprehensive_security_assessment_report.md
```

### 2. 崩溃安全日志

**特性：**

1. **追加式日志** - 立即刷新，能承受 kill -9
2. **原子写入** - session.json 使用原子替换
3. **事件驱动** - 记录 tool_start, tool_end, llm_response
4. **互斥锁** - SessionMutex 防止并发竞态条件

**实现：**

```typescript
// 追加式日志（崩溃安全）
async appendLog(agent: string, event: LogEvent) {
  const line = JSON.stringify(event) + '\n';
  await fs.appendFile(logPath, line);
  // 立即刷新到磁盘
}

// 原子式会话更新
async updateSession(updates: Partial<SessionData>) {
  const tempPath = sessionPath + '.tmp';
  await fs.writeFile(tempPath, JSON.stringify(data));
  await fs.rename(tempPath, sessionPath);  // 原子操作
}
```

### 3. 指标聚合

**文件：** `src/audit/metrics-tracker.ts`

**聚合层次：**

1. **Agent 级别**
   - 持续时间
   - 成本
   - 轮数
   - 尝试次数

2. **阶段级别**
   - 所有 agent 的总成本
   - 阶段总时长
   - 代理计数

3. **工作流级别**
   - 总成本
   - 总时长
   - 完成的代理数

## 配置系统

### 1. YAML 配置

**文件：** `configs/example-config.yaml`

**配置结构：**

```yaml
authentication:
  login_type: form | sso | api | basic
  login_url: "https://app.com/login"
  credentials:
    username: "test@example.com"
    password: "password"
    totp_secret: "BASE32SECRET"  # 可选：双因素认证

  login_flow:  # AI 的登录步骤
    - "Type $username into email field"
    - "Type $password into password field"
    - "If TOTP prompt appears, generate code with generate_totp tool"
    - "Click Sign In button"

  success_condition:
    type: url_contains | element_visible | text_present
    value: "/dashboard"

rules:
  avoid:  # AI 应该避免的区域
    - description: "Don't test logout"
      type: path
      url_path: "/logout"

  focus:  # AI 应该重点测试的区域
    - description: "Focus on API endpoints"
      type: path
      url_path: "/api"
```

### 2. JSON Schema 验证

**文件：** `configs/config-schema.json`

**验证器：**

```typescript
// src/config-parser.ts
const ajv = new Ajv();
addFormats(ajv);
const validate = ajv.compile(configSchema);

export async function parseConfig(configPath: string) {
  const config = yaml.load(await fs.readFile(configPath, 'utf8'));

  if (!validate(config)) {
    throw new ConfigError(
      `Invalid config: ${ajv.errorsText(validate.errors)}`
    );
  }

  return config;
}
```

### 3. 配置分发

**策略：** 不同的 agent 接收不同的配置部分

```typescript
// src/config-parser.ts
export function distributeConfig(config: Config): DistributedConfig {
  return {
    // 所有 agent 都接收 authentication
    authentication: config.authentication,

    // 只有 vuln/exploit agents 接收 rules
    rules: config.rules,

    // 针对特定 agent 的自定义配置
    customInstructions: config.custom_instructions
  };
}
```

## 提示模板系统

### 1. 模板结构

**目录：** `prompts/`

**文件：**
- `pre-recon-code.txt` - 预侦察提示
- `recon.txt` - 侦察提示
- `vuln-injection.txt` - 注入漏洞分析提示
- `vuln-xss.txt` - XSS 漏洞分析提示
- `exploit-injection.txt` - 注入利用提示
- `shared/login-instructions.txt` - 共享的登录说明

### 2. 变量替换

**文件：** `src/prompts/prompt-manager.ts`

**变量：**
- `{{TARGET_URL}}` - 目标 URL
- `{{CONFIG_CONTEXT}}` - 配置上下文（认证、规则）
- `{{LOGIN_INSTRUCTIONS}}` - 登录流程说明
- `{{DELIVERABLE_PATH}}` - 输出文件路径

**示例：**

```typescript
export async function loadPrompt(
  promptName: string,
  vars: { webUrl: string, repoPath: string },
  config: DistributedConfig | null,
  testingMode: boolean
) {
  let template = await fs.readFile(promptPath, 'utf8');

  // 替换变量
  template = template.replace(/\{\{TARGET_URL\}\}/g, vars.webUrl);

  // 注入配置上下文
  if (config?.authentication) {
    const context = formatAuthContext(config.authentication);
    template = template.replace(/\{\{CONFIG_CONTEXT\}\}/g, context);
  }

  // 测试模式：使用最小提示
  if (testingMode) {
    template = getMinimalPrompt(promptName);
  }

  return template;
}
```

## Docker 容器化

### 1. 服务架构

**文件：** `docker-compose.yml`

**服务：**

```yaml
services:
  temporal:
    # Temporal 服务器（工作流引擎）
    image: temporalio/temporal:latest
    ports:
      - "7233:7233"   # gRPC API
      - "8233:8233"   # Web UI
    volumes:
      - temporal-data:/home/temporal  # 持久化工作流状态

  worker:
    # Shannon worker（执行 agents）
    build: .
    environment:
      - ANTHROPIC_API_KEY
      - TEMPORAL_ADDRESS=temporal:7233
    volumes:
      - ./prompts:/app/prompts           # 提示模板
      - ./audit-logs:/app/audit-logs     # 审计日志
      - ${TARGET_REPO}:/target-repo      # 目标代码库
    shm_size: 2gb                        # Chromium 需要
    ipc: host

  router:
    # 可选：多模型路由
    profiles: ["router"]
    command: "ccr start"
    ports:
      - "3456:3456"
```

### 2. Dockerfile

**文件：** `Dockerfile`

**关键步骤：**

```dockerfile
FROM node:20-slim

# 安装 Chromium（用于 Playwright）
RUN apt-get update && apt-get install -y \
    chromium \
    chromium-driver \
    # ... 更多依赖

# 安装安全工具
RUN apt-get install -y \
    nmap \
    subfinder \
    whatweb

# 设置环境变量
ENV SHANNON_DOCKER=true
ENV PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1

# 复制应用代码
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 运行 worker
CMD ["node", "dist/temporal/worker.js"]
```

### 3. 网络配置

**本地应用测试：**

Docker 容器无法访问主机的 `localhost`。使用 `host.docker.internal`：

```bash
./shannon start \
  URL=http://host.docker.internal:3000 \
  REPO=/path/to/repo
```

## CLI 接口

### 1. Shannon 脚本

**文件：** `shannon`

**命令：**

```bash
# 启动渗透测试
./shannon start URL=<url> REPO=<path>

# 查看日志
./shannon logs ID=<workflow-id>

# 查询进度
./shannon query ID=<workflow-id>

# 停止所有容器
./shannon stop

# 完全清理（包括数据）
./shannon stop CLEAN=true
```

### 2. 工作流 ID 生成

**格式：** `{hostname}_shannon-{timestamp}`

**示例：** `example.com_shannon-1234567890`

这使得可以轻松识别和追踪工作流。

### 3. 进度监控

**方式 1：实时日志**

```bash
./shannon logs ID=example.com_shannon-1234567890
```

**方式 2：查询状态**

```bash
./shannon query ID=shannon-1234567890
```

输出：
```json
{
  "status": "running",
  "currentPhase": "vulnerability-exploitation",
  "currentAgent": "pipelines",
  "completedAgents": ["pre-recon", "recon"],
  "elapsedMs": 180000,
  "agentMetrics": {
    "pre-recon": { "durationMs": 60000, "costUsd": 0.52 },
    "recon": { "durationMs": 90000, "costUsd": 0.78 }
  }
}
```

**方式 3：Temporal Web UI**

访问 `http://localhost:8233` 查看：
- 工作流历史
- 活动执行详情
- 重试历史
- 事件时间线

## 路由器模式（实验性）

### 1. 多模型支持

Shannon 可以通过 `claude-code-router` 使用替代 AI 提供商：

**支持的提供商：**
- OpenAI (`gpt-5.2`, `gpt-5-mini`)
- OpenRouter (`google/gemini-3-flash-preview`)

### 2. 配置

**文件：** `configs/router-config.json`

```json
{
  "host": "${HOST:-0.0.0.0}",
  "port": 3456,
  "apiKey": "shannon-router-key",
  "providers": {
    "openai": {
      "apiKey": "${OPENAI_API_KEY}",
      "baseUrl": "https://api.openai.com/v1"
    },
    "openrouter": {
      "apiKey": "${OPENROUTER_API_KEY}",
      "baseUrl": "https://openrouter.ai/api/v1"
    }
  },
  "defaultModel": "${ROUTER_DEFAULT:-openai,gpt-4o}"
}
```

### 3. 启动路由器模式

```bash
# 在 .env 中设置
OPENAI_API_KEY=sk-...
ROUTER_DEFAULT=openai,gpt-5.2

# 使用路由器启动
./shannon start URL=<url> REPO=<path> ROUTER=true
```

**工作原理：**

```
Shannon Agent → Router (localhost:3456) → OpenAI/OpenRouter
```

## 安全考虑

### 1. 破坏性操作警告

Shannon 执行实际的攻击，可能会造成破坏性影响：

**不要在生产环境运行！**

可能的影响：
- 创建新用户账户
- 修改或删除数据
- 破坏测试账户
- 触发意外的副作用

### 2. 授权要求

**法律要求：**
- 必须拥有目标系统或获得明确的书面授权
- 未经授权的扫描是非法的（参见 CFAA 法案）

### 3. 成本控制

**典型成本：** ~$50 USD（使用 Claude 4.5 Sonnet）
**典型时长：** 1-1.5 小时

**花费限额保护：**
- 多层检测机制
- 自动重试（5-30 分钟退避）
- 防止意外的高额账单

## 性能优化

### 1. 并行化

**传统顺序执行：**
```
pre-recon → recon → [vuln1 → vuln2 → vuln3 → vuln4 → vuln5] → [exploit1 → exploit2 → ...] → report
总时间：~2.5 小时
```

**Shannon 流水线并行化：**
```
pre-recon → recon → [
  vuln1 → exploit1
  vuln2 → exploit2  ← 5 个流水线并行
  vuln3 → exploit3
  vuln4 → exploit4
  vuln5 → exploit5
] → report
总时间：~1 小时
```

**速度提升：** 5 倍

### 2. 资源管理

**Temporal 心跳：**
- 每 2 秒发送心跳
- 防止长时间运行的活动被视为死亡
- 在资源受限的 worker 上支持并发

**Git 检查点：**
- 快速回滚（git stash pop）
- 避免昂贵的克隆操作
- 保持工作区清洁

## 测试模式

### 1. 流水线测试模式

```bash
./shannon start URL=<url> REPO=<path> PIPELINE_TESTING=true
```

**变化：**
- 使用最小提示（更快）
- 快速重试间隔（10 秒而非 5 分钟）
- 跳过外部工具执行

**用途：**
- 快速开发迭代
- 提示模板测试
- CI/CD 集成测试

### 2. 离线开发

可以在没有外部工具的情况下开发：

```typescript
// src/tool-checker.ts
export function validateTools() {
  if (process.env.PIPELINE_TESTING) {
    return { allAvailable: true, missing: [] };
  }
  // 正常验证...
}
```

## 基准测试结果

**XBOW 基准测试** - Shannon Lite 成绩：96.15%

测试应用：
- OWASP Juice Shop - 20+ 关键漏洞
- c{api}tal API - 15 个关键漏洞
- OWASP crAPI - 15+ 关键漏洞

**关键成就：**
- 完整的认证绕过
- 数据库泄露（通过注入）
- 权限提升
- SSRF 攻击
- 零误报（XSS 防护正确识别）

## 未来路线图

### 计划中的漏洞类型

- XML 外部实体 (XXE)
- 服务器端模板注入 (SSTI)
- 不安全的反序列化
- 业务逻辑漏洞
- WebSocket 安全

### Shannon Pro 功能

**高级功能：**
- 基于 LLM 的数据流分析引擎
- 全代码库图分析
- CI/CD 集成
- 企业支持
- 更深入的漏洞检测

## 总结

Shannon 是一个复杂的、生产级的 AI 驱动渗透测试框架，结合了：

1. **强大的编排** - Temporal 工作流具有崩溃恢复能力
2. **智能分析** - Claude Agent SDK 具有深度代码理解能力
3. **实际验证** - 通过浏览器自动化进行真实的漏洞利用
4. **企业就绪** - 审计日志、指标、配置管理
5. **可扩展** - 模块化架构，易于添加新的漏洞类型

**核心创新：**
- "No Exploit, No Report" 方法论 → 零误报
- 流水线并行化 → 5 倍速度提升
- 白盒 + 黑盒混合 → 高准确性
- Temporal 编排 → 生产级可靠性

这使得 Shannon 能够充当持续的"红队"，与"蓝队"开发人员一起工作，在每次构建时发现漏洞，而不是一年一次的渗透测试。
