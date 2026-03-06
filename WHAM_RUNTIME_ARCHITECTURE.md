# WHAM·MCP Integrated Runtime Architecture

## System Overview

This document describes the complete WHAM (World Hierarchical Agent Model) runtime that transforms the 3D virtual campus into a **living knowledge base** where:

1. **Agent Avatars = LoRA Adapters** - Each 3D agent represents a specialized LoRA fine-tuned model
2. **Virtual Campus = Vector Store** - The spatial layout encodes institutional knowledge
3. **Document Upload → Vector Embedding** - Files become navigable knowledge in 3D space
4. **RAG Queries = Spatial Navigation** - Finding information means walking to the right agent

---

## Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT BROWSER                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  WHAM 3D Virtual Campus (Three.js)                 │    │
│  │  • Agent avatars with LoRA metadata                │    │
│  │  • Spatial vector manifold visualization          │    │
│  │  • Document upload interface                       │    │
│  └────────────────┬───────────────────────────────────┘    │
└───────────────────┼──────────────────────────────────────────┘
                    │ WebSocket + HTTP
┌───────────────────┼──────────────────────────────────────────┐
│            WHAM RUNTIME SERVER (Node.js)                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Document Processor                                  │    │
│  │  • File parsing (PDF, DOCX, TXT, MD)               │    │
│  │  • Chunking strategy (512 token windows)           │    │
│  │  • Metadata extraction                              │    │
│  └─────────┬───────────────────────────────────────────┘    │
│            │                                                  │
│  ┌─────────▼───────────────────────────────────────────┐    │
│  │  Vector Embedding Engine                             │    │
│  │  • Voyage AI / OpenAI embeddings                    │    │
│  │  • 384-dim vectors                                   │    │
│  │  • Spatial positioning (x, y, z) from semantics     │    │
│  └─────────┬───────────────────────────────────────────┘    │
│            │                                                  │
│  ┌─────────▼───────────────────────────────────────────┐    │
│  │  Spatial Vector Store                                │    │
│  │  • ChromaDB backend                                  │    │
│  │  • 3D coordinates as metadata                        │    │
│  │  • Agent proximity indexing                          │    │
│  └─────────┬───────────────────────────────────────────┘    │
│            │                                                  │
│  ┌─────────▼───────────────────────────────────────────┐    │
│  │  LoRA Agent Router                                   │    │
│  │  • Maps query → nearest agent(s)                    │    │
│  │  • Loads agent-specific LoRA weights                │    │
│  │  • Assembles context from spatial proximity         │    │
│  └─────────┬───────────────────────────────────────────┘    │
│            │                                                  │
│  ┌─────────▼───────────────────────────────────────────┐    │
│  │  LLM Inference Engine                                │    │
│  │  • Base model: Claude Sonnet 4 / GPT-4             │    │
│  │  • LoRA adapters per agent                          │    │
│  │  • RAG context injection                            │    │
│  └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Document Processing Pipeline

### File Upload Flow

```javascript
// client/document-upload.js
class DocumentUploader {
  constructor(whamRuntime) {
    this.runtime = whamRuntime;
    this.supportedFormats = ['pdf', 'docx', 'txt', 'md', 'pptx'];
  }

  async uploadDocument(file) {
    // 1. Validate file
    if (!this.validateFile(file)) {
      throw new Error('Unsupported file format');
    }

    // 2. Upload to server
    const formData = new FormData();
    formData.append('document', file);
    formData.append('metadata', JSON.stringify({
      name: file.name,
      size: file.size,
      type: file.type,
      uploadedAt: new Date().toISOString()
    }));

    const response = await fetch('/api/documents/upload', {
      method: 'POST',
      body: formData
    });

    const result = await response.json();

    // 3. Visualize in 3D world
    this.visualizeDocument(result);

    return result;
  }

  visualizeDocument(docData) {
    // Create document entity in 3D space
    const { chunks, position, assignedAgent } = docData;

    // Add glowing document icon at computed position
    const docMesh = this.runtime.scene.createDocumentMesh({
      position: new THREE.Vector3(position.x, position.y, position.z),
      color: assignedAgent.color,
      label: docData.filename,
      chunkCount: chunks.length
    });

    // Draw connection line to assigned agent
    this.runtime.scene.drawConnection(docMesh, assignedAgent.mesh);

    // Animate document "absorption" into agent's knowledge base
    this.animateKnowledgeAbsorption(docMesh, assignedAgent.mesh);
  }

  animateKnowledgeAbsorption(docMesh, agentMesh) {
    // Particle effect showing document being absorbed
    const particles = [];
    for (let i = 0; i < 20; i++) {
      const particle = new THREE.Mesh(
        new THREE.SphereGeometry(0.05),
        new THREE.MeshBasicMaterial({
          color: docMesh.material.color,
          transparent: true,
          opacity: 0.8
        })
      );
      particle.position.copy(docMesh.position);
      particles.push(particle);
      this.runtime.scene.add(particle);
    }

    // Animate particles flowing to agent
    const duration = 2000;
    const startTime = Date.now();

    const animate = () => {
      const elapsed = Date.now() - startTime;
      const progress = Math.min(elapsed / duration, 1);

      particles.forEach((particle, i) => {
        const offset = (i / particles.length) * Math.PI * 2;
        const spiral = {
          x: Math.cos(offset + progress * Math.PI * 4) * (1 - progress) * 2,
          y: progress * 3,
          z: Math.sin(offset + progress * Math.PI * 4) * (1 - progress) * 2
        };

        particle.position.lerpVectors(
          docMesh.position,
          agentMesh.position.clone().add(new THREE.Vector3(spiral.x, spiral.y, spiral.z)),
          progress
        );

        particle.material.opacity = 0.8 * (1 - progress);
      });

      if (progress < 1) {
        requestAnimationFrame(animate);
      } else {
        // Cleanup
        particles.forEach(p => this.runtime.scene.remove(p));
        this.runtime.scene.remove(docMesh);
        
        // Update agent's knowledge indicator
        this.runtime.updateAgentKnowledge(agentMesh.userData.agentId, 1);
      }
    };

    animate();
  }
}
```

### Server-Side Processing

```javascript
// server/document-processor.js
const pdf = require('pdf-parse');
const mammoth = require('mammoth');
const { RecursiveCharacterTextSplitter } = require('langchain/text_splitter');

class DocumentProcessor {
  constructor(vectorStore, embeddingEngine) {
    this.vectorStore = vectorStore;
    this.embeddings = embeddingEngine;
    this.splitter = new RecursiveCharacterTextSplitter({
      chunkSize: 512,
      chunkOverlap: 50
    });
  }

  async processDocument(file, metadata) {
    // 1. Extract text based on file type
    const text = await this.extractText(file);

    // 2. Split into chunks
    const chunks = await this.splitter.splitText(text);

    // 3. Generate embeddings for each chunk
    const embeddings = await this.embeddings.embedDocuments(chunks);

    // 4. Compute spatial positions from embeddings
    const positions = this.computeSpatialPositions(embeddings);

    // 5. Find nearest agent(s) for each chunk
    const assignments = await this.assignToAgents(embeddings, positions);

    // 6. Store in vector database
    const docId = await this.vectorStore.addDocuments(
      chunks.map((chunk, i) => ({
        id: `${metadata.name}_chunk_${i}`,
        text: chunk,
        embedding: embeddings[i],
        position: positions[i],
        assignedAgent: assignments[i],
        metadata: {
          ...metadata,
          chunkIndex: i,
          totalChunks: chunks.length
        }
      }))
    );

    // 7. Return results for visualization
    return {
      documentId: docId,
      filename: metadata.name,
      totalChunks: chunks.length,
      chunks: chunks.map((chunk, i) => ({
        text: chunk.substring(0, 100) + '...',
        position: positions[i],
        assignedAgent: assignments[i]
      })),
      position: this.computeDocumentCenter(positions),
      assignedAgent: this.findPrimaryAgent(assignments)
    };
  }

  async extractText(file) {
    const ext = file.originalname.split('.').pop().toLowerCase();

    switch (ext) {
      case 'pdf':
        const pdfData = await pdf(file.buffer);
        return pdfData.text;

      case 'docx':
        const docData = await mammoth.extractRawText({ buffer: file.buffer });
        return docData.value;

      case 'txt':
      case 'md':
        return file.buffer.toString('utf-8');

      case 'pptx':
        // Use pptx parser
        const pptxData = await this.parsePPTX(file.buffer);
        return pptxData;

      default:
        throw new Error(`Unsupported file type: ${ext}`);
    }
  }

  computeSpatialPositions(embeddings) {
    // Use UMAP/t-SNE to project high-dimensional embeddings to 3D
    // For simplicity, use PCA-like reduction
    return embeddings.map((emb, i) => {
      // Simple projection: use first 3 dimensions scaled
      const scale = 15; // Campus radius
      return {
        x: emb[0] * scale,
        y: Math.abs(emb[1]) * 2 + 0.5, // Height above ground
        z: emb[2] * scale
      };
    });
  }

  async assignToAgents(embeddings, positions) {
    // Load agent positions and expertise embeddings
    const agents = await this.vectorStore.getAgents();

    return positions.map((pos, i) => {
      // Find nearest agent in 3D space
      let minDist = Infinity;
      let nearestAgent = agents[0];

      agents.forEach(agent => {
        const dist = Math.sqrt(
          Math.pow(pos.x - agent.position.x, 2) +
          Math.pow(pos.y - agent.position.y, 2) +
          Math.pow(pos.z - agent.position.z, 2)
        );

        // Also consider semantic similarity
        const semanticSim = this.cosineSimilarity(embeddings[i], agent.expertiseEmbedding);
        const combinedScore = dist * 0.4 + (1 - semanticSim) * 0.6;

        if (combinedScore < minDist) {
          minDist = combinedScore;
          nearestAgent = agent;
        }
      });

      return {
        agentId: nearestAgent.id,
        name: nearestAgent.name,
        color: nearestAgent.color,
        distance: minDist
      };
    });
  }

  computeDocumentCenter(positions) {
    const center = positions.reduce(
      (acc, pos) => ({
        x: acc.x + pos.x,
        y: acc.y + pos.y,
        z: acc.z + pos.z
      }),
      { x: 0, y: 0, z: 0 }
    );

    return {
      x: center.x / positions.length,
      y: center.y / positions.length,
      z: center.z / positions.length
    };
  }

  cosineSimilarity(vecA, vecB) {
    const dotProduct = vecA.reduce((sum, a, i) => sum + a * vecB[i], 0);
    const magA = Math.sqrt(vecA.reduce((sum, a) => sum + a * a, 0));
    const magB = Math.sqrt(vecB.reduce((sum, b) => sum + b * b, 0));
    return dotProduct / (magA * magB);
  }
}

module.exports = { DocumentProcessor };
```

---

## 2. Spatial Vector Store

### ChromaDB Integration with 3D Coordinates

```python
# server/spatial_vector_store.py
import chromadb
from chromadb.config import Settings
import numpy as np
from typing import List, Dict, Tuple

class SpatialVectorStore:
    """
    Vector store that maintains 3D spatial coordinates as metadata.
    Each embedding is associated with a position in the virtual campus.
    """
    
    def __init__(self, persist_directory: str = "./chroma_db"):
        self.client = chromadb.Client(Settings(
            persist_directory=persist_directory,
            anonymized_telemetry=False
        ))
        
        # Create or get collection
        self.collection = self.client.get_or_create_collection(
            name="wham_campus_knowledge",
            metadata={
                "hnsw:space": "cosine",
                "description": "WHAM Virtual Campus spatial knowledge base"
            }
        )
        
        # Initialize agent registry
        self.agents = {}
        self._load_agents()
    
    def _load_agents(self):
        """Load CDYP7 agent registry with spatial positions and LoRA configs"""
        agent_configs = [
            {
                "id": "CDYP71",
                "name": "Orchestration Agent",
                "position": {"x": 0, "y": 1.45, "z": 9},
                "color": "#00e5ff",
                "lora_config": {
                    "rank": 16,
                    "alpha": 32,
                    "target_modules": ["q_proj", "v_proj", "k_proj", "o_proj"],
                    "lora_dropout": 0.05
                },
                "expertise": "Orchestration, planning, handshake coordination",
                "retrieval_k": 5
            },
            {
                "id": "DEC001",
                "name": "Decision Agent",
                "position": {"x": -7.8, "y": 1.45, "z": 4.5},
                "color": "#ff2d78",
                "lora_config": {
                    "rank": 8,
                    "alpha": 16,
                    "target_modules": ["q_proj", "v_proj"],
                    "lora_dropout": 0.1
                },
                "expertise": "Decision making, DMN evaluation, RBAC auditing",
                "retrieval_k": 3
            },
            {
                "id": "VEC001",
                "name": "Vector Bus Agent",
                "position": {"x": 7.8, "y": 1.45, "z": 4.5},
                "color": "#00ff9f",
                "lora_config": {
                    "rank": 4,
                    "alpha": 8,
                    "target_modules": ["q_proj", "v_proj"],
                    "lora_dropout": 0.05
                },
                "expertise": "Vector operations, embeddings, similarity search",
                "retrieval_k": 10
            },
            {
                "id": "RAG001",
                "name": "RAG Agent",
                "position": {"x": -7.8, "y": 1.45, "z": -4.5},
                "color": "#ff7629",
                "lora_config": {
                    "rank": 16,
                    "alpha": 32,
                    "target_modules": ["q_proj", "v_proj", "k_proj", "o_proj"],
                    "lora_dropout": 0.05
                },
                "expertise": "RAG operations, knowledge retrieval, context assembly",
                "retrieval_k": 5
            },
            # Add remaining agents...
        ]
        
        for agent in agent_configs:
            # Generate expertise embedding
            # In production, use actual embedding model
            expertise_emb = self._embed_text(agent["expertise"])
            
            agent["expertise_embedding"] = expertise_emb
            self.agents[agent["id"]] = agent
    
    def _embed_text(self, text: str) -> List[float]:
        """
        Generate embedding for text.
        In production, use Voyage AI or OpenAI embeddings.
        """
        # Placeholder - use actual embedding model
        import hashlib
        h = int(hashlib.md5(text.encode()).hexdigest(), 16)
        np.random.seed(h % (2**32))
        return np.random.randn(384).tolist()
    
    def add_documents(
        self,
        texts: List[str],
        embeddings: List[List[float]],
        positions: List[Dict[str, float]],
        agent_assignments: List[str],
        metadatas: List[Dict]
    ) -> List[str]:
        """
        Add documents to the spatial vector store.
        
        Args:
            texts: Document chunks
            embeddings: Pre-computed embeddings
            positions: 3D coordinates {x, y, z}
            agent_assignments: Agent IDs responsible for each chunk
            metadatas: Additional metadata per chunk
        
        Returns:
            List of document IDs
        """
        ids = [f"doc_{i}_{hash(text)}" for i, text in enumerate(texts)]
        
        # Enhance metadata with spatial and agent info
        enhanced_metadata = []
        for i, meta in enumerate(metadatas):
            enhanced_metadata.append({
                **meta,
                "position_x": positions[i]["x"],
                "position_y": positions[i]["y"],
                "position_z": positions[i]["z"],
                "assigned_agent": agent_assignments[i],
                "agent_distance": self._compute_agent_distance(
                    positions[i], 
                    self.agents[agent_assignments[i]]["position"]
                )
            })
        
        # Add to ChromaDB
        self.collection.add(
            ids=ids,
            embeddings=embeddings,
            documents=texts,
            metadatas=enhanced_metadata
        )
        
        return ids
    
    def query_spatial(
        self,
        query_text: str,
        query_position: Dict[str, float] = None,
        n_results: int = 5,
        agent_filter: str = None,
        spatial_radius: float = None
    ) -> Dict:
        """
        Query the vector store with optional spatial filtering.
        
        Args:
            query_text: Search query
            query_position: Optional 3D position to search near
            n_results: Number of results to return
            agent_filter: Optional agent ID to filter by
            spatial_radius: Optional radius to search within
        
        Returns:
            Query results with spatial annotations
        """
        # Generate query embedding
        query_embedding = self._embed_text(query_text)
        
        # Build where clause for filtering
        where_clause = {}
        if agent_filter:
            where_clause["assigned_agent"] = agent_filter
        
        # Query ChromaDB
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=n_results * 2,  # Get extra for spatial filtering
            where=where_clause if where_clause else None
        )
        
        # Apply spatial filtering if position provided
        if query_position and spatial_radius:
            filtered_results = self._filter_by_spatial_radius(
                results,
                query_position,
                spatial_radius
            )
        else:
            filtered_results = results
        
        # Limit to requested number
        return self._format_results(filtered_results, n_results)
    
    def _compute_agent_distance(
        self,
        pos1: Dict[str, float],
        pos2: Dict[str, float]
    ) -> float:
        """Compute Euclidean distance between two 3D positions"""
        return np.sqrt(
            (pos1["x"] - pos2["x"]) ** 2 +
            (pos1["y"] - pos2["y"]) ** 2 +
            (pos1["z"] - pos2["z"]) ** 2
        )
    
    def _filter_by_spatial_radius(
        self,
        results: Dict,
        center: Dict[str, float],
        radius: float
    ) -> Dict:
        """Filter results to only include items within spatial radius"""
        filtered = {
            "ids": [],
            "documents": [],
            "metadatas": [],
            "distances": []
        }
        
        for i, meta in enumerate(results["metadatas"][0]):
            pos = {
                "x": meta["position_x"],
                "y": meta["position_y"],
                "z": meta["position_z"]
            }
            
            dist = self._compute_agent_distance(center, pos)
            
            if dist <= radius:
                filtered["ids"].append(results["ids"][0][i])
                filtered["documents"].append(results["documents"][0][i])
                filtered["metadatas"].append(meta)
                filtered["distances"].append(results["distances"][0][i])
        
        return filtered
    
    def get_agent_knowledge(self, agent_id: str) -> Dict:
        """Get all knowledge chunks assigned to a specific agent"""
        results = self.collection.get(
            where={"assigned_agent": agent_id},
            include=["documents", "metadatas", "embeddings"]
        )
        
        return {
            "agent": self.agents[agent_id],
            "knowledge_count": len(results["ids"]),
            "chunks": [
                {
                    "text": doc,
                    "metadata": meta,
                    "position": {
                        "x": meta["position_x"],
                        "y": meta["position_y"],
                        "z": meta["position_z"]
                    }
                }
                for doc, meta in zip(results["documents"], results["metadatas"])
            ]
        }
    
    def _format_results(self, results: Dict, limit: int) -> Dict:
        """Format query results for API response"""
        return {
            "results": [
                {
                    "id": results["ids"][0][i],
                    "text": results["documents"][0][i],
                    "score": 1 - results["distances"][0][i],  # Convert distance to similarity
                    "position": {
                        "x": results["metadatas"][0][i]["position_x"],
                        "y": results["metadatas"][0][i]["position_y"],
                        "z": results["metadatas"][0][i]["position_z"]
                    },
                    "assigned_agent": results["metadatas"][0][i]["assigned_agent"],
                    "metadata": results["metadatas"][0][i]
                }
                for i in range(min(limit, len(results["ids"][0])))
            ]
        }
```

---

## 3. LoRA Agent Router

### Dynamic LoRA Loading Based on Spatial Context

```python
# server/lora_agent_router.py
from typing import List, Dict, Optional
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel, LoraConfig, get_peft_model

class LoRAAgentRouter:
    """
    Routes queries to appropriate agent LoRAs based on:
    1. Semantic similarity
    2. Spatial proximity in virtual campus
    3. Agent expertise domains
    """
    
    def __init__(
        self,
        base_model_name: str = "meta-llama/Llama-2-7b-chat-hf",
        lora_dir: str = "./lora_adapters"
    ):
        self.base_model_name = base_model_name
        self.lora_dir = lora_dir
        
        # Load base model (shared across all agents)
        print(f"Loading base model: {base_model_name}")
        self.base_model = AutoModelForCausalLM.from_pretrained(
            base_model_name,
            torch_dtype=torch.float16,
            device_map="auto"
        )
        self.tokenizer = AutoTokenizer.from_pretrained(base_model_name)
        
        # Cache for loaded LoRA adapters
        self.lora_cache = {}
        
        # Agent registry (matches spatial vector store)
        self.agents = self._load_agent_registry()
    
    def _load_agent_registry(self) -> Dict:
        """Load agent configurations with LoRA paths"""
        return {
            "CDYP71": {
                "name": "Orchestration Agent",
                "lora_path": f"{self.lora_dir}/orchestration_r16_a32",
                "expertise": ["planning", "coordination", "handshake"],
                "lora_config": {"r": 16, "lora_alpha": 32}
            },
            "DEC001": {
                "name": "Decision Agent",
                "lora_path": f"{self.lora_dir}/decision_r8_a16",
                "expertise": ["decision_making", "dmn", "rbac"],
                "lora_config": {"r": 8, "lora_alpha": 16}
            },
            "VEC001": {
                "name": "Vector Bus Agent",
                "lora_path": f"{self.lora_dir}/vector_r4_a8",
                "expertise": ["embeddings", "vector_search", "similarity"],
                "lora_config": {"r": 4, "lora_alpha": 8}
            },
            "RAG001": {
                "name": "RAG Agent",
                "lora_path": f"{self.lora_dir}/rag_r16_a32",
                "expertise": ["retrieval", "context_assembly", "knowledge"],
                "lora_config": {"r": 16, "lora_alpha": 32}
            },
            # Add other agents...
        }
    
    def load_lora_adapter(self, agent_id: str) -> PeftModel:
        """
        Load LoRA adapter for specific agent.
        Caches loaded adapters for performance.
        """
        if agent_id in self.lora_cache:
            return self.lora_cache[agent_id]
        
        agent = self.agents[agent_id]
        lora_path = agent["lora_path"]
        
        print(f"Loading LoRA for {agent_id} from {lora_path}")
        
        # Load LoRA adapter
        model = PeftModel.from_pretrained(
            self.base_model,
            lora_path,
            torch_dtype=torch.float16
        )
        
        # Cache for reuse
        self.lora_cache[agent_id] = model
        
        return model
    
    def route_query(
        self,
        query: str,
        spatial_context: Optional[Dict] = None,
        rag_results: Optional[List[Dict]] = None
    ) -> Dict:
        """
        Route query to appropriate agent(s) and generate response.
        
        Args:
            query: User query text
            spatial_context: Optional 3D position context
            rag_results: Retrieved knowledge chunks from spatial vector store
        
        Returns:
            Response with agent info and generated text
        """
        # 1. Determine which agent(s) to use
        selected_agents = self._select_agents(query, spatial_context, rag_results)
        
        # 2. Load primary agent's LoRA
        primary_agent_id = selected_agents[0]["id"]
        model = self.load_lora_adapter(primary_agent_id)
        
        # 3. Assemble context from RAG results
        context = self._assemble_context(rag_results, selected_agents)
        
        # 4. Build prompt with agent identity
        prompt = self._build_prompt(
            query=query,
            agent=self.agents[primary_agent_id],
            context=context,
            spatial_info=spatial_context
        )
        
        # 5. Generate response
        response = self._generate(model, prompt)
        
        return {
            "answer": response,
            "primary_agent": {
                "id": primary_agent_id,
                "name": self.agents[primary_agent_id]["name"],
                "lora_config": self.agents[primary_agent_id]["lora_config"]
            },
            "supporting_agents": selected_agents[1:],
            "rag_sources": rag_results,
            "spatial_context": spatial_context
        }
    
    def _select_agents(
        self,
        query: str,
        spatial_context: Optional[Dict],
        rag_results: Optional[List[Dict]]
    ) -> List[Dict]:
        """
        Select agent(s) to handle query based on:
        1. Query semantic content
        2. Spatial proximity (if context provided)
        3. Agent assignments in RAG results
        """
        agent_scores = {}
        
        # Score by semantic match
        query_lower = query.lower()
        for agent_id, agent in self.agents.items():
            score = 0.0
            for expertise_term in agent["expertise"]:
                if expertise_term in query_lower:
                    score += 1.0
            agent_scores[agent_id] = score
        
        # Boost score for agents in RAG results
        if rag_results:
            for result in rag_results:
                assigned_agent = result.get("assigned_agent")
                if assigned_agent in agent_scores:
                    agent_scores[assigned_agent] += 2.0
        
        # Sort by score
        sorted_agents = sorted(
            agent_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )
        
        # Return top agents
        return [
            {
                "id": agent_id,
                "score": score,
                "name": self.agents[agent_id]["name"]
            }
            for agent_id, score in sorted_agents[:3]
            if score > 0
        ]
    
    def _assemble_context(
        self,
        rag_results: Optional[List[Dict]],
        selected_agents: List[Dict]
    ) -> str:
        """Assemble RAG context for prompt"""
        if not rag_results:
            return ""
        
        context_parts = ["Retrieved Knowledge:\n"]
        
        for i, result in enumerate(rag_results[:5], 1):
            context_parts.append(
                f"[{i}] (score: {result['score']:.2f}, agent: {result['assigned_agent']})\n"
                f"{result['text']}\n"
            )
        
        return "\n".join(context_parts)
    
    def _build_prompt(
        self,
        query: str,
        agent: Dict,
        context: str,
        spatial_info: Optional[Dict]
    ) -> str:
        """Build prompt with agent identity and context"""
        prompt_parts = [
            f"You are the {agent['name']}, specialized in: {', '.join(agent['expertise'])}.",
            f"\nYour LoRA configuration: rank={agent['lora_config']['r']}, alpha={agent['lora_config']['lora_alpha']}",
        ]
        
        if spatial_info:
            prompt_parts.append(
                f"\nSpatial context: position=({spatial_info.get('x', 0):.1f}, "
                f"{spatial_info.get('y', 0):.1f}, {spatial_info.get('z', 0):.1f})"
            )
        
        if context:
            prompt_parts.append(f"\n{context}")
        
        prompt_parts.append(f"\nQuery: {query}\n\nResponse:")
        
        return "\n".join(prompt_parts)
    
    def _generate(self, model: PeftModel, prompt: str, max_length: int = 512) -> str:
        """Generate response using LoRA model"""
        inputs = self.tokenizer(prompt, return_tensors="pt").to(model.device)
        
        with torch.no_grad():
            outputs = model.generate(
                **inputs,
                max_new_tokens=max_length,
                temperature=0.7,
                top_p=0.9,
                do_sample=True
            )
        
        response = self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        
        # Extract response after prompt
        response = response[len(prompt):].strip()
        
        return response
```

---

## 4. Complete API Server

```javascript
// server/wham-runtime-server.js
const express = require('express');
const multer = require('multer');
const cors = require('cors');
const { DocumentProcessor } = require('./document-processor');
const { PythonShell } = require('python-shell');
const WebSocket = require('ws');

const app = express();
const upload = multer({ storage: multer.memoryStorage() });

app.use(cors());
app.use(express.json());
app.use(express.static('public'));

// Initialize components
const vectorStore = initializeVectorStore();
const docProcessor = new DocumentProcessor(vectorStore);

// WebSocket for real-time updates
const wss = new WebSocket.Server({ port: 8080 });

// ============================================================================
// API ENDPOINTS
// ============================================================================

// Upload document
app.post('/api/documents/upload', upload.single('document'), async (req, res) => {
  try {
    const file = req.file;
    const metadata = JSON.parse(req.body.metadata);

    console.log(`Processing document: ${file.originalname}`);

    // Process document
    const result = await docProcessor.processDocument(file, metadata);

    // Broadcast to connected clients
    broadcast({
      type: 'document_uploaded',
      data: result
    });

    res.json({
      success: true,
      ...result
    });
  } catch (error) {
    console.error('Document upload error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Query with RAG
app.post('/api/query', async (req, res) => {
  try {
    const { query, spatial_context, agent_filter, use_lora } = req.body;

    // 1. Query spatial vector store
    const ragResults = await queryVectorStore({
      query,
      spatial_context,
      agent_filter
    });

    // 2. Route to LoRA agent if requested
    let response;
    if (use_lora) {
      response = await routeToLoRAAgent({
        query,
        spatial_context,
        rag_results: ragResults.results
      });
    } else {
      // Use base LLM without LoRA
      response = await queryBaseLLM({
        query,
        context: ragResults.results
      });
    }

    res.json({
      success: true,
      ...response,
      rag_results: ragResults.results
    });
  } catch (error) {
    console.error('Query error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Get agent knowledge summary
app.get('/api/agents/:agentId/knowledge', async (req, res) => {
  try {
    const { agentId } = req.params;

    const knowledge = await PythonShell.run('get_agent_knowledge.py', {
      args: [agentId]
    });

    res.json({
      success: true,
      agent_id: agentId,
      knowledge: JSON.parse(knowledge[0])
    });
  } catch (error) {
    console.error('Agent knowledge error:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Get spatial map
app.get('/api/spatial/map', async (req, res) => {
  try {
    const map = await vectorStore.getSpatialMap();
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
// HELPER FUNCTIONS
// ============================================================================

function queryVectorStore(params) {
  return new Promise((resolve, reject) => {
    PythonShell.run('query_vector_store.py', {
      args: [JSON.stringify(params)]
    }, (err, results) => {
      if (err) reject(err);
      else resolve(JSON.parse(results[0]));
    });
  });
}

function routeToLoRAAgent(params) {
  return new Promise((resolve, reject) => {
    PythonShell.run('route_lora_agent.py', {
      args: [JSON.stringify(params)]
    }, (err, results) => {
      if (err) reject(err);
      else resolve(JSON.parse(results[0]));
    });
  });
}

async function queryBaseLLM(params) {
  // Use Anthropic API without LoRA
  const anthropic = require('@anthropic-ai/sdk');
  const client = new anthropic.Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  const context = params.context
    .map((r, i) => `[${i + 1}] ${r.text}`)
    .join('\n\n');

  const message = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    messages: [{
      role: 'user',
      content: `Context:\n${context}\n\nQuery: ${params.query}`
    }]
  });

  return {
    answer: message.content[0].text,
    model: 'claude-sonnet-4',
    lora_used: false
  };
}

function broadcast(message) {
  wss.clients.forEach(client => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(message));
    }
  });
}

// ============================================================================
// START SERVER
// ============================================================================

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 WHAM Runtime Server running on port ${PORT}`);
  console.log(`📊 Vector Store: ChromaDB`);
  console.log(`🤖 LoRA Agents: 7 configured`);
  console.log(`🌐 WebSocket: ws://localhost:8080`);
});
```

---

## 5. Complete System Summary

### What This System Provides

**1. Document → Knowledge Pipeline**
- Upload PDF/DOCX/TXT/MD/PPTX files
- Automatic chunking (512 tokens)
- Embedding generation (384-dim vectors)
- 3D spatial positioning based on semantics
- Assignment to nearest specialized agent

**2. Spatial Vector Store**
- ChromaDB backend with 3D coordinates
- Each chunk has (x, y, z) position in virtual campus
- Proximity-based retrieval
- Agent-centric knowledge organization

**3. Agent Avatar = LoRA Adapter**
- Each 3D agent represents a specialized LoRA fine-tune
- Base model: Llama-2-7b (or Claude API)
- Different LoRA ranks per agent (r=4 to r=16)
- Dynamic loading based on query routing

**4. Spatial RAG**
- Query by text similarity
- Filter by 3D proximity
- Agent-specific knowledge retrieval
- Multi-agent consultation

**5. Visual Knowledge Navigation**
- Walk through 3D campus to explore knowledge
- See documents as glowing particles absorbed by agents
- Agent halos grow brighter with more knowledge
- Spatial clusters reveal knowledge domains

### System Requirements

**Server:**
- Node.js 18+
- Python 3.10+
- 16GB+ RAM (for LoRA models)
- CUDA GPU (optional, for faster inference)

**Dependencies:**
```bash
# Node.js
npm install express multer cors ws @anthropic-ai/sdk python-shell

# Python
pip install chromadb transformers peft torch accelerate sentence-transformers
```

**Storage:**
- ChromaDB: ~1GB per 10,000 documents
- LoRA adapters: ~500MB per agent (7 agents = 3.5GB)
- Base model: ~13GB (Llama-2-7b-chat)

### Deployment

```bash
# 1. Clone repository
git clone https://github.com/zf/wham-runtime.git
cd wham-runtime

# 2. Install dependencies
npm install
pip install -r requirements.txt

# 3. Configure environment
cp .env.example .env
# Edit: ANTHROPIC_API_KEY, CUDA_VISIBLE_DEVICES, etc.

# 4. Initialize vector store
python scripts/init_vector_store.py

# 5. Train LoRA adapters (optional)
python scripts/train_lora_adapters.py

# 6. Start server
npm start

# 7. Open browser
open http://localhost:3000
```

---

This complete system transforms your WHAM virtual campus into a **living, navigable knowledge base** where documents become part of the 3D world and AI agents are specialized LoRA adapters that can be consulted by walking to them or querying their expertise domains.
