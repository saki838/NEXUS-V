# Nexus-V — Agentic Multimodal Inventory Intelligence

> Real-time warehouse inventory auditing powered by Vision-Language Models,
> LangGraph agentic workflows, and a React Human-in-the-Loop dashboard.

---

## Architecture

```
Camera (RTSP)
     │
     ▼
[Frame Sampler]
     │  image bytes
     ▼
[FastAPI  /analyze-stream]
     │
     ▼
[LangGraph ReAct Agent]
  ├── node_analyze_image   ← Llama-4-Vision (VLM)
  ├── node_resolve_skus    ← CLIP embeddings → Milvus vector search
  ├── node_detect_anomalies
  └── node_take_action
        ├── trigger_reorder  → ERP (SAP / NetSuite)
        ├── log_anomaly      → PostgreSQL
        └── broadcast_alert  → WebSocket → React Dashboard
```

---

## Quick Start

### Prerequisites
- Docker + Docker Compose
- GPU recommended (CUDA 12+) for VLM inference; CPU works but is slow.

### 1. Clone & configure

```bash
git clone <repo> nexus-v
cd nexus-v
cp .env.example .env          # set ERP_URL and model paths
```

### 2. Start services

```bash
docker compose up -d
```

Services started:
| Service    | Port  | Purpose                        |
|------------|-------|--------------------------------|
| backend    | 8000  | FastAPI + LangGraph agent      |
| frontend   | 3000  | React HITL dashboard           |
| postgres   | 5432  | Anomaly log + WS broadcast     |
| milvus     | 19530 | SKU vector database            |

### 3. Seed the SKU database

```bash
python scripts/seed_milvus.py --catalog data/sku_catalog.csv
```

### 4. Send a test frame

```bash
curl -X POST http://localhost:8000/analyze-stream \
  -F "file=@data/sample_shelf.jpg" \
  -F "camera_id=cam-01"
```

---

## Key Files

| Path | Description |
|------|-------------|
| `models/vlm_inference.py` | Llama-4-Vision wrapper |
| `models/embedder.py` | CLIP product embedder |
| `backend/agents/inventory_agent.py` | LangGraph ReAct workflow |
| `backend/agents/tools.py` | ERP, Milvus, and alert tools |
| `backend/main.py` | FastAPI entry point + WebSocket |
| `backend/database.py` | PostgreSQL + Milvus init |
| `frontend/src/components/InventoryDashboard.jsx` | React HITL UI |
| `docker-compose.yml` | Full stack orchestration |

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgresql://nexus:nexus@postgres:5432/nexusv` | Postgres DSN |
| `MILVUS_HOST` | `milvus` | Milvus hostname |
| `ERP_URL` | `http://erp-mock:8080` | ERP reorder endpoint |
| `LOW_STOCK_THRESHOLD` | `20.0` | Fill % below which reorder triggers |

---

## Extending the Agent

Add new tools in `backend/agents/tools.py` and wire them into the
LangGraph graph in `inventory_agent.py`. The ReAct loop will call them
automatically based on detected anomaly types.
