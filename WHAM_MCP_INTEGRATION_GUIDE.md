# WHAM Runtime Integration with Existing MCP RAG System

## Overview

This document shows how to integrate your **existing CDYP7 MoA Graph Planner + MCP RAG system** with the new **WHAM 3D Virtual Campus runtime**.

The integration creates a unified system where:
- Your existing `mcp_rag_embeddings.jsonl` powers spatial knowledge
- The MCP RAG server (`localhost:3333`) provides vector search
- Graph planner (`graph_planner_moa.yaml`) orchestrates multi-agent workflows
- 3D virtual campus visualizes agent interactions in real-time
- A2A registry and reward profiles drive agent behavior

---

## Architecture: Existing System → WHAM Integration

```
┌─────────────────────────────────────────────────────────────┐
│              EXISTING MCP RAG SYSTEM                         │
│  ┌────────────────────────────────────────────────────┐    │
│  │  mcp_rag_embeddings.jsonl                          │    │
│  │  • Pre-computed embeddings                         │    │
│  │  • Role-based knowledge                            │    │
│  │  • Domain classifications (PROCESS/REQ/SYS/etc)    │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────┐    │
│  │  MCP RAG Server (port 3333)                        │    │
│  │  • FastAPI endpoint: POST /rag/query               │    │
│  │  • TF-IDF similarity search                        │    │
│  │  • Returns top-k results with scores               │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────┐    │
│  │  Graph Planner MoA                                 │    │
│  │  planner/graph_planner_moa.yaml                    │    │
│  │  • Sub-agent routing                               │    │
│  │  • Multi-step planning                             │    │
│  │  • Domain → Role mapping                           │    │
│  └────────────────────────────────────────────────────┘    │
└───────────────────┬──────────────────────────────────────────┘
                    │
                    │ Integration Layer
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              NEW WHAM 3D RUNTIME                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │  WHAM Virtual Campus (Three.js)                    │    │
│  │  • 3D agent avatars                                │    │
│  │  • Spatial knowledge visualization                 │    │
│  │  • Real-time handshake arcs                        │    │
│  │  • Document upload interface                       │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                          │
│  ┌────────────────▼───────────────────────────────────┐    │
│  │  WHAM Runtime Server (port 3000)                   │    │
│  │  • Proxies to MCP RAG server (3333)                │    │
│  │  • Loads graph planner YAML                        │    │
│  │  • Reads A2A registry & rewards                    │    │
│  │  • Coordinates agent behaviors                     │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Integration Points

### 1. MCP RAG Server Connection

**Your existing server:**
```python
# mcp_rag_server/server.py
@app.post("/rag/query")
def rag_query(q: Query):
    # Returns: {"results": [{"id", "text", "score"}, ...]}
```

**WHAM integration:**
```javascript
// server/wham-mcp-bridge.js
class MCPRAGBridge {
  constructor(mcpEndpoint = 'http://localhost:3333') {
    this.endpoint = mcpEndpoint;
  }

  async query(queryText, topK = 5) {
    const response = await fetch(`${this.endpoint}/rag/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        query: queryText,
        top_k: topK
      })
    });

    const data = await response.json();
    
    // Enhance results with spatial positions
    return this.enhanceWithSpatialData(data.results);
  }

  enhanceWithSpatialData(ragResults) {
    // Map RAG results to 3D positions based on domain
    const domainPositions = {
      'ZF-PROCESS': { x: -10, z: -10 },
      'ZF-REQ': { x: -10, z: 0 },
      'ZF-SYS': { x: 0, z: 0 },
      'ZF-TEST': { x: 10, z: 0 },
      'ZF-SW': { x: 10, z: 10 },
      'ZF-TOOLS': { x: 0, z: 10 }
    };

    return ragResults.map((result, i) => {
      // Extract domain from result.id or text
      const domain = this.inferDomain(result);
      const basePos = domainPositions[domain] || { x: 0, z: 0 };

      return {
        ...result,
        position: {
          x: basePos.x + (Math.random() - 0.5) * 3,
          y: 0.5 + i * 0.2, // Stack vertically
          z: basePos.z + (Math.random() - 0.5) * 3
        },
        domain
      };
    });
  }

  inferDomain(result) {
    const text = (result.text || result.id || '').toLowerCase();
    
    if (text.includes('process') || text.includes('workflow')) return 'ZF-PROCESS';
    if (text.includes('requirement') || text.includes('spec')) return 'ZF-REQ';
    if (text.includes('system') || text.includes('architecture')) return 'ZF-SYS';
    if (text.includes('test') || text.includes('validation')) return 'ZF-TEST';
    if (text.includes('software') || text.includes('code')) return 'ZF-SW';
    if (text.includes('tool') || text.includes('utility')) return 'ZF-TOOLS';
    
    return 'ZF-SYS'; // Default
  }
}

module.exports = { MCPRAGBridge };
```

### 2. Graph Planner Integration

**Load your existing planner:**
```javascript
// server/graph-planner-loader.js
const yaml = require('js-yaml');
const fs = require('fs');

class GraphPlannerLoader {
  constructor(plannerPath = './planner/graph_planner_moa.yaml') {
    this.planner = this.loadPlanner(plannerPath);
    this.registry = this.loadRegistry('./registry/registry.yaml');
    this.rewards = this.loadRewards('./registry/reward_profiles.yaml');
  }

  loadPlanner(path) {
    const content = fs.readFileSync(path, 'utf8');
    return yaml.load(content);
  }

  loadRegistry(path) {
    const content = fs.readFileSync(path, 'utf8');
    return yaml.load(content);
  }

  loadRewards(path) {
    const content = fs.readFileSync(path, 'utf8');
    return yaml.load(content);
  }

  async planTask(product, domains) {
    // Execute graph planner logic
    // Returns: multi-step plan with agent assignments
    
    const plan = {
      product,
      domains,
      steps: [],
      agents: []
    };

    // Parse planner YAML to create execution plan
    // This would match your existing graph_planner_moa.yaml structure
    
    for (const domain of domains) {
      const step = await this.createStepForDomain(domain, product);
      plan.steps.push(step);
      
      if (!plan.agents.includes(step.agent)) {
        plan.agents.push(step.agent);
      }
    }

    return plan;
  }

  async createStepForDomain(domain, product) {
    // Map domain to agent based on your existing logic
    const agentMapping = {
      'ZF-PROCESS': 'Braking_System_Manager',
      'ZF-REQ': 'Braking_System_Manager',
      'ZF-SYS': 'Central_System_Manager',
      'ZF-TEST': 'Steering_System_Manager',
      'ZF-SW': 'Central_Tools_Manager',
      'ZF-TOOLS': 'Central_Tools_Manager'
    };

    const agent = agentMapping[domain] || 'Central_System_Manager';

    return {
      domain,
      agent,
      action: `Process ${domain} for ${product}`,
      dependencies: this.getDependencies(domain),
      reward_profile: this.getRewardProfile(agent)
    };
  }

  getDependencies(domain) {
    // Based on your graph planner structure
    const deps = {
      'ZF-PROCESS': [],
      'ZF-REQ': ['ZF-PROCESS'],
      'ZF-SYS': ['ZF-REQ'],
      'ZF-TEST': ['ZF-SYS'],
      'ZF-SW': ['ZF-SYS'],
      'ZF-TOOLS': []
    };
    return deps[domain] || [];
  }

  getRewardProfile(agent) {
    return this.rewards?.[agent] || { base_reward: 1.0 };
  }
}

module.exports = { GraphPlannerLoader };
```

### 3. Agent Runtime Integration

**Connect to your existing runtime:**
```javascript
// server/agent-runtime-adapter.js
const { spawn } = require('child_process');

class AgentRuntimeAdapter {
  constructor(runtimePath = './runtime/agent_runtime_loader.py') {
    this.runtimePath = runtimePath;
    this.activeAgents = new Map();
  }

  async loadAgent(agentConfig) {
    // Spawn Python runtime for agent
    const process = spawn('python', [
      this.runtimePath,
      '--agent', agentConfig.name,
      '--config', JSON.stringify(agentConfig)
    ]);

    const agentId = agentConfig.id;

    this.activeAgents.set(agentId, {
      process,
      config: agentConfig,
      status: 'initializing'
    });

    // Listen for agent ready signal
    process.stdout.on('data', (data) => {
      const message = data.toString();
      if (message.includes('AGENT_READY')) {
        const agent = this.activeAgents.get(agentId);
        agent.status = 'ready';
        this.emit('agent_ready', agentId);
      }
    });

    return agentId;
  }

  async executeTask(agentId, task) {
    const agent = this.activeAgents.get(agentId);
    if (!agent || agent.status !== 'ready') {
      throw new Error(`Agent ${agentId} not ready`);
    }

    // Send task to agent runtime
    agent.process.stdin.write(JSON.stringify({
      type: 'task',
      task
    }) + '\n');

    // Wait for response
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error('Agent task timeout'));
      }, 30000);

      agent.process.stdout.once('data', (data) => {
        clearTimeout(timeout);
        const response = JSON.parse(data.toString());
        resolve(response);
      });
    });
  }

  getAgentStatus(agentId) {
    const agent = this.activeAgents.get(agentId);
    return agent ? agent.status : 'not_loaded';
  }
}

module.exports = { AgentRuntimeAdapter };
```

### 4. Complete Integration Server

**Unified WHAM + MCP server:**
```javascript
// server/integrated-server.js
const express = require('express');
const cors = require('cors');
const { MCPRAGBridge } = require('./wham-mcp-bridge');
const { GraphPlannerLoader } = require('./graph-planner-loader');
const { AgentRuntimeAdapter } = require('./agent-runtime-adapter');

const app = express();
app.use(cors());
app.use(express.json());
app.use(express.static('public'));

// Initialize components
const mcpBridge = new MCPRAGBridge('http://localhost:3333');
const planner = new GraphPlannerLoader();
const runtime = new AgentRuntimeAdapter();

// ============================================================================
// INTEGRATED API ENDPOINTS
// ============================================================================

// Query RAG with spatial visualization
app.post('/api/rag/query', async (req, res) => {
  try {
    const { query, top_k = 5 } = req.body;

    // Query existing MCP RAG server
    const results = await mcpBridge.query(query, top_k);

    res.json({
      success: true,
      query,
      results, // Enhanced with spatial positions
      endpoint: 'mcp://localhost:3333'
    });
  } catch (error) {
    console.error('RAG query error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Create execution plan using graph planner
app.post('/api/plan', async (req, res) => {
  try {
    const { product, domains } = req.body;

    // Use your existing graph planner
    const plan = await planner.planTask(product, domains);

    res.json({
      success: true,
      plan,
      planner_version: planner.planner?.version || '1.0'
    });
  } catch (error) {
    console.error('Planning error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Execute plan with agents
app.post('/api/execute', async (req, res) => {
  try {
    const { plan } = req.body;

    const results = [];

    // Execute each step with assigned agent
    for (const step of plan.steps) {
      // Load agent if not already loaded
      const agentConfig = planner.registry?.agents?.[step.agent];
      if (agentConfig) {
        const agentId = await runtime.loadAgent(agentConfig);
        
        // Execute task
        const result = await runtime.executeTask(agentId, step);
        results.push({
          step: step.action,
          agent: step.agent,
          result
        });
      }
    }

    res.json({
      success: true,
      execution_results: results
    });
  } catch (error) {
    console.error('Execution error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Get agent registry
app.get('/api/registry', async (req, res) => {
  res.json({
    success: true,
    registry: planner.registry,
    rewards: planner.rewards
  });
});

// Get spatial map for 3D visualization
app.get('/api/spatial/map', async (req, res) => {
  try {
    // Query all embeddings for spatial map
    const allDomains = ['ZF-PROCESS', 'ZF-REQ', 'ZF-SYS', 'ZF-TEST', 'ZF-SW', 'ZF-TOOLS'];
    
    const map = {
      domains: {},
      agents: planner.registry?.agents || {}
    };

    for (const domain of allDomains) {
      const results = await mcpBridge.query(domain, 10);
      map.domains[domain] = results;
    }

    res.json({
      success: true,
      map
    });
  } catch (error) {
    console.error('Spatial map error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// ============================================================================
// WEBSOCKET FOR REAL-TIME UPDATES
// ============================================================================

const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('Client connected to WHAM runtime');

  // Send initial state
  ws.send(JSON.stringify({
    type: 'init',
    agents: Object.keys(planner.registry?.agents || {}),
    mcp_status: 'connected'
  }));

  // Listen for agent events
  runtime.on('agent_ready', (agentId) => {
    ws.send(JSON.stringify({
      type: 'agent_ready',
      agent_id: agentId
    }));
  });
});

// ============================================================================
// START SERVER
// ============================================================================

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 WHAM Integrated Server running on port ${PORT}`);
  console.log(`📊 MCP RAG Server: http://localhost:3333`);
  console.log(`🤖 Graph Planner: loaded`);
  console.log(`📋 A2A Registry: loaded`);
  console.log(`🎁 Reward Profiles: loaded`);
  console.log(`🌐 WebSocket: ws://localhost:8080`);
  console.log(`\nReady for 3D visualization at http://localhost:${PORT}`);
});
```

---

## Usage Examples

### Example 1: Query RAG and Visualize in 3D

**Client (WHAM Virtual Campus):**
```javascript
// In your browser console or WHAM UI
async function queryAndVisualize(query) {
  const response = await fetch('http://localhost:3000/api/rag/query', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, top_k: 5 })
  });

  const data = await response.json();

  // Visualize results in 3D world
  data.results.forEach(result => {
    whamEngine.createKnowledgeNode({
      text: result.text,
      position: result.position,
      domain: result.domain,
      score: result.score
    });
  });
}

// Example usage
queryAndVisualize("EMB system architecture");
```

### Example 2: Create Plan and Execute

**Client:**
```javascript
async function planAndExecute(product, domains) {
  // Step 1: Create plan
  const planResponse = await fetch('http://localhost:3000/api/plan', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ product, domains })
  });

  const { plan } = await planResponse.json();

  // Visualize plan in 3D
  visualizePlanGraph(plan);

  // Step 2: Execute plan
  const execResponse = await fetch('http://localhost:3000/api/execute', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ plan })
  });

  const { execution_results } = await execResponse.json();

  // Animate execution in 3D
  animateExecution(execution_results);
}

// Example usage
planAndExecute("P768 EMB", ["ZF-PROCESS", "ZF-REQ", "ZF-SYS"]);
```

### Example 3: Load Spatial Map on Startup

**Client:**
```javascript
async function loadSpatialMap() {
  const response = await fetch('http://localhost:3000/api/spatial/map');
  const { map } = await response.json();

  // Create 3D visualization of entire knowledge base
  Object.entries(map.domains).forEach(([domain, results]) => {
    const domainCluster = whamEngine.createDomainCluster(domain);
    
    results.forEach(result => {
      domainCluster.addKnowledgeNode({
        text: result.text,
        position: result.position,
        score: result.score
      });
    });
  });

  // Position agents based on registry
  Object.entries(map.agents).forEach(([agentId, agentConfig]) => {
    whamEngine.spawnAgent({
      id: agentId,
      name: agentConfig.name,
      position: agentConfig.position || { x: 0, y: 1.45, z: 0 }
    });
  });
}

// Call on page load
window.addEventListener('DOMContentLoaded', loadSpatialMap);
```

---

## File Structure After Integration

```
your-repo/
├── mcp_rag_embeddings.jsonl        # Existing
├── mcp_rag_server/                 # Existing
│   └── server.py                   # Running on :3333
├── planner/                        # Existing
│   └── graph_planner_moa.yaml
├── registry/                       # Existing
│   ├── registry.yaml
│   └── reward_profiles.yaml
├── runtime/                        # Existing
│   └── agent_runtime_loader.py
├── skills/                         # Existing
│   └── cdyp7-training-rag/
│       └── SKILL.md
├── .github/workflows/              # Existing
│   └── mcp-rag-ci.yml
│
├── server/                         # NEW - WHAM Integration
│   ├── integrated-server.js        # Main server
│   ├── wham-mcp-bridge.js          # MCP RAG connector
│   ├── graph-planner-loader.js     # YAML loader
│   └── agent-runtime-adapter.js    # Python runtime bridge
│
├── public/                         # NEW - Frontend
│   ├── wham-campus.html            # 3D virtual campus
│   ├── js/
│   │   ├── wham-engine.js          # Three.js engine
│   │   ├── mcp-client.js           # API client
│   │   └── visualizer.js           # 3D visualizations
│   └── assets/
│       └── shaders/
│
├── package.json                    # NEW - Node.js deps
└── README_INTEGRATION.md          # This file
```

---

## Deployment Steps

### 1. Keep Existing MCP RAG Server Running

```bash
# Terminal 1: Your existing MCP RAG server
uvicorn mcp_rag_server.server:app --port 3333
```

### 2. Install Node.js Dependencies

```bash
# Install new integration layer
npm install express cors ws js-yaml
```

### 3. Start Integrated WHAM Server

```bash
# Terminal 2: New WHAM integration server
node server/integrated-server.js
```

### 4. Open 3D Virtual Campus

```bash
# Browser
open http://localhost:3000/wham-campus.html
```

### 5. Verify Integration

```bash
# Test RAG query
curl -X POST http://localhost:3000/api/rag/query \
  -H "Content-Type: application/json" \
  -d '{"query": "system architecture", "top_k": 5}'

# Test planner
curl -X POST http://localhost:3000/api/plan \
  -H "Content-Type: application/json" \
  -d '{"product": "P768", "domains": ["ZF-PROCESS", "ZF-SYS"]}'

# Get spatial map
curl http://localhost:3000/api/spatial/map
```

---

## Benefits of Integration

✅ **Keep Existing Infrastructure**: Your MCP RAG server, graph planner, and agent runtime continue working as-is

✅ **Add 3D Visualization**: See your knowledge base and agent interactions in real-time 3D

✅ **Spatial Knowledge Navigation**: Walk through domains visually instead of text queries

✅ **Agent Behavior Visualization**: Watch handshake arcs, task execution, and reward profiles in action

✅ **Enhanced Debugging**: Visual representation of agent routing and plan execution

✅ **Better Onboarding**: New team members can explore knowledge spatially
