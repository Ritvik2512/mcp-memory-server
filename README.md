# MCP Memory Server for Claude

A semantic memory system built specifically for Claude using the Model Context Protocol (MCP). This project enables persistent, searchable memory for Claude Desktop by integrating vector embeddings with a custom MCP server.

> **Note:** Built before Anthropic released native memory. This project demonstrates semantic memory implementation via MCP and remains useful for custom memory backends.

---

## Features

- **MCP-compatible memory server** for Claude (JSON-RPC based)
- **Semantic memory** using transformer embeddings
- **Vector database integration** with Qdrant for efficient similarity search
- **Agent-based memory isolation** for multi-user/multi-agent scenarios
- **FastAPI backend** for handling MCP requests
- **HTTP ↔ MCP bridge** for Claude Desktop integration
- **Dockerized setup** for easy deployment
- **Infrastructure-ready** with load balancer support (AWS ALB)

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | Python, FastAPI |
| Embeddings | Sentence Transformers |
| Vector Database | Qdrant |
| Protocol | Model Context Protocol (MCP) |
| Infrastructure | Docker, Docker Compose |

---

## How It Works

1. Claude sends requests via MCP (JSON-RPC)
2. The memory server processes and stores data as vector embeddings
3. Data is stored in Qdrant for efficient similarity search
4. Relevant memories are retrieved and returned to Claude based on semantic similarity

---

## Project Structure

```
mcp-memory-server/
│
├── app/                    # Core MCP server + FastAPI bridge
│   ├── server.py          # Main MCP server implementation
│   ├── embeddings.py      # Embedding generation logic
│   └── qdrant_client.py   # Qdrant database interactions
│
├── config/                # Claude MCP configuration
│   └── claude_config.json # Claude Desktop integration config
│
├── scripts/               # Setup and deployment scripts
│   ├── setup.sh          # Initial setup
│   └── deploy.sh         # Deployment helper
│
├── docker/               # Docker configuration
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .env.example          # Environment variables template
├── requirements.txt      # Python dependencies
├── README.md            # This file
├── .gitignore
└── memory.py            # Main entry point
```

---

## Quick Start

### Prerequisites

- Python 3.8+
- Docker & Docker Compose (for containerized setup)
- 2GB+ RAM (for embeddings model)

### Option 1: Docker Compose (Recommended)

```bash
# Clone the repository
git clone https://github.com/Ritvik2512/mcp-memory-server.git
cd mcp-memory-server

# Set up environment
cp .env.example .env

# Start everything
docker-compose up --build
```

The server will start on `http://localhost:8000` with Qdrant on `http://localhost:6333`.

### Option 2: Local Setup

```bash
# Clone the repository
git clone https://github.com/Ritvik2512/mcp-memory-server.git
cd mcp-memory-server

# Set up environment
cp .env.example .env

# Install dependencies
pip install -r requirements.txt

# Start Qdrant (in a separate terminal)
docker run -d -p 6333:6333 -p 6334:6334 qdrant/qdrant

# Run the memory server
python memory.py
```

---

## Configuration

### Environment Variables

Create a `.env` file based on `.env.example`:

```env
# Qdrant Configuration
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_API_KEY=                    # Leave empty for local dev

# Server Configuration
SERVER_HOST=localhost
SERVER_PORT=8000
DEBUG=true

# Embeddings Configuration
EMBEDDING_MODEL=all-MiniLM-L6-v2   # Sentence Transformers model
COLLECTION_NAME=claude_memory      # Qdrant collection name

# Agent Configuration
DEFAULT_AGENT_ID=default
MAX_MEMORY_ITEMS=10000
EMBEDDING_BATCH_SIZE=32
```

### Claude Desktop Integration

1. **Locate Claude Desktop config:**
   - **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
   - **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
   - **Linux:** `~/.config/Claude/claude_desktop_config.json`

2. **Add the MCP server to `mcpServers`:**

```json
{
  "mcpServers": {
    "memory": {
      "command": "python",
      "args": ["/path/to/mcp-memory-server/memory.py"],
      "env": {
        "QDRANT_HOST": "localhost",
        "QDRANT_PORT": "6333"
      }
    }
  }
}
```

3. **Restart Claude Desktop** — the memory server will now be available to Claude.

---

## Usage

### Via Claude Desktop

Once configured, Claude can use memory through natural conversation:

```
You: "Remember that I prefer Python over JavaScript."
Claude: [Stores this in memory using semantic embeddings]

You: "What's my language preference?"
Claude: [Retrieves relevant memory] "You prefer Python over JavaScript."
```

### Via API (FastAPI)

```bash
# Store a memory
curl -X POST http://localhost:8000/memory/store \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "default",
    "content": "User prefers Python",
    "metadata": {"timestamp": "2024-01-01"}
  }'

# Search memory
curl http://localhost:8000/memory/search \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "default",
    "query": "programming languages",
    "limit": 5
  }'

# Retrieve all memories
curl http://localhost:8000/memory/retrieve \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "default"}'
```

---

## Deployment

### Docker Compose (Recommended)

```bash
docker-compose up --build -d
```

### AWS Deployment (with ALB)

Infrastructure setup includes support for AWS Application Load Balancer. Update `.env` with your AWS details and use the provided deployment scripts:

```bash
./scripts/deploy.sh
```

### Manual Server

```bash
# Install dependencies
pip install -r requirements.txt

# Run with production settings
python memory.py --host 0.0.0.0 --port 8000 --workers 4
```

---

## Architecture

```
┌─────────────────────┐
│   Claude Desktop    │
└──────────┬──────────┘
           │ (MCP/JSON-RPC)
           ↓
┌─────────────────────────────────────┐
│  MCP Memory Server (FastAPI)        │
│  ├── MCP Handler                    │
│  ├── Embedding Generator            │
│  └── Qdrant Client                  │
└──────────┬──────────────────────────┘
           │
           ↓
┌─────────────────────┐
│  Qdrant Vector DB   │
│  (Persistent Store) │
└─────────────────────┘
```

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/memory/store` | POST | Store a new memory with embeddings |
| `/memory/search` | POST | Search memory by semantic similarity |
| `/memory/retrieve` | POST | Get all memories for an agent |
| `/memory/delete` | DELETE | Remove a specific memory |
| `/health` | GET | Health check |

---

## Development

### Running Tests

```bash
pytest tests/
```

### Code Style

```bash
# Format code
black app/

# Lint
flake8 app/
```

---

## Limitations & Notes

- Embeddings are generated on-the-fly; for large-scale use, consider batch processing
- Qdrant storage is currently in-memory by default; enable persistence in `docker-compose.yml` for production
- Memory isolation is agent-based; implement authentication if deploying multi-user

---

## Future Work

- [ ] Web UI for memory management
- [ ] Support for multiple embedding models
- [ ] Advanced filtering and memory decay
- [ ] Backup and recovery mechanisms
- [ ] Multi-modal embeddings (text + images)

---

## Troubleshooting

### Qdrant Connection Error

```
ConnectionError: Failed to connect to Qdrant at localhost:6333
```

**Solution:** Ensure Qdrant is running:

```bash
docker ps  # Check if Qdrant container is active
docker-compose up qdrant  # Start Qdrant specifically
```

### Embedding Model Download

The first run will download the embedding model (~50MB). This is normal and happens once.

### Claude Desktop Not Recognizing Server

- Verify the config path is correct
- Restart Claude Desktop completely (not just refresh)
- Check `memory.py` is executable and path is absolute

---

## Contributing

Contributions are welcome! Feel free to open issues or submit PRs.

---

## License

MIT License — see LICENSE file for details.

---

## Acknowledgments

- Built with [Sentence Transformers](https://www.sbert.net/)
- Vector database by [Qdrant](https://qdrant.tech/)
- Protocol specification by [Anthropic's MCP](https://modelcontextprotocol.io/)
