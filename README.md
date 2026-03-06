
# CDYP7 MoA Graph Planner + MCP RAG Host (Role-Based)

## Quickstart
1. Ensure `mcp_rag_embeddings.jsonl` exists at repo root.
2. Start MCP RAG server:
   ```bash
   uvicorn mcp_rag_server.server:app --port 3333
   ```
3. Open the demo host:
   ```
   web/mcp_host_demo.html
   ```
4. Use `rag.query <text>` or `plan <product> <domains,...>`.

## Graph Planner
- Defined in `planner/graph_planner_moa.yaml`.
- Load/run example: `python runtime/agent_runtime_loader.py`.

## A2A Registry & Rewards
- `registry/registry.yaml`
- `registry/reward_profiles.yaml`

## Skills
- `skills/cdyp7-training-rag/SKILL.md`

## CI (GitHub Actions)
- `.github/workflows/mcp-rag-ci.yml` validates registry, rewards, RAG server, and skills.
