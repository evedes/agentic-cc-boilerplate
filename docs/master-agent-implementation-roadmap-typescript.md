# Master Agent Implementation Roadmap (TypeScript)

## From Architecture to Working Code

This document bridges the gap between our multi-agent system architecture and a functional master agent that you can spawn and interact with using TypeScript/Node.js.

## Quick Start Goal

**What we're building first**: A minimal master agent that:
1. Spawns in a zellij session
2. Accepts user commands via CLI
3. Manages its own environment
4. Provides visual feedback of its actions

## Implementation Path

### Step 1: Basic Master Agent Script (TypeScript)
```typescript
// src/masterAgent.ts - Minimal viable master agent

import * as readline from 'readline';
import { EventEmitter } from 'events';

interface AgentState {
  agentId: string;
  status: 'idle' | 'processing' | 'error';
  context: Record<string, any>;
}

class MasterAgent extends EventEmitter {
  private state: AgentState;
  private agents: Map<string, any>;

  constructor() {
    super();
    this.state = {
      agentId: 'master-001',
      status: 'idle',
      context: {}
    };
    this.agents = new Map();
  }

  async handleUserInput(message: string): Promise<string> {
    if (message.startsWith('/')) {
      return await this.handleCommand(message);
    } else {
      return await this.processTask(message);
    }
  }

  private async handleCommand(command: string): Promise<string> {
    const [cmd, ...args] = command.split(' ');

    switch (cmd) {
      case '/status':
        return `Master Agent Status: ${this.state.status}`;
      case '/help':
        return 'Available commands: /status, /spawn, /list, /help';
      case '/spawn':
        return await this.spawnAgent(args[0]);
      case '/list':
        return this.listAgents();
      default:
        return 'Unknown command. Type /help for available commands.';
    }
  }

  private async processTask(task: string): Promise<string> {
    this.state.status = 'processing';
    this.emit('taskReceived', task);

    // Here we'll add task decomposition logic
    const response = `Understood: '${task}'. Breaking down into subtasks...`;

    this.state.status = 'idle';
    return response;
  }

  private async spawnAgent(agentType?: string): Promise<string> {
    if (!agentType) {
      return 'Please specify agent type: frontend, backend, etc.';
    }

    // Future: Actually spawn agent in new zellij pane
    this.agents.set(`${agentType}-001`, {
      type: agentType,
      status: 'active',
      spawnedAt: new Date()
    });

    return `Would spawn ${agentType} agent (zellij integration pending)`;
  }

  private listAgents(): string {
    if (this.agents.size === 0) {
      return 'No agents currently active';
    }

    const agentList = Array.from(this.agents.entries())
      .map(([id, info]) => `  - ${id}: ${info.status}`)
      .join('\n');

    return `Active agents:\n${agentList}`;
  }
}

// Main CLI interface
async function main() {
  const agent = new MasterAgent();
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: '\n> '
  });

  console.log('🚀 Master Agent initialized. Type /help for commands.');
  rl.prompt();

  rl.on('line', async (input) => {
    const trimmed = input.trim();

    if (trimmed.toLowerCase() === 'exit' || trimmed.toLowerCase() === 'quit') {
      rl.close();
      return;
    }

    const response = await agent.handleUserInput(trimmed);
    console.log(`\n[Master Agent]: ${response}`);
    rl.prompt();
  });

  rl.on('close', () => {
    console.log('\nMaster Agent shutting down...');
    process.exit(0);
  });
}

if (require.main === module) {
  main().catch(console.error);
}

export { MasterAgent };
```

### Step 2: Zellij Integration Module
```typescript
// src/zellijManager.ts - Handles zellij session/pane management

import { exec, spawn } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

interface PaneInfo {
  id: string;
  name: string;
  isActive: boolean;
}

class ZellijManager {
  private sessionName: string;
  private panes: Map<string, PaneInfo>;

  constructor(sessionName: string = 'multi-agent-system') {
    this.sessionName = sessionName;
    this.panes = new Map();
  }

  async createSession(): Promise<boolean> {
    try {
      await execAsync(`zellij attach ${this.sessionName} --create`);
      return true;
    } catch (error) {
      console.error('Failed to create zellij session:', error);
      return false;
    }
  }

  async createPane(paneName: string, direction: 'right' | 'down' = 'right'): Promise<string> {
    try {
      await execAsync(`zellij action new-pane --direction ${direction}`);

      // Store pane info
      const paneId = await this.getLatestPaneId();
      this.panes.set(paneName, {
        id: paneId,
        name: paneName,
        isActive: true
      });

      return paneId;
    } catch (error) {
      throw new Error(`Failed to create pane: ${error}`);
    }
  }

  async sendToPane(paneName: string, command: string): Promise<void> {
    const pane = this.panes.get(paneName);
    if (!pane) {
      throw new Error(`Pane ${paneName} not found`);
    }

    // Send command to specific pane
    await execAsync(`zellij action write-chars "${command}"`);
    await execAsync(`zellij action write-chars $'\\n'`);
  }

  private async getLatestPaneId(): Promise<string> {
    // This would query zellij for pane info
    // For now, return a placeholder
    return `pane-${Date.now()}`;
  }

  async listPanes(): Promise<PaneInfo[]> {
    return Array.from(this.panes.values());
  }
}

export { ZellijManager, PaneInfo };
```

### Step 3: Message Bus for Agent Communication
```typescript
// src/messageBus.ts - Inter-agent communication

import { EventEmitter } from 'events';
import { v4 as uuidv4 } from 'uuid';

interface Message {
  fromAgent: string;
  toAgent: string | 'broadcast';
  msgType: 'task' | 'status' | 'query' | 'response' | 'error';
  payload: any;
  priority: number;
  timestamp: string;
  correlationId: string;
}

type MessageHandler = (message: Message) => void | Promise<void>;

class MessageBus extends EventEmitter {
  private subscribers: Map<string, Set<MessageHandler>>;
  private messageQueue: Message[];
  private isProcessing: boolean;

  constructor() {
    super();
    this.subscribers = new Map();
    this.messageQueue = [];
    this.isProcessing = false;
  }

  createMessage(
    fromAgent: string,
    toAgent: string | 'broadcast',
    msgType: Message['msgType'],
    payload: any,
    priority: number = 3
  ): Message {
    return {
      fromAgent,
      toAgent,
      msgType,
      payload,
      priority,
      timestamp: new Date().toISOString(),
      correlationId: uuidv4()
    };
  }

  async publish(message: Message): Promise<void> {
    this.messageQueue.push(message);
    this.messageQueue.sort((a, b) => b.priority - a.priority);

    if (!this.isProcessing) {
      await this.processMessages();
    }
  }

  subscribe(agentId: string, handler: MessageHandler): void {
    if (!this.subscribers.has(agentId)) {
      this.subscribers.set(agentId, new Set());
    }
    this.subscribers.get(agentId)!.add(handler);
  }

  unsubscribe(agentId: string, handler?: MessageHandler): void {
    if (!handler) {
      this.subscribers.delete(agentId);
    } else {
      this.subscribers.get(agentId)?.delete(handler);
    }
  }

  private async processMessages(): Promise<void> {
    this.isProcessing = true;

    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;

      // Direct message
      if (message.toAgent !== 'broadcast') {
        const handlers = this.subscribers.get(message.toAgent);
        if (handlers) {
          for (const handler of handlers) {
            try {
              await handler(message);
            } catch (error) {
              console.error(`Handler error for ${message.toAgent}:`, error);
            }
          }
        }
      }
      // Broadcast message
      else {
        for (const [agentId, handlers] of this.subscribers) {
          if (agentId !== message.fromAgent) {
            for (const handler of handlers) {
              try {
                await handler(message);
              } catch (error) {
                console.error(`Broadcast handler error for ${agentId}:`, error);
              }
            }
          }
        }
      }
    }

    this.isProcessing = false;
  }
}

export { MessageBus, Message };
```

### Step 4: Enhanced Master Agent with Integration
```typescript
// src/enhancedMasterAgent.ts - Full implementation with zellij and messaging

import { MasterAgent } from './masterAgent';
import { ZellijManager } from './zellijManager';
import { MessageBus, Message } from './messageBus';
import * as readline from 'readline';

interface Agent {
  id: string;
  type: string;
  status: 'active' | 'inactive' | 'error';
  pane?: string;
  spawnedAt: Date;
}

class EnhancedMasterAgent extends MasterAgent {
  private zellij: ZellijManager;
  private messageBus: MessageBus;
  private activeAgents: Map<string, Agent>;

  constructor() {
    super();
    this.zellij = new ZellijManager();
    this.messageBus = new MessageBus();
    this.activeAgents = new Map();

    // Subscribe to messages
    this.messageBus.subscribe('master-001', this.handleMessage.bind(this));
  }

  async initialize(): Promise<void> {
    console.log('🚀 Initializing Master Agent environment...');

    // Create zellij session
    const sessionCreated = await this.zellij.createSession();
    if (sessionCreated) {
      console.log('✅ Zellij session created');

      // Create monitoring panes in master tab
      await this.zellij.createPane('logs', 'down');
      await this.zellij.createPane('metrics', 'right');
      console.log('✅ Monitoring panes created');
    }

    console.log('✅ Message bus started');
    console.log('✅ Master Agent ready!');
  }

  private async handleMessage(message: Message): Promise<void> {
    console.log(`\n[${message.fromAgent}]: ${JSON.stringify(message.payload)}`);

    switch (message.msgType) {
      case 'status':
        await this.handleStatusUpdate(message);
        break;
      case 'query':
        await this.handleQuery(message);
        break;
      case 'error':
        await this.handleError(message);
        break;
    }
  }

  private async handleStatusUpdate(message: Message): Promise<void> {
    const agent = this.activeAgents.get(message.fromAgent);
    if (agent) {
      agent.status = message.payload.status || agent.status;
    }
  }

  private async handleQuery(message: Message): Promise<void> {
    // Respond to agent queries
    const response = this.messageBus.createMessage(
      'master-001',
      message.fromAgent,
      'response',
      { answer: 'Processing your query...' }
    );
    await this.messageBus.publish(response);
  }

  private async handleError(message: Message): Promise<void> {
    console.error(`Error from ${message.fromAgent}:`, message.payload);
    // Handle agent errors, possibly restart or reassign tasks
  }

  async spawnFrontendAgent(): Promise<string> {
    const agentId = 'frontend-001';

    // Create new tab for frontend
    await this.zellij.createPane('frontend', 'right');

    // Launch frontend agent process
    const cmd = 'npm run agent:frontend';
    await this.zellij.sendToPane('frontend', cmd);

    // Register agent
    this.activeAgents.set(agentId, {
      id: agentId,
      type: 'frontend',
      status: 'active',
      pane: 'frontend',
      spawnedAt: new Date()
    });

    // Announce to message bus
    const announcement = this.messageBus.createMessage(
      'master-001',
      'broadcast',
      'status',
      { event: 'agent_spawned', agentId, type: 'frontend' }
    );
    await this.messageBus.publish(announcement);

    return `Frontend agent ${agentId} spawned successfully`;
  }
}

export { EnhancedMasterAgent };
```

## Immediate Next Steps to Get Running

### 1. Initialize Node.js Project
```bash
cd agentic-cc-boilerplate
npm init -y
npm install --save-dev typescript @types/node ts-node nodemon
npm install readline uuid
```

### 2. Create TypeScript Configuration
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 3. Update package.json Scripts
```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/masterAgent.js",
    "dev": "ts-node src/masterAgent.ts",
    "watch": "nodemon --exec ts-node src/masterAgent.ts",
    "agent:master": "ts-node src/masterAgent.ts",
    "agent:frontend": "ts-node src/frontendAgent.ts"
  }
}
```

### 4. Run the Master Agent
```bash
# Development mode with hot reload
npm run watch

# Or direct execution
npm run dev
```

## Quick Testing Commands

Once you have the master agent running:

```bash
# Test basic interaction
> Hello master agent
[Master Agent]: Understood: 'Hello master agent'. Breaking down into subtasks...

# Test commands
> /status
[Master Agent]: Master Agent Status: idle

> /spawn frontend
[Master Agent]: Would spawn frontend agent (zellij integration pending)

> /list
[Master Agent]: Active agents:
  - frontend-001: active

# Test task assignment
> Create a React component for user login
[Master Agent]: Understood: 'Create a React component for user login'. Breaking down into subtasks...
```

## Project Structure
```
agentic-cc-boilerplate/
├── src/
│   ├── masterAgent.ts         # Basic master agent
│   ├── zellijManager.ts       # Zellij integration
│   ├── messageBus.ts          # Inter-agent communication
│   ├── enhancedMasterAgent.ts # Full implementation
│   └── agents/
│       └── frontendAgent.ts   # Frontend specialist (future)
├── dist/                       # Compiled JavaScript
├── docs/                       # Documentation
├── tsconfig.json              # TypeScript config
├── package.json               # Dependencies
└── README.md
```

## Key Implementation Decisions

1. **Language**: TypeScript for type safety and better IDE support
2. **Runtime**: Node.js for async operations and system access
3. **Communication**: EventEmitter initially, WebSockets for distributed
4. **State**: In-memory Map structures, Redis/SQLite later
5. **CLI**: readline module initially, Ink/Blessed.js for rich UI later

## Iterative Enhancement Path

**Week 1: Basic Interaction**
- Run basic masterAgent.ts ✅
- Add command parsing ✅
- Create help system ✅

**Week 2: Zellij Integration**
- Add zellijManager.ts
- Test pane creation
- Implement pane communication

**Week 3: Message Bus**
- Add messageBus.ts
- Implement agent registration
- Test inter-agent messaging

**Week 4: First Specialized Agent**
- Create frontendAgent.ts
- Implement task delegation
- Test full workflow

## Common Issues & Solutions

### Issue: Zellij commands not working
**Solution**: Ensure zellij is installed and in PATH. For alternative terminal multiplexers:
- tmux: Modify zellijManager to use tmux commands
- screen: Use screen command syntax
- No multiplexer: Use child_process.spawn for separate processes

### Issue: TypeScript compilation errors
**Solution**: Ensure all dependencies are installed:
```bash
npm install --save-dev @types/node @types/uuid
```

## Success Criteria for First Version

✅ Can spawn master agent
✅ Can accept user input
✅ Can respond to basic commands
✅ Can outline task decomposition
✅ Shows status and help
✅ TypeScript type safety

## Ready to Code?

Start with creating `src/masterAgent.ts` and run:
```bash
npm run dev
```

You'll have a working TypeScript master agent in under 5 minutes!
