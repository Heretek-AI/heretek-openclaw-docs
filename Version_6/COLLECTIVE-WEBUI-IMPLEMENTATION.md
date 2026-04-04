# Heretek-AI Collective WebUI — Implementation Plan

**Date:** 2026-04-04  
**Version:** 1.0.0  
**Stack:** Next.js 14, Tailwind CSS, React Flow, Zustand, WebSocket

---

## 1. Project Overview

A real-time monitoring and control interface for the Heretek-AI Collective, providing:
- **Neural Map** — Live visualization of agent topology and A2A communications
- **Agent & Skill Manager** — Deploy, configure, and manage agents and skills
- **Memory Deep Search** — Query vector DB with episodic/semantic/shared filters
- **A2A Monitor** — Live stream of inter-agent "internal monologue"
- **Collective Command Line** — Global broadcast input for autonomous debate

---

## 2. Folder Structure

```
heretek-collective-ui/
├── package.json
├── next.config.js
├── tailwind.config.js
├── postcss.config.js
├── tsconfig.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .env.local
│
├── public/
│   ├── favicon.ico
│   ├── fonts/
│   │   └── jetbrains-mono/          # Monospace font for terminal
│   └── images/
│       └── heretek-logo.svg
│
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── layout.tsx                # Root layout with theme provider
│   │   ├── page.tsx                  # Dashboard home (Neural Map)
│   │   ├── agents/
│   │   │   └── page.tsx              # Agent & Skill Manager
│   │   ├── memory/
│   │   │   └── page.tsx              # Memory Deep Search
│   │   ├── a2a/
│   │   │   └── page.tsx              # A2A Monitor (Internal Monologue)
│   │   ├── config/
│   │   │   └── page.tsx              # Gateway & LiteLLM config
│   │   └── api/                      # API Routes (BFF layer)
│   │       ├── agents/
│   │       │   ├── route.ts          # GET /api/agents
│   │       │   └── [id]/
│   │       │       ├── route.ts      # GET/PUT /api/agents/:id
│   │       │       └── deploy/
│   │       │           └── route.ts  # POST /api/agents/:id/deploy
│   │       ├── skills/
│   │       │   ├── route.ts          # GET/POST /api/skills
│   │       │   └── [id]/
│   │       │       └── route.ts      # DELETE /api/skills/:id
│   │       ├── memory/
│   │       │   ├── route.ts          # POST /api/memory/search
│   │       │   └── [id]/
│   │       │       ├── route.ts      # GET/DELETE /api/memory/:id
│   │       │       └── pin/
│   │       │           └── route.ts  # POST /api/memory/:id/pin
│   │       ├── a2a/
│   │       │   ├── stream/
│   │       │   │   └── route.ts      # GET /api/a2a/stream (SSE)
│   │       │   └── broadcast/
│   │       │       └── route.ts      # POST /api/a2a/broadcast
│   │       ├── gateway/
│   │       │   ├── health/
│   │       │   │   └── route.ts      # GET /api/gateway/health
│   │       │   └── config/
│   │       │       └── route.ts      # GET/PUT /api/gateway/config
│   │       └── litellm/
│   │           ├── metrics/
│   │           │   └── route.ts      # GET /api/litellm/metrics
│   │           └── config/
│   │               └── route.ts      # GET/PUT /api/litellm/config
│   │
│   ├── components/                   # React Components
│   │   ├── ui/                       # Base UI components
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── input.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── toast.tsx
│   │   │   └── skeleton.tsx
│   │   │
│   │   ├── neural-map/               # Neural Map components
│   │   │   ├── NeuralMap.tsx         # Main React Flow canvas
│   │   │   ├── AgentNode.tsx         # Custom agent node
│   │   │   ├── A2AEdge.tsx           # Animated edge for A2A
│   │   │   ├── ThoughtStack.tsx      # Gantt chart overlay
│   │   │   ├── NodeDetails.tsx       # Side panel for node info
│   │   │   └── MapControls.tsx       # Zoom/pan/filter controls
│   │   │
│   │   ├── agent-manager/            # Agent & Skill Manager
│   │   │   ├── AgentList.tsx         # Agent registry table
│   │   │   ├── AgentCard.tsx         # Agent status card
│   │   │   ├── SkillStore.tsx        # Plugin/skill browser
│   │   │   ├── SkillCard.tsx         # Individual skill display
│   │   │   ├── DeployDialog.tsx      # One-click deploy modal
│   │   │   └── StatusToggle.tsx      # Active/Idle/Debating toggle
│   │   │
│   │   ├── memory-search/            # Memory Deep Search
│   │   │   ├── MemorySearch.tsx      # Search bar with filters
│   │   │   ├── MemoryResults.tsx     # Search results list
│   │   │   ├── MemoryCard.tsx        # Individual memory item
│   │   │   ├── MemoryEditor.tsx      # Pin/Prune editor
│   │   │   ├── FilterBar.tsx         # Episodic/Semantic/Shared
│   │   │   └── MemoryStats.tsx       # Memory usage stats
│   │   │
│   │   ├── a2a-monitor/              # A2A Monitor
│   │   │   ├── A2AStream.tsx         # Live log stream
│   │   │   ├── EnvelopeViewer.tsx    # A2A envelope JSON viewer
│   │   │   ├── TraceDetails.tsx      # TraceID details panel
│   │   │   ├── LatencyChart.tsx      # Real-time latency graph
│   │   │   └── FilterBar.tsx         # Message type filters
│   │   │
│   │   ├── command-line/             # Collective Command Line
│   │   │   ├── CommandLine.tsx       # Global broadcast input
│   │   │   ├── CommandHistory.tsx    # Previous commands
│   │   │   └── ResponsePanel.tsx     # Agent debate responses
│   │   │
│   │   └── layout/                   # Layout components
│   │       ├── Header.tsx            # Top navigation
│   │       ├── Sidebar.tsx           # Side navigation
│   │       ├── Footer.tsx            # Status bar
│   │       └── ThemeToggle.tsx       # Dark/light toggle
│   │
│   ├── lib/                          # Utilities
│   │   ├── api.ts                    # API client helpers
│   │   ├── websocket.ts              # WebSocket connection manager
│   │   ├── sse.ts                    # Server-Sent Events client
│   │   ├── formatters.ts             # Data formatting utilities
│   │   └── constants.ts              # App constants
│   │
│   ├── hooks/                        # Custom React Hooks
│   │   ├── useAgentStatus.ts         # Poll agent status
│   │   ├── useA2AStream.ts           # Subscribe to A2A stream
│   │   ├── useMemorySearch.ts        # Memory search hook
│   │   ├── useGatewayHealth.ts       # Gateway health hook
│   │   └── useNeuralMap.ts           # Neural Map state hook
│   │
│   └── store/                        # Zustand State Stores
│       ├── index.ts                  # Store exports
│       ├── agentStore.ts             # Agent state & actions
│       ├── neuralMapStore.ts         # Neural Map state
│       ├── memoryStore.ts            # Memory search state
│       ├── a2aStore.ts               # A2A message buffer
│       └── configStore.ts            # App configuration
│
├── styles/
│   ├── globals.css                   # Global styles & theme
│   └── cybernetic.css                # Cybernetic theme variables
│
└── docs/
    ├── ARCHITECTURE.md               # Technical architecture
    ├── API.md                        # API documentation
    ├── DEPLOYMENT.md                 # Deployment guide
    └── THEMING.md                    # Theme customization
```

---

## 3. Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Heretek Collective UI
  collective-ui:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: heretek-collective-ui
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - GATEWAY_URL=ws://host.docker.internal:18789
      - LITELLM_URL=http://host.docker.internal:4000
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
      - DATABASE_URL=postgresql://heretek:zHNb5MMUOWEHWcv8pHTOpl+hwoLCAi1v@host.docker.internal:5432/heretek
      - REDIS_URL=redis://host.docker.internal:6379
    depends_on:
      - gateway-health
    restart: unless-stopped
    networks:
      - heretek-network

  # Gateway Health Check (wait for gateway)
  gateway-health:
    image: curlimages/curl:latest
    container_name: heretek-gateway-health
    command: >
      sh -c '
        echo "Waiting for Gateway...";
        until curl -sf http://host.docker.internal:18789/health; do
          sleep 2;
        done;
        echo "Gateway ready!"
      '
    restart: "no"
    networks:
      - heretek-network

  # Optional: Langfuse for observability
  langfuse:
    image: langfuse/langfuse:latest
    container_name: heretek-langfuse
    ports:
      - "3001:3000"
    environment:
      - DATABASE_URL=postgresql://langfuse:langfuse@langfuse-db:5432/langfuse
      - NEXTAUTH_SECRET=your-secret-key
      - NEXTAUTH_URL=http://localhost:3001
    depends_on:
      - langfuse-db
    restart: unless-stopped
    networks:
      - heretek-network

  langfuse-db:
    image: postgres:15-alpine
    container_name: heretek-langfuse-db
    environment:
      - POSTGRES_USER=langfuse
      - POSTGRES_PASSWORD=langfuse
      - POSTGRES_DB=langfuse
    volumes:
      - langfuse-db-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - heretek-network

volumes:
  langfuse-db-data:

networks:
  heretek-network:
    driver: bridge
```

---

## 4. Dockerfile

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# Dependencies
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

# Build
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# Production
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

---

## 5. Core React Components

### 5.1 Neural Map (React Flow)

```tsx
// src/components/neural-map/NeuralMap.tsx
'use client';

import React, { useCallback, useEffect } from 'react';
import ReactFlow, {
  Node,
  Edge,
  Controls,
  Background,
  useNodesState,
  useEdgesState,
  Connection,
  addEdge,
  BackgroundVariant,
} from 'reactflow';
import 'reactflow/dist/style.css';
import { useAgentStore } from '@/store/agentStore';
import { useNeuralMapStore } from '@/store/neuralMapStore';
import { useA2AStore } from '@/store/a2aStore';
import AgentNode from './AgentNode';
import A2AEdge from './A2AEdge';
import MapControls from './MapControls';
import NodeDetails from './NodeDetails';

const nodeTypes = {
  agent: AgentNode,
};

const edgeTypes = {
  a2a: A2AEdge,
};

export default function NeuralMap() {
  const { agents } = useAgentStore();
  const { a2aMessages } = useA2AStore();
  const { selectedNode, setSelectedNode } = useNeuralMapStore();

  // Convert agents to React Flow nodes
  const [nodes, setNodes, onNodesChange] = useNodesState(
    agents.map((agent, index) => ({
      id: agent.id,
      type: 'agent',
      position: {
        x: (index % 4) * 250,
        y: Math.floor(index / 4) * 200,
      },
      data: {
        label: agent.name,
        status: agent.status,
        model: agent.model,
        memoryUsage: agent.memoryUsage,
        lastActive: agent.lastActive,
      },
    }))
  );

  // Convert A2A messages to animated edges
  const [edges, setEdges, onEdgesChange] = useEdgesState(
    a2aMessages
      .filter((msg) => msg.to && msg.from)
      .map((msg, index) => ({
        id: `edge-${msg.from}-${msg.to}-${index}`,
        source: msg.from,
        target: msg.to,
        type: 'a2a',
        animated: true,
        style: { stroke: '#06b6d4', strokeWidth: 2 },
        data: {
          traceId: msg.traceId,
          messageType: msg.type,
          latency: msg.latency,
          timestamp: msg.timestamp,
        },
      }))
  );

  const onConnect = useCallback(
    (params: Connection) => setEdges((eds) => addEdge(params, eds)),
    [setEdges]
  );

  const onNodeClick = useCallback(
    (_: React.MouseEvent, node: Node) => {
      setSelectedNode(node.id);
    },
    [setSelectedNode]
  );

  return (
    <div className="w-full h-[800px] bg-gray-950 border border-cyan-900/50 rounded-lg">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onConnect={onConnect}
        onNodeClick={onNodeClick}
        fitView
        snapToGrid
        snapGrid={[15, 15]}
        className="bg-gray-950"
      >
        <Background
          variant={BackgroundVariant.Dots}
          gap={20}
          size={1}
          color="#0891b2"
        />
        <Controls className="bg-gray-900 border-cyan-800" />
        <MapControls />
      </ReactFlow>
      {selectedNode && <NodeDetails nodeId={selectedNode} onClose={() => setSelectedNode(null)} />}
    </div>
  );
}
```

### 5.2 Agent Node Component

```tsx
// src/components/neural-map/AgentNode.tsx
'use client';

import React, { memo } from 'react';
import { Handle, Position, NodeProps } from 'reactflow';
import { Badge } from '@/components/ui/badge';

const statusColors = {
  online: 'bg-green-500',
  offline: 'bg-red-500',
  debating: 'bg-yellow-500',
  idle: 'bg-gray-500',
};

function AgentNode({ data, selected }: NodeProps) {
  return (
    <div
      className={`
        px-4 py-3 rounded-lg border-2 
        ${selected ? 'border-cyan-400 shadow-lg shadow-cyan-500/50' : 'border-cyan-800'}
        bg-gray-900/90 backdrop-blur-sm
        min-w-[180px]
      `}
    >
      {/* Input Handle */}
      <Handle
        type="target"
        position={Position.Left}
        className="!bg-cyan-600 !border-cyan-400"
      />

      {/* Agent Name */}
      <div className="flex items-center justify-between mb-2">
        <span className="font-mono text-cyan-400 font-bold">{data.label}</span>
        <Badge
          className={`
            ${statusColors[data.status as keyof typeof statusColors] || statusColors.idle}
            text-white text-xs px-2 py-0.5 rounded-full
          `}
        >
          {data.status}
        </Badge>
      </div>

      {/* Model */}
      <div className="text-xs text-gray-400 font-mono mb-1">
        {data.model}
      </div>

      {/* Memory Usage */}
      <div className="text-xs text-gray-400 font-mono">
        Memory: {data.memoryUsage || 'N/A'}
      </div>

      {/* Output Handle */}
      <Handle
        type="source"
        position={Position.Right}
        className="!bg-cyan-600 !border-cyan-400"
      />
    </div>
  );
}

export default memo(AgentNode);
```

### 5.3 A2A Animated Edge

```tsx
// src/components/neural-map/A2AEdge.tsx
'use client';

import React, { memo } from 'react';
import { EdgeProps, getBezierPath } from 'reactflow';

function A2AEdge({
  id,
  sourceX,
  sourceY,
  targetX,
  targetY,
  sourcePosition,
  targetPosition,
  style = {},
  markerEnd,
  data,
}: EdgeProps) {
  const [edgePath] = getBezierPath({
    sourceX,
    sourceY,
    sourcePosition,
    targetX,
    targetY,
    targetPosition,
  });

  return (
    <>
      {/* Base path */}
      <path
        id={id}
        style={{
          ...style,
          stroke: '#06b6d4',
          strokeWidth: 2,
          fill: 'none',
        }}
        d={edgePath}
        markerEnd={markerEnd}
      />
      
      {/* Animated pulse */}
      <circle r="4" fill="#22d3ee">
        <animateMotion
          dur="1.5s"
          repeatCount="indefinite"
          path={edgePath}
        />
      </circle>

      {/* Latency tooltip on hover */}
      <title>
        {data?.messageType || 'A2A Message'}
        {'\n'}Trace: {data?.traceId || 'N/A'}
        {'\n'}Latency: {data?.latency || 'N/A'}ms
      </title>
    </>
  );
}

export default memo(A2AEdge);
```

### 5.4 Memory Deep Search

```tsx
// src/components/memory-search/MemorySearch.tsx
'use client';

import React, { useState, useCallback } from 'react';
import { useMemoryStore } from '@/store/memoryStore';
import MemoryResults from './MemoryResults';
import FilterBar from './FilterBar';
import MemoryStats from './MemoryStats';

type MemoryType = 'episodic' | 'semantic' | 'shared';

export default function MemorySearch() {
  const [query, setQuery] = useState('');
  const [memoryType, setMemoryType] = useState<MemoryType>('semantic');
  const { searchMemory, isLoading, results } = useMemoryStore();

  const handleSearch = useCallback(async () => {
    if (!query.trim()) return;
    await searchMemory(query, memoryType);
  }, [query, memoryType, searchMemory]);

  const handleKeyPress = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === 'Enter') {
        handleSearch();
      }
    },
    [handleSearch]
  );

  return (
    <div className="space-y-6">
      {/* Search Header */}
      <div className="bg-gray-900/50 border border-cyan-900/50 rounded-lg p-6">
        <h2 className="text-2xl font-mono text-cyan-400 mb-4">
          🧠 Memory Deep Search
        </h2>
        
        {/* Search Input */}
        <div className="flex gap-4 mb-4">
          <input
            type="text"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            onKeyPress={handleKeyPress}
            placeholder="Search collective memory..."
            className="flex-1 bg-gray-950 border border-cyan-800 rounded-lg px-4 py-3 
                       text-cyan-100 font-mono placeholder-gray-500
                       focus:outline-none focus:border-cyan-500 focus:ring-1 focus:ring-cyan-500"
          />
          <button
            onClick={handleSearch}
            disabled={isLoading || !query.trim()}
            className="px-6 py-3 bg-cyan-600 hover:bg-cyan-500 disabled:bg-gray-700
                       text-white font-mono rounded-lg transition-colors
                       disabled:cursor-not-allowed"
          >
            {isLoading ? 'Searching...' : 'Search'}
          </button>
        </div>

        {/* Filters */}
        <FilterBar
          currentType={memoryType}
          onChange={setMemoryType}
        />
      </div>

      {/* Memory Stats */}
      <MemoryStats />

      {/* Results */}
      {results.length > 0 && <MemoryResults results={results} />}
    </div>
  );
}
```

### 5.5 A2A Stream Monitor

```tsx
// src/components/a2a-monitor/A2AStream.tsx
'use client';

import React, { useEffect, useRef } from 'react';
import { useA2AStore } from '@/store/a2aStore';
import EnvelopeViewer from './EnvelopeViewer';

export default function A2AStream() {
  const { a2aMessages, subscribeToStream, unsubscribeFromStream } = useA2AStore();
  const streamRef = useRef<EventSource | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // Connect to SSE stream
    streamRef.current = new EventSource('/api/a2a/stream');
    subscribeToStream(streamRef.current);

    return () => {
      unsubscribeFromStream();
      streamRef.current?.close();
    };
  }, [subscribeToStream, unsubscribeFromStream]);

  // Auto-scroll to bottom
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [a2aMessages]);

  return (
    <div className="bg-gray-950 border border-cyan-900/50 rounded-lg p-4 h-[600px] overflow-hidden flex flex-col">
      <h3 className="text-lg font-mono text-cyan-400 mb-4 flex items-center gap-2">
        <span className="w-2 h-2 bg-green-500 rounded-full animate-pulse" />
        Internal Monologue — Live
      </h3>

      {/* Message Stream */}
      <div className="flex-1 overflow-y-auto space-y-2 font-mono text-sm">
        {a2aMessages.map((msg, index) => (
          <EnvelopeViewer key={`${msg.traceId}-${index}`} envelope={msg} />
        ))}
        <div ref={messagesEndRef} />
      </div>
    </div>
  );
}
```

### 5.6 Collective Command Line

```tsx
// src/components/command-line/CommandLine.tsx
'use client';

import React, { useState, useCallback } from 'react';
import { useA2AStore } from '@/store/a2aStore';

export default function CommandLine() {
  const [input, setInput] = useState('');
  const [isBroadcasting, setIsBroadcasting] = useState(false);
  const { broadcastToCollective } = useA2AStore();

  const handleBroadcast = useCallback(async () => {
    if (!input.trim()) return;
    setIsBroadcasting(true);
    try {
      await broadcastToCollective(input);
      setInput('');
    } finally {
      setIsBroadcasting(false);
    }
  }, [input, broadcastToCollective]);

  const handleKeyPress = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        handleBroadcast();
      }
    },
    [handleBroadcast]
  );

  return (
    <div className="bg-gray-900/50 border border-cyan-900/50 rounded-lg p-6">
      <h3 className="text-xl font-mono text-cyan-400 mb-4">
        📢 Collective Broadcast
      </h3>
      <p className="text-gray-400 text-sm mb-4">
        Send a message to the Global Workspace. The Triad will deliberate and respond autonomously.
      </p>

      {/* Input */}
      <textarea
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyPress={handleKeyPress}
        placeholder="Type your broadcast message... (Enter to send)"
        rows={4}
        className="w-full bg-gray-950 border border-cyan-800 rounded-lg px-4 py-3
                   text-cyan-100 font-mono placeholder-gray-500
                   focus:outline-none focus:border-cyan-500 focus:ring-1 focus:ring-cyan-500
                   resize-none"
      />

      {/* Send Button */}
      <div className="mt-4 flex justify-end">
        <button
          onClick={handleBroadcast}
          disabled={isBroadcasting || !input.trim()}
          className="px-6 py-3 bg-cyan-600 hover:bg-cyan-500 disabled:bg-gray-700
                     text-white font-mono rounded-lg transition-colors
                     disabled:cursor-not-allowed
                     flex items-center gap-2"
        >
          {isBroadcasting ? (
            <>
              <span className="w-4 h-4 border-2 border-white/30 border-t-white rounded-full animate-spin" />
              Broadcasting...
            </>
          ) : (
            <>
              🦞 Broadcast to Collective
            </>
          )}
        </button>
      </div>
    </div>
  );
}
```

---

## 6. Zustand Store Example

```ts
// src/store/agentStore.ts
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

export interface Agent {
  id: string;
  name: string;
  status: 'online' | 'offline' | 'debating' | 'idle';
  model: string;
  memoryUsage?: string;
  lastActive?: string;
  websocketReadyState?: number;
  timeSinceLastSeenMs?: number;
}

interface AgentState {
  agents: Agent[];
  isLoading: boolean;
  error: string | null;
  lastUpdated: Date | null;
  
  // Actions
  fetchAgents: () => Promise<void>;
  setAgents: (agents: Agent[]) => void;
  updateAgent: (id: string, updates: Partial<Agent>) => void;
  deployAgent: (id: string) => Promise<void>;
}

export const useAgentStore = create<AgentState>()(
  subscribeWithSelector((set, get) => ({
    agents: [],
    isLoading: false,
    error: null,
    lastUpdated: null,

    fetchAgents: async () => {
      set({ isLoading: true, error: null });
      try {
        const res = await fetch('/api/agents');
        if (!res.ok) throw new Error('Failed to fetch agents');
        const data = await res.json();
        set({ agents: data.agents, isLoading: false, lastUpdated: new Date() });
      } catch (err) {
        set({ 
          error: err instanceof Error ? err.message : 'Unknown error', 
          isLoading: false 
        });
      }
    },

    setAgents: (agents) => set({ agents, lastUpdated: new Date() }),

    updateAgent: (id, updates) => {
      set((state) => ({
        agents: state.agents.map((a) => (a.id === id ? { ...a, ...updates } : a)),
        lastUpdated: new Date(),
      }));
    },

    deployAgent: async (id) => {
      const res = await fetch(`/api/agents/${id}/deploy`, { method: 'POST' });
      if (!res.ok) throw new Error('Deploy failed');
    },
  }))
);
```

---

## 7. Tailwind Cybernetic Theme

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        // Cybernetic color palette
        cyan: {
          50: '#f0fdfa',
          100: '#ecfeff',
          200: '#cffafe',
          300: '#a5f3fc',
          400: '#22d3ee',
          500: '#06b6d4',
          600: '#0891b2',
          700: '#0e7490',
          800: '#155e75',
          900: '#164e63',
          950: '#083344',
        },
        gray: {
          950: '#030712',  // Deep space black
        },
      },
      fontFamily: {
        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
      },
      animation: {
        'pulse-fast': 'pulse 1s cubic-bezier(0.4, 0, 0.6, 1) infinite',
        'glow': 'glow 2s ease-in-out infinite alternate',
      },
      keyframes: {
        glow: {
          '0%': { boxShadow: '0 0 5px #06b6d4, 0 0 10px #06b6d4' },
          '100%': { boxShadow: '0 0 10px #22d3ee, 0 0 20px #22d3ee' },
        },
      },
    },
  },
  plugins: [],
};
```

---

## 8. API Endpoints Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/agents` | GET | List all agents with status |
| `/api/agents/:id` | GET/PUT | Get/update specific agent |
| `/api/agents/:id/deploy` | POST | Deploy agent instance |
| `/api/skills` | GET/POST | List/install skills |
| `/api/skills/:id` | DELETE | Remove skill |
| `/api/memory/search` | POST | Search vector DB |
| `/api/memory/:id` | GET/DELETE | Get/delete memory |
| `/api/memory/:id/pin` | POST | Pin memory (prevent pruning) |
| `/api/a2a/stream` | GET | SSE stream of A2A messages |
| `/api/a2a/broadcast` | POST | Send to Global Workspace |
| `/api/gateway/health` | GET | Gateway health check |
| `/api/gateway/config` | GET/PUT | Gateway configuration |
| `/api/litellm/metrics` | GET | LiteLLM metrics |
| `/api/litellm/config` | GET/PUT | LiteLLM configuration |

---

## 9. Next Steps

1. **Initialize Project:**
   ```bash
   npx create-next-app@latest heretek-collective-ui --typescript --tailwind --app
   cd heretek-collective-ui
   npm install reactflow zustand
   ```

2. **Copy Folder Structure** from above

3. **Configure Environment Variables** in `.env.local`

4. **Run Development Server:**
   ```bash
   npm run dev
   ```

5. **Deploy with Docker:**
   ```bash
   docker-compose up -d
   ```

---

**Document Status:** Complete  
**Ready for Implementation:** ✅

🦞 *The lobster way — Any OS. Any Platform. Together. The thought that never ends.*
