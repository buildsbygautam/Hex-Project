# 🎯 Hex AI - Professional Red Teaming Architecture

## 🔴 This is for PROFESSIONAL Red Teamers

You're right - we need the REAL deal, not toy examples. Here's the proper architecture:

---

## 🏗️ Professional Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│  ┌────────────────────────────────────────────────┐     │
│  │  Chat Interface (existing)                     │     │
│  └────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────┐     │
│  │  Terminal/Console Window (NEW)                 │     │
│  │  - Real-time command output                    │     │
│  │  - Syntax highlighting                         │     │
│  │  - Copy output                                 │     │
│  │  - Export logs                                 │     │
│  └────────────────────────────────────────────────┘     │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │ WebSocket / SSE
                    │
┌───────────────────▼─────────────────────────────────────┐
│              Backend Execution Server                    │
│  ┌────────────────────────────────────────────────┐     │
│  │  Command Executor                              │     │
│  │  - Nmap, Metasploit, Burp Suite               │     │
│  │  - SQLMap, Nikto, Gobuster                    │     │
│  │  - Hydra, John, Hashcat                       │     │
│  └────────────────────────────────────────────────┘     │
│  ┌────────────────────────────────────────────────┐     │
│  │  Security Layer                                │     │
│  │  - Command validation                          │     │
│  │  - User authentication                         │     │
│  │  - Resource limits                             │     │
│  └────────────────────────────────────────────────┘     │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │ Spawns processes
                    │
┌───────────────────▼─────────────────────────────────────┐
│              Docker Container (Kali Linux)               │
│  - Nmap, Metasploit Framework                           │
│  - SQLMap, Nikto, Dirb, Gobuster                        │
│  - Hydra, John the Ripper, Hashcat                      │
│  - Burp Suite CLI, ZAP                                  │
│  - All standard pentesting tools                        │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 Terminal UI Component

### What Users Will See

```
┌─────────────────────────────────────────────────────────┐
│ 🤖 Hex AI Chat                                          │
├─────────────────────────────────────────────────────────┤
│ USER: Scan 192.168.1.1 with nmap                        │
│                                                          │
│ HEX: I'll run an nmap scan for you...                   │
│                                                          │
│ ┌─ Terminal ────────────────────────────────────────┐   │
│ │ $ nmap -sV -sC -p- 192.168.1.1                   │   │
│ │                                                   │   │
│ │ Starting Nmap 7.94 ( https://nmap.org )          │   │
│ │ Nmap scan report for 192.168.1.1                 │   │
│ │ Host is up (0.0012s latency).                    │   │
│ │ Not shown: 65530 closed ports                    │   │
│ │ PORT    STATE SERVICE  VERSION                   │   │
│ │ 22/tcp  open  ssh      OpenSSH 8.2p1            │   │
│ │ 80/tcp  open  http     Apache httpd 2.4.41      │   │
│ │ 443/tcp open  ssl/http Apache httpd 2.4.41      │   │
│ │                                                   │   │
│ │ Nmap done: 1 IP address scanned in 124.32s      │   │
│ │                                                   │   │
│ │ ✅ Completed in 2m 4s                            │   │
│ └───────────────────────────────────────────────────┘   │
│                                                          │
│ HEX: I found 3 open ports. SSH on 22, HTTP on 80,      │
│      and HTTPS on 443. Would you like me to run         │
│      vulnerability scans on these services?              │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 Backend Options (Choose One)

### Option 1: Separate Node.js Server (Recommended for Pro Use)

**Pros:**
- ✅ Full control over execution
- ✅ WebSocket support for real-time output
- ✅ Can run long-running scans
- ✅ Better resource management
- ✅ Can be deployed on VPS/EC2

**Cons:**
- ❌ Requires separate deployment
- ❌ More complex infrastructure

**Files:**
```
server/
├── package.json
├── execution-server.js        # Main server
├── tools/
│   ├── nmap-executor.js
│   ├── metasploit-executor.js
│   ├── sqlmap-executor.js
│   └── general-executor.js
├── security/
│   ├── auth-middleware.js
│   └── command-validator.js
└── docker/
    └── kali-tools.Dockerfile
```

### Option 2: Netlify Functions (Limited but Easier)

**Pros:**
- ✅ Easy deployment (already on Netlify)
- ✅ Automatic scaling
- ✅ No server management

**Cons:**
- ❌ 10-second timeout limit (kills long scans)
- ❌ Limited CPU/RAM
- ❌ Cannot run Docker
- ❌ Not suitable for heavy tools

**Verdict:** ❌ Not recommended for professional red teaming

---

## 🚀 Recommended Stack

### Backend: Node.js + Express + WebSocket

```javascript
// server/execution-server.js
const express = require('express');
const WebSocket = require('ws');
const { spawn } = require('child_process');

const app = express();
const wss = new WebSocket.Server({ port: 8081 });

// Execute command and stream output
wss.on('connection', (ws) => {
  ws.on('message', async (message) => {
    const { command, args, userId } = JSON.parse(message);
    
    // Validate user (check Supabase auth)
    // Validate command (whitelist)
    
    // Execute in Docker container
    const proc = spawn('docker', [
      'exec', 'kali-container',
      command, ...args
    ]);
    
    // Stream stdout to client
    proc.stdout.on('data', (data) => {
      ws.send(JSON.stringify({
        type: 'stdout',
        data: data.toString()
      }));
    });
    
    // Stream stderr to client
    proc.stderr.on('data', (data) => {
      ws.send(JSON.stringify({
        type: 'stderr',
        data: data.toString()
      }));
    });
    
    // Send completion status
    proc.on('close', (code) => {
      ws.send(JSON.stringify({
        type: 'complete',
        exitCode: code
      }));
    });
  });
});
```

### Frontend: React Component

```typescript
// src/components/TerminalWindow.tsx
import { useEffect, useState, useRef } from 'react';

interface TerminalOutput {
  type: 'stdout' | 'stderr' | 'info';
  content: string;
  timestamp: Date;
}

export const TerminalWindow = () => {
  const [output, setOutput] = useState<TerminalOutput[]>([]);
  const [isRunning, setIsRunning] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    // Connect to backend WebSocket
    wsRef.current = new WebSocket('ws://localhost:8081');
    
    wsRef.current.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      if (message.type === 'stdout') {
        setOutput(prev => [...prev, {
          type: 'stdout',
          content: message.data,
          timestamp: new Date()
        }]);
      } else if (message.type === 'complete') {
        setIsRunning(false);
      }
    };
    
    return () => wsRef.current?.close();
  }, []);

  return (
    <div className="terminal-window bg-black rounded-lg border border-green-500/30 p-4 font-mono text-sm">
      <div className="flex items-center gap-2 mb-3 pb-2 border-b border-green-500/20">
        <Terminal className="h-4 w-4 text-green-400" />
        <span className="text-green-300">Terminal Output</span>
        {isRunning && (
          <div className="ml-auto flex items-center gap-2">
            <div className="animate-spin h-3 w-3 border-2 border-green-400 border-t-transparent rounded-full" />
            <span className="text-green-400 text-xs">Running...</span>
          </div>
        )}
      </div>
      
      <div className="space-y-1 max-h-96 overflow-y-auto">
        {output.map((line, i) => (
          <div
            key={i}
            className={
              line.type === 'stderr'
                ? 'text-red-400'
                : 'text-green-300'
            }
          >
            {line.content}
          </div>
        ))}
      </div>
    </div>
  );
};
```

---

## 🐳 Docker Setup

### Kali Linux Container with Security Tools

```dockerfile
# server/docker/kali-tools.Dockerfile
FROM kalilinux/kali-rolling:latest

# Update and install pentesting tools
RUN apt-get update && apt-get install -y \
    nmap \
    metasploit-framework \
    sqlmap \
    nikto \
    dirb \
    gobuster \
    hydra \
    john \
    hashcat \
    wireshark-cli \
    tcpdump \
    aircrack-ng \
    wpscan \
    whatweb \
    enum4linux \
    smbmap \
    crackmapexec \
    exploitdb \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user for security
RUN useradd -m -s /bin/bash hexagent && \
    echo "hexagent:hexagent" | chpasswd

# Set resource limits
ENV MAX_MEMORY=2g
ENV MAX_CPU=2

# Working directory
WORKDIR /home/hexagent

# Switch to non-root user
USER hexagent

CMD ["/bin/bash"]
```

### Build and Run

```bash
# Build the container
docker build -t hex-kali-tools -f server/docker/kali-tools.Dockerfile .

# Run the container
docker run -d \
  --name kali-container \
  --memory="2g" \
  --cpus="2" \
  --network="bridge" \
  hex-kali-tools

# Keep it running
docker exec -it kali-container /bin/bash
```

---

## 🔒 Security Implementation

### Command Validator

```typescript
// server/security/command-validator.ts
export class ProfessionalCommandValidator {
  // Whitelist of allowed security tools
  private allowedTools = new Set([
    // Network scanning
    'nmap', 'masscan', 'zmap',
    
    // Web application testing
    'sqlmap', 'nikto', 'gobuster', 'dirb', 'wpscan', 'whatweb',
    
    // Exploitation
    'msfconsole', 'msfvenom',
    
    // Password cracking
    'hydra', 'john', 'hashcat',
    
    // Wireless
    'aircrack-ng', 'reaver', 'wifite',
    
    // Enumeration
    'enum4linux', 'smbmap', 'crackmapexec',
    
    // General utilities
    'curl', 'wget', 'dig', 'nslookup', 'whois', 'ping'
  ]);

  // Patterns that should NEVER be allowed
  private blockedPatterns = [
    /rm\s+-rf\s+\//, // Delete system
    /mkfs/, // Format disk
    /dd\s+if=.*of=\/dev/, // Overwrite disk
    /:(){ :|:& };:/, // Fork bomb
    /chmod\s+777\s+\//, // Dangerous permissions on root
    /\/etc\/shadow/, // Shadow file access
    /\/etc\/passwd/, // Password file modification
  ];

  validate(command: string, args: string[]): ValidationResult {
    const fullCommand = `${command} ${args.join(' ')}`;
    
    // 1. Check if tool is allowed
    if (!this.allowedTools.has(command)) {
      return {
        valid: false,
        reason: `Tool '${command}' not allowed. Contact admin to add it.`
      };
    }
    
    // 2. Check for blocked patterns
    for (const pattern of this.blockedPatterns) {
      if (pattern.test(fullCommand)) {
        return {
          valid: false,
          reason: 'Command contains dangerous pattern'
        };
      }
    }
    
    // 3. Check for privilege escalation attempts
    if (args.includes('sudo') || args.includes('su')) {
      return {
        valid: false,
        reason: 'Privilege escalation not allowed'
      };
    }
    
    // 4. Validate targets (must be in allowed scope)
    const ipPattern = /\b(\d{1,3}\.){3}\d{1,3}\b/g;
    const ips = fullCommand.match(ipPattern) || [];
    
    for (const ip of ips) {
      if (!this.isAllowedTarget(ip)) {
        return {
          valid: false,
          reason: `Target ${ip} not in allowed scope`
        };
      }
    }
    
    return { valid: true };
  }

  private isAllowedTarget(ip: string): boolean {
    // Check if IP is in user's allowed scope
    // This would check against their Supabase profile settings
    const privateRanges = [
      /^10\./, // 10.0.0.0/8
      /^172\.(1[6-9]|2[0-9]|3[01])\./, // 172.16.0.0/12
      /^192\.168\./, // 192.168.0.0/16
    ];
    
    // For now, only allow private IPs (user's own lab)
    return privateRanges.some(pattern => pattern.test(ip));
  }
}
```

---

## 🛠️ Professional Tools Integration

### 1. Nmap Integration

```typescript
// server/tools/nmap-executor.ts
export class NmapExecutor {
  async scan(target: string, scanType: string, options: any) {
    const args = this.buildNmapCommand(target, scanType, options);
    
    return this.executeInDocker('nmap', args);
  }

  private buildNmapCommand(target: string, scanType: string, options: any): string[] {
    const baseArgs = [];
    
    switch (scanType) {
      case 'quick':
        baseArgs.push('-sn'); // Ping scan
        break;
      case 'port':
        baseArgs.push('-sV', '-sC'); // Version + default scripts
        if (options.ports) baseArgs.push('-p', options.ports);
        break;
      case 'full':
        baseArgs.push('-A', '-T4', '-p-'); // Aggressive, all ports
        break;
      case 'stealth':
        baseArgs.push('-sS', '-T2'); // SYN scan, slower
        break;
      case 'vuln':
        baseArgs.push('--script', 'vuln'); // Vulnerability scan
        break;
    }
    
    baseArgs.push(target);
    return baseArgs;
  }
}
```

### 2. SQLMap Integration

```typescript
// server/tools/sqlmap-executor.ts
export class SQLMapExecutor {
  async test(url: string, options: any) {
    const args = ['--url', url, '--batch'];
    
    if (options.dbs) args.push('--dbs');
    if (options.tables) args.push('--tables');
    if (options.dump) args.push('--dump');
    if (options.level) args.push('--level', options.level);
    if (options.risk) args.push('--risk', options.risk);
    
    return this.executeInDocker('sqlmap', args);
  }
}
```

### 3. Metasploit Integration

```typescript
// server/tools/metasploit-executor.ts
export class MetasploitExecutor {
  async search(query: string) {
    const args = ['-q', '-x', `search ${query}; exit`];
    return this.executeInDocker('msfconsole', args);
  }

  async useExploit(exploit: string, options: any) {
    const commands = [
      `use ${exploit}`,
      `set RHOSTS ${options.target}`,
      `set RPORT ${options.port || 80}`,
      `check`,
      'exit'
    ];
    
    const args = ['-q', '-x', commands.join('; ')];
    return this.executeInDocker('msfconsole', args);
  }
}
```

---

## 📊 DeepSeek Tool Schemas for Pro Tools

```typescript
export const professionalSecurityTools = [
  {
    type: 'function',
    function: {
      name: 'nmap_scan',
      description: 'Run Nmap network scanner for reconnaissance',
      parameters: {
        type: 'object',
        properties: {
          target: {
            type: 'string',
            description: 'Target IP or domain'
          },
          scan_type: {
            type: 'string',
            enum: ['quick', 'port', 'full', 'stealth', 'vuln'],
            description: 'Type of scan to perform'
          },
          ports: {
            type: 'string',
            description: 'Port range (e.g., "80,443" or "1-1000")'
          }
        },
        required: ['target', 'scan_type']
      }
    }
  },
  {
    type: 'function',
    function: {
      name: 'sqlmap_test',
      description: 'Test for SQL injection vulnerabilities',
      parameters: {
        type: 'object',
        properties: {
          url: {
            type: 'string',
            description: 'Target URL to test'
          },
          dbs: {
            type: 'boolean',
            description: 'Enumerate databases'
          },
          level: {
            type: 'integer',
            minimum: 1,
            maximum: 5,
            description: 'Level of tests (1-5)'
          }
        },
        required: ['url']
      }
    }
  },
  {
    type: 'function',
    function: {
      name: 'gobuster_dir',
      description: 'Directory and file brute-forcing',
      parameters: {
        type: 'object',
        properties: {
          url: {
            type: 'string',
            description: 'Target URL'
          },
          wordlist: {
            type: 'string',
            enum: ['common', 'medium', 'large'],
            description: 'Wordlist size'
          },
          extensions: {
            type: 'string',
            description: 'File extensions (e.g., "php,html,js")'
          }
        },
        required: ['url']
      }
    }
  },
  {
    type: 'function',
    function: {
      name: 'metasploit_search',
      description: 'Search Metasploit database for exploits',
      parameters: {
        type: 'object',
        properties: {
          query: {
            type: 'string',
            description: 'Search term (e.g., "apache", "smb")'
          }
        },
        required: ['query']
      }
    }
  }
];
```

---

## 🚀 Deployment Strategy

### Development

```bash
# Start Docker container
docker-compose up -d

# Start backend server
cd server
npm install
npm run dev

# Start frontend (separate terminal)
npm run dev
```

### Production

**Option 1: VPS (DigitalOcean, AWS EC2)**
- Deploy Docker container
- Run backend server with PM2
- Frontend on Netlify (existing)
- WebSocket proxy through Nginx

**Option 2: Cloud Run (Google Cloud)**
- Containerize everything
- Auto-scaling
- Pay per use

---

## 📝 Implementation Timeline

### Week 1: Foundation
- ✅ Terminal UI component
- ✅ Docker container setup
- ✅ Basic command execution
- ✅ WebSocket communication

### Week 2: Tool Integration
- ✅ Nmap integration
- ✅ SQLMap integration
- ✅ Gobuster integration
- ✅ Command validation

### Week 3: DeepSeek Integration
- ✅ Tool calling in Index.tsx
- ✅ Tool schemas
- ✅ Response handling
- ✅ Error management

### Week 4: Polish & Deploy
- ✅ UI improvements
- ✅ Security hardening
- ✅ Documentation
- ✅ Production deployment

---

## ❓ Questions Before We Start

1. **Deployment**: Do you want to deploy the backend on:
   - VPS (DigitalOcean/AWS) - Full control
   - Your local machine initially - Easier testing
   - Docker Desktop locally - Simplest start

2. **Access Control**: Should tool execution be:
   - Premium only?
   - All authenticated users?
   - Specific permission levels?

3. **Scope**: What tools are highest priority?
   - Nmap (network scanning)
   - SQLMap (SQL injection)
   - Gobuster/Dirb (directory brute-forcing)
   - Metasploit (exploitation)
   - All of the above?

**What do you think?** Let me know your preferences and I'll start building! 🚀



