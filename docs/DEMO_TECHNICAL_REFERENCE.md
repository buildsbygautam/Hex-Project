# 🔧 Hex AI - Technical Reference with Diagrams

Complete technical documentation for the Hex AI demo.

---

## 📐 **ARCHITECTURE OVERVIEW**

```
                    ┌─────────────────────────────────────────────┐
                    │          USER INTERFACE                     │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  React Frontend (TypeScript + Vite)   │  │
                    │  │  - Chat Interface (Index.tsx)         │  │
                    │  │  - Terminal Window (TerminalWindow.tsx)│  │
                    │  │  - Tool Execution Hook                │  │
                    │  │    (use-tool-execution.ts)            │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ HTTP (DeepSeek API)
                                       │ WebSocket (Execution Server)
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
        ┌───────────▼──────────┐          ┌───────────────▼───────────┐
        │   DeepSeek API       │          │   Backend Server         │
        │   Function Calling   │          │   (Node.js)              │
        │   - Tool Schemas      │          │   - WebSocket            │
        │   - Tool Calls        │          │   - Auth (JWT)           │
        │                       │          │   - Docker Exec           │
        └───────────────────────┘          └───────────────┬───────────┘
                                                           │
                                                           │ Docker Exec
                                                           │
                                         ┌─────────────────▼─────────────────┐
                                         │   Docker Container                 │
                                         │   (Kali Linux)                     │
                                         │   - Nmap                           │
                                         │   - SQLMap                         │
                                         │   - Gobuster                       │
                                         │   - Security Tools                 │
                                         └───────────────────────────────────┘
```

---

## 🔄 **FEATURE 1: TOOL SCHEMA & FUNCTION CALLING**

### **Data Flow Diagram**

```
                    ┌─────────────────┐
                    │   User Input    │
                    │  "Scan target"  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────────────────┐
                    │  Index.tsx: sendMessage()   │
                    │  - Builds message history   │
                    │  - Includes tool schemas    │
                    │  - Calls DeepSeek API       │
                    └────────┬────────────────────┘
                             │
                             │ POST /v1/chat/completions
                             │ {
                             │   model: 'deepseek-chat',
                             │   messages: [...],
                             │   tools: professionalSecurityTools,
                             │   tool_choice: 'auto'
                             │ }
                             │
                             ▼
                    ┌─────────────────────────────┐
                    │  DeepSeek API                │
                    │  - Analyzes request          │
                    │  - Matches to tool schema    │
                    │  - Returns tool_calls        │
                    └────────┬────────────────────┘
                             │
                             │ Stream Response
                             │ {
                             │   tool_calls: [{
                             │     id: 'call_123',
                             │     function: {
                             │       name: 'nmap_scan',
                             │       arguments: '{"target":"192.168.1.1","scan_type":"quick"}'
                             │     }
                             │   }]
                             │ }
                             │
                             ▼
                    ┌─────────────────────────────┐
                    │  Index.tsx: Stream Handler  │
                    │  - Accumulates tool_calls    │
                    │  - Parses arguments         │
                    │  - Calls executeTool()       │
                    └────────┬────────────────────┘
                             │
                             ▼
                    ┌─────────────────────────────┐
                    │  useToolExecution:          │
                    │  executeTool()              │
                    │  - Maps tool name to command│
                    │  - Builds command + args    │
                    │  - Sends via WebSocket       │
                    └─────────────────────────────┘
```

### **Code Snippets**

**Tool Schema** (`src/lib/tools-schema.ts`): JSON Schema format with `name`, `description`, `parameters`. DeepSeek uses these schemas to match user requests to tools.

**Stream Accumulation** (`src/pages/Index.tsx`): `accumulatedToolCalls[index].function.arguments += delta.function.arguments` - accumulates streaming JSON arguments piece by piece.

**Command Mapping** (`src/lib/tools-schema.ts`): `toolCommandMap['nmap_scan']` → `'nmap'`. `buildCommand()` transforms `scan_type: 'quick'` → `-F` flag.

---

## 🌐 **FEATURE 2: WEBSOCKET & REAL-TIME STREAMING**

### **Data Flow Diagram**

```
                    ┌─────────────────────────────────────────────┐
                    │  Frontend: useToolExecution Hook               │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  executeTool(toolName, args)          │  │
                    │  │  1. Build command from tool schema    │  │
                    │  │  2. Generate executionId              │  │
                    │  │  3. Send via WebSocket                 │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ WebSocket Message
                                       │ {
                                       │   type: 'execute',
                                       │   payload: {
                                       │     command: 'nmap',
                                       │     args: ['-F', '192.168.1.100'],
                                       │     executionId: 'exec_123456'
                                       │   }
                                       │ }
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Backend: server/index.js                    │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  WebSocket Handler                     │  │
                    │  │  1. Authenticate (JWT)                 │  │
                    │  │  2. Execute in Docker                  │  │
                    │  │  3. Stream output                      │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ Docker Exec
                                       │ spawn('docker', ['exec', 'hex-kali-tools', ...])
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Docker Container: hex-kali-tools            │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Process: nmap -F 192.168.1.100      │  │
                    │  │  stdout.on('data') → Stream to WS     │  │
                    │  │  stderr.on('data') → Stream to WS      │  │
                    │  │  on('close') → Send completion        │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ WebSocket Messages (streaming)
                                       │ { type: 'output', payload: {...} }
                                       │ { type: 'output', payload: {...} }
                                       │ { type: 'complete', payload: {...} }
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Frontend: WebSocket Handler                  │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  ws.onmessage()                       │  │
                    │  │  - Parse message                      │  │
                    │  │  - Update terminal outputs             │  │
                    │  │  - Trigger onComplete callback        │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  TerminalWindow Component                    │
                    │  - Displays outputs in real-time            │
                    │  - Auto-scrolls to bottom                   │
                    │  - Shows command input                      │
                    └─────────────────────────────────────────────┘
```

### **Code Snippets**

**Frontend** (`use-tool-execution.ts`): `ws.onopen` → sends `{ type: 'auth', payload: { token } }`. `executeTool()` sends `{ type: 'execute', payload: { command, args, executionId } }`.

**Backend** (`server/index.js`): `authenticateUser(token)` validates JWT via Supabase. `spawn('docker', ['exec', 'hex-kali-tools', ...])` runs command. `proc.stdout.on('data')` streams to WebSocket as `{ type: 'output', payload: { data, outputType } }`.

---

## 🔄 **FEATURE 3: AUTONOMOUS ERROR ITERATION**

### **Data Flow Diagram**

```
                    ┌─────────────────────────────────────────────┐
                    │  Command Execution (Docker)                  │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  nmap --invalid-flag 192.168.1.100    │  │
                    │  │  ❌ Error: Unrecognized option         │  │
                    │  │     '--invalid-flag'                   │  │
                    │  │  Exit Code: 255                        │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ WebSocket: complete
                                       │ { type: 'complete',
                                       │   payload: { exitCode: 255 } }
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Frontend: onComplete Callback               │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  if (exitCode !== 0) {                 │  │
                    │  │    // Extract error output              │  │
                    │  │    const errorOutput = outputs          │  │
                    │  │      .filter(o => o.type === 'stderr') │  │
                    │  │      .slice(-10)                       │  │
                    │  │      .map(o => o.content)              │  │
                    │  │      .join('\n');                     │  │
                    │  │                                         │  │
                    │  │    // Build error message               │  │
                    │  │    const errorMessage =                │  │
                    │  │      `Command failed...`                │  │
                    │  │                                         │  │
                    │  │    // Auto-trigger AI analysis          │  │
                    │  │    sendMessage(false, true,             │  │
                    │  │                  errorMessage);         │  │
                    │  │  }                                      │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ sendMessage(autoTrigger=true)
                                       │ Bypasses input validation
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  DeepSeek API: Error Analysis               │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Input: "Command failed with exit      │  │
                    │  │         code 255. Error: Unrecognized  │  │
                    │  │         option '--invalid-flag'.        │  │
                    │  │         Analyze and suggest a fix."    │  │
                    │  │                                         │  │
                    │  │  AI Analysis:                            │  │
                    │  │  - Identifies syntax error               │  │
                    │  │  - Understands Nmap doesn't recognize  │  │
                    │  │    flag                                 │  │
                    │  │  - Suggests correct syntax               │  │
                    │  │  - Provides corrected command            │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       │ AI Response
                                       │ "The command failed because
                                       │  '--invalid-flag' is not a
                                       │  valid Nmap option. Try:
                                       │  nmap -F 192.168.1.100"
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  User Sees: AI Analysis + Fix                │
                    │  - Option 1: User manually retries           │
                    │  - Option 2: AI can auto-retry               │
                    │    (if configured)                          │
                    └─────────────────────────────────────────────┘
```

### **Code Snippets**

**Error Detection** (`Index.tsx`): `onComplete(exitCode, outputs)` → if `exitCode !== 0`, extracts last 10 stderr messages, builds error context, calls `sendMessage(false, true, errorMessage)`.

**Auto-Trigger** (`Index.tsx`): `sendMessage(autoTrigger: true, directMessage)` bypasses input validation. Error message includes exit code, full error output, and instructions for AI analysis.

**Success Handling**: On `exitCode === 0`, extracts last 2000 chars of output, sends to AI for report generation.

---

## 🏗️ **FEATURE 4: MCP (MODEL CONTEXT PROTOCOL) ARCHITECTURE**

### **MCP Structure Diagram**

```
                    ┌─────────────────────────────────────────────┐
                    │          MCP ARCHITECTURE                   │
                    │                                              │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Tool Definition Layer (MCP Compliant)│  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  JSON Schema Format             │  │  │
                    │  │  │  - name: string                 │  │  │
                    │  │  │  - description: string          │  │  │
                    │  │  │  - parameters: JSON Schema      │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    │                    │                         │
                    │                    ▼                         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Tool Mapping Layer                   │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  toolCommandMap: {              │  │  │
                    │  │  │    'nmap_scan': 'nmap',         │  │  │
                    │  │  │    'sqlmap_test': 'sqlmap',    │  │  │
                    │  │  │    ...                           │  │  │
                    │  │  │  }                               │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    │                    │                         │
                    │                    ▼                         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Command Building Layer               │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  buildCommand(toolName, args)   │  │  │
                    │  │  │  - Maps tool name to command    │  │  │
                    │  │  │  - Transforms args to CLI flags │  │  │
                    │  │  │  - Returns { command, args }    │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    │                    │                         │
                    │                    ▼                         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Execution Interface (Standardized)    │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  executeTool(toolName, args)     │  │  │
                    │  │  │  → WebSocket → Backend → Docker  │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    │                    │                         │
                    │                    ▼                         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Result Handling (Structured)         │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  TerminalOutput {               │  │  │
                    │  │  │    type: 'stdout'|'stderr'|'info'│  │  │
                    │  │  │    content: string               │  │  │
                    │  │  │    timestamp: Date                │  │  │
                    │  │  │  }                               │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    └─────────────────────────────────────────────┘
```

### **Code Snippets**

**MCP Tool Definition**: `{ type: 'function', function: { name, description, parameters: JSONSchema } }` - standard format for function calling.

**Tool Registry**: `professionalSecurityTools` array contains all tool schemas. `toolCommandMap` maps tool names to CLI commands.

**Extensibility**: Add new tool = 1) Define schema, 2) Add to registry, 3) Map to command, 4) Implement `buildCommand` logic. Execution interface stays the same.

---

## 🐳 **FEATURE 5: DOCKER EXECUTION**

### **Docker Architecture Diagram**

```
                    ┌─────────────────────────────────────────────┐
                    │  Backend Server (Node.js)                    │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  spawn('docker', ['exec', '-i',        │  │
                    │  │         'hex-kali-tools', ...])         │  │
                    │  │  - Executes command as non-root user   │  │
                    │  │    (hexagent)                          │  │
                    │  │  - Streams stdout/stderr               │  │
                    │  │  - Handles process lifecycle           │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                       │
                       │ Docker Exec API
                       │
                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Docker Container: hex-kali-tools            │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  Base Image:                          │  │
                    │  │  kalilinux/kali-rolling:latest        │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  Installed Tools:               │  │  │
                    │  │  │  - Nmap                         │  │  │
                    │  │  │  - SQLMap                       │  │  │
                    │  │  │  - Gobuster                     │  │  │
                    │  │  │  - Nikto                        │  │  │
                    │  │  │  - Hydra                        │  │  │
                    │  │  │  - ... (15+ tools)              │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  │  ┌─────────────────────────────────┐  │  │
                    │  │  │  User: hexagent (non-root)     │  │  │
                    │  │  │  Working Directory:            │  │  │
                    │  │  │    /home/hexagent/workspace    │  │  │
                    │  │  │  Network: Isolated               │  │  │
                    │  │  └─────────────────────────────────┘  │  │
                    │  └───────────────────────────────────────┘  │
                    └─────────────────────────────────────────────┘
```

### **Code Snippets**

**Dockerfile**: Base `kalilinux/kali-rolling`, installs security tools, creates `hexagent` non-root user, sets `WORKDIR /home/hexagent/workspace`.

**Execution**: `spawn('docker', ['exec', '-i', 'hex-kali-tools', 'su', '-', 'hexagent', '-c', command])` - runs as non-root user in isolated container.

**Capabilities**: `cap_add: [NET_RAW, NET_ADMIN, SYS_PTRACE]` - minimal permissions for network scanning tools.

---

## 📊 **FEATURE 6: REPORT GENERATION**

### **Report Generation Flow**

```
                    ┌─────────────────────────────────────────────┐
                    │  Tool Execution Completes (exitCode: 0)      │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  onComplete Callback                         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  if (exitCode === 0) {                 │  │
                    │  │    const allOutput = outputs           │  │
                    │  │      .map(o => o.content)              │  │
                    │  │      .join('\n')                       │  │
                    │  │      .slice(-2000);                    │  │
                    │  │                                         │  │
                    │  │    const reportMessage =                │  │
                    │  │      `Tool execution completed.       │  │
                    │  │       Analyze output and generate       │  │
                    │  │       professional report.`;           │  │
                    │  │                                         │  │
                    │  │    sendMessage(false, true,             │  │
                    │  │                  reportMessage);        │  │
                    │  │  }                                      │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  DeepSeek API: Report Generation             │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  System Prompt:                        │  │
                    │  │  "Generate professional security       │  │
                    │  │   assessment report"                  │  │
                    │  │                                         │  │
                    │  │  AI Analyzes:                          │  │
                    │  │  - Open ports                          │  │
                    │  │  - Services detected                   │  │
                    │  │  - Vulnerabilities                     │  │
                    │  │  - Risk assessment                     │  │
                    │  │  - Recommendations                     │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Generated Report (Markdown)                │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  # Security Assessment Report          │  │
                    │  │                                         │  │
                    │  │  ## Executive Summary                  │  │
                    │  │  ...                                   │  │
                    │  │                                         │  │
                    │  │  ## Findings                           │  │
                    │  │  ...                                   │  │
                    │  │                                         │  │
                    │  │  ## Recommendations                   │  │
                    │  │  ...                                   │  │
                    │  └───────────────────────────────────────┘  │
                    └─────────────────────────────────────────────┘
```

---

## 🔐 **SECURITY ARCHITECTURE**

### **Security Layers Diagram**

```
                    ┌─────────────────────────────────────────────┐
                    │  Security Layer 1: Authentication           │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  - JWT token validation (Supabase)    │  │
                    │  │  - WebSocket authentication            │  │
                    │  │  - User permission checks             │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Security Layer 2: Input Validation         │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  - Command whitelisting                 │  │
                    │  │  - Parameter sanitization              │  │
                    │  │  - Tool-specific validation            │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Security Layer 3: Container Isolation      │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  - Docker containerization             │  │
                    │  │  - Non-root user (hexagent)            │  │
                    │  │  - Network isolation                   │  │
                    │  │  - Resource limits (CPU, memory)        │  │
                    │  └───────────────────────────────────────┘  │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  Security Layer 4: Process Management        │
                    │  ┌───────────────────────────────────────┐  │
                    │  │  - Process timeout (future)             │  │
                    │  │  - Process killing on cancel            │  │
                    │  │  - Audit logging (future)               │  │
                    │  └───────────────────────────────────────┘  │
                    └─────────────────────────────────────────────┘
```

---

## 📝 **QUICK REFERENCE**

### **File Structure**
```
src/
├── pages/
│   └── Index.tsx              # Main chat interface, tool execution orchestration
├── hooks/
│   └── use-tool-execution.ts  # WebSocket connection, tool execution
├── components/
│   └── TerminalWindow.tsx     # Terminal UI component
└── lib/
    └── tools-schema.ts        # Tool schemas, command mapping

server/
├── index.js                   # WebSocket server, Docker execution
└── docker/
    ├── Dockerfile.kali        # Kali Linux container
    └── docker-compose.yml     # Container configuration
```

### **Key Functions**
- `executeTool(toolName, args)` - Execute tool via WebSocket
- `buildCommand(toolName, args)` - Map tool schema to CLI command
- `onComplete(exitCode, outputs)` - Handle completion, trigger error iteration
- `sendMessage(autoTrigger, directMessage)` - Send message to AI (with auto-trigger support)

### **Message Types**
- `auth` - WebSocket authentication
- `execute` - Execute command
- `cancel` - Cancel execution
- `output` - Stream output
- `complete` - Execution complete

---

---

## 🔐 **FEATURE 7: AUTHENTICATION & AUTHORIZATION**

### **GitHub OAuth + Supabase Flow**

```
                    ┌─────────────┐
                    │ User Clicks │
                    │ "Sign in"   │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Frontend: useAuth Hook      │
                    │  ┌─────────────────────────┐ │
                    │  │ signInWithGitHub()      │ │
                    │  │ supabase.auth.          │ │
                    │  │   signInWithOAuth        │ │
                    │  │   provider: 'github'     │ │
                    │  └─────────────────────────┘ │
                    └──────┬───────────────────────┘
                           │
                           │ Redirect to GitHub
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  GitHub OAuth                │
                    │  - User authorizes app       │
                    │  - Returns auth code         │
                    └──────┬───────────────────────┘
                           │
                           │ Redirect to Supabase
                           │ /auth/v1/callback
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Supabase Auth              │
                    │  ┌─────────────────────────┐ │
                    │  │ 1. Exchange code for     │ │
                    │  │    token                 │ │
                    │  │ 2. Create/update user    │ │
                    │  │ 3. Create profile record │ │
                    │  │ 4. Return JWT session   │ │
                    │  └─────────────────────────┘ │
                    └──────┬───────────────────────┘
                           │
                           │ JWT Session
                           │ { access_token, user }
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Frontend: AuthCallback     │
                    │  ┌─────────────────────────┐ │
                    │  │ supabase.auth.           │ │
                    │  │   getSession()           │ │
                    │  │ - Loads user profile     │ │
                    │  │ - Loads daily usage      │ │
                    │  │ - Redirects to home       │ │
                    │  └─────────────────────────┘ │
                    └──────────────────────────────┘
```

### **Database Schema**

**User Profiles** (`user_profiles`):
- `id` (UUID, FK to `auth.users`)
- `subscription_status` ('free' | 'premium')
- `subscription_start_date`, `subscription_end_date`
- `github_username`, `full_name`, `avatar_url`

**Daily Usage** (`daily_usage`):
- `user_id`, `usage_date`, `message_count`
- Tracks free tier limits (10 messages/day)

### **Code Snippets**

**Sign In** (`use-auth.ts`): `supabase.auth.signInWithOAuth({ provider: 'github' })` redirects to GitHub, then to Supabase callback.

**Session Management** (`use-auth.ts`): `supabase.auth.onAuthStateChange()` subscribes to auth events. `loadUserData()` fetches profile and daily usage in parallel.

**WebSocket Auth** (`server/index.js`): `authenticateUser(token)` validates JWT via `supabase.auth.getUser(token)`. WebSocket requires auth before execution.

---

## 💳 **FEATURE 8: PAYMENT SYSTEM (INSTASEND)**

### **Payment Flow Diagram**

```
                    ┌─────────────┐
                    │ User Clicks │
                    │ "Upgrade"   │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Billing.tsx                 │
                    │  ┌─────────────────────────┐ │
                    │  │ Initialize InstaSend    │ │
                    │  │   SDK                    │ │
                    │  │ new IntaSend({           │ │
                    │  │   publicAPIKey,         │ │
                    │  │   live: true             │ │
                    │  │ })                       │ │
                    │  └─────────────────────────┘ │
                    └──────┬───────────────────────┘
                           │
                           │ User completes payment
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  InstaSend SDK               │
                    │  - Processes M-Pesa/other   │
                    │  - Triggers COMPLETE event   │
                    └──────┬───────────────────────┘
                           │
                           │ Event: COMPLETE
                           │ { invoice_id, amount,
                           │   api_ref }
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Frontend: on("COMPLETE")    │
                    │  ┌─────────────────────────┐ │
                    │  │ 1. Create transaction    │ │
                    │  │    record                │ │
                    │  │    billingFunctions.     │ │
                    │  │    createTransaction()   │ │
                    │  │ 2. Update status to       │ │
                    │  │    'completed'            │ │
                    │  │ 3. Upgrade user to        │ │
                    │  │    premium                │ │
                    │  │    upgradeUserToPremium() │ │
                    │  │ 4. Refresh profile        │ │
                    │  └─────────────────────────┘ │
                    └──────┬───────────────────────┘
                           │
                           │ Webhook (background)
                           │
                           ▼
                    ┌─────────────────────────────┐
                    │  Supabase Edge Function       │
                    │  /functions/instasend-webhook │
                    │  ┌─────────────────────────┐ │
                    │  │ 1. Verify signature     │ │
                    │  │ 2. Extract userId from   │ │
                    │  │    api_ref               │ │
                    │  │ 3. Update subscription   │ │
                    │  │    status to 'premium'   │ │
                    │  │ 4. Set end_date          │ │
                    │  │    (+30 days)            │ │
                    │  └─────────────────────────┘ │
                    └──────────────────────────────┘
```

### **Payment Architecture**

**InstaSend Integration**:
- Frontend SDK: Handles payment UI and callbacks
- Webhook: Backup verification via Supabase Edge Function
- Transaction Tracking: All payments logged in `billing_transactions` table

**Subscription Management**:
- Premium users: `subscription_status = 'premium'`, unlimited messages
- Free users: `subscription_status = 'free'`, 10 messages/day limit
- Auto-expiry: `checkAndExpireSubscriptions()` RPC function runs daily

### **Code Snippets**

**Payment Init** (`Billing.tsx`): `new IntaSend({ publicAPIKey, live: true })` initializes SDK. `.on("COMPLETE")` handles success.

**Transaction Creation**: `billingFunctions.createTransaction(userId, 3.00, 'subscription')` creates record. `upgradeUserToPremium()` sets `subscription_end_date = now + 30 days`.

**Webhook** (`supabase/functions/instasend-webhook/index.ts`): Verifies HMAC signature, extracts `userId` from `api_ref` (format: `hex_premium_{userId}_{timestamp}`), updates profile.

**Usage Check** (`supabase.ts`): `getDailyUsage()` checks `subscription_status`. Premium = unlimited. Free = checks `daily_usage.message_count < 10`.

---

## 🔄 **COMPLETE SYSTEM FLOW**

### **End-to-End Architecture**

```
                    ┌─────────────────────────────────────────────┐
                    │  USER AUTHENTICATION                        │
                    │  GitHub OAuth → Supabase → JWT Session     │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  USAGE VALIDATION                            │
                    │  Check subscription_status & daily_usage    │
                    │  Premium: unlimited | Free: 10/day limit    │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  USER INPUT → DEEPSEEK API                   │
                    │  - Includes tool schemas                    │
                    │  - DeepSeek returns tool_calls              │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  TOOL EXECUTION                              │
                    │  buildCommand() → WebSocket → Backend →     │
                    │  Docker                                      │
                    │  Stream output back via WebSocket           │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  ERROR ITERATION / REPORT GENERATION         │
                    │  exitCode !== 0 → Auto-trigger AI error    │
                    │    analysis                                  │
                    │  exitCode === 0 → Auto-generate security    │
                    │    report                                    │
                    └─────────────────────────────────────────────┘
```

---

## 📝 **QUICK REFERENCE**

### **File Structure**
```
src/
├── pages/
│   ├── Index.tsx              # Main chat, tool execution orchestration
│   ├── Billing.tsx             # Payment page (InstaSend integration)
│   └── AuthCallback.tsx        # OAuth callback handler
├── hooks/
│   ├── use-tool-execution.ts  # WebSocket, tool execution
│   └── use-auth.ts            # Authentication state management
├── components/
│   └── TerminalWindow.tsx     # Terminal UI component
└── lib/
    ├── tools-schema.ts        # Tool schemas, command mapping
    └── supabase.ts            # Supabase client, auth/billing functions

server/
├── index.js                   # WebSocket server, Docker execution
└── docker/
    ├── Dockerfile.kali        # Kali Linux container
    └── docker-compose.yml     # Container configuration

supabase/
└── functions/
    └── instasend-webhook/     # Payment webhook handler
```

### **Key Functions**
- `executeTool(toolName, args)` - Execute tool via WebSocket
- `buildCommand(toolName, args)` - Map tool schema to CLI command
- `onComplete(exitCode, outputs)` - Handle completion, trigger error iteration
- `sendMessage(autoTrigger, directMessage)` - Send message to AI (with auto-trigger support)
- `signInWithGitHub()` - GitHub OAuth authentication
- `upgradeUserToPremium()` - Upgrade subscription after payment

### **Message Types**
- `auth` - WebSocket authentication
- `execute` - Execute command
- `cancel` - Cancel execution
- `output` - Stream output
- `complete` - Execution complete

### **Database Tables**
- `auth.users` - Supabase managed user accounts
- `user_profiles` - Custom user data, subscription status
- `daily_usage` - Daily message count tracking
- `billing_transactions` - Payment transaction records

---

**Complete technical reference for the Hex AI demo. 🚀**

*This document supports the demo script at `docs/DEMO_SCRIPT.md`*
