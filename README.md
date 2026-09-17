# Mini HDFS - Distributed File System Implementation

A lightweight implementation of the **Hadoop Distributed File System (HDFS)** architecture, built with Python and featuring a modern web-based user interface for file management and system monitoring.

##  Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [System Components](#system-components)
- [Technology Stack](#technology-stack)
- [Installation & Setup](#installation--setup)
- [Usage](#usage)
- [Configuration](#configuration)
- [API Endpoints](#api-endpoints)
- [Known Limitations](#known-limitations)
- [Project Structure](#project-structure)

## Overview

Mini HDFS is a simplified yet functional implementation of the distributed file system pattern used in Hadoop. It demonstrates key concepts including:

- **Distributed storage** across multiple datanodes
- **Data replication** for fault tolerance
- **Metadata management** via a centralized namenode
- **Chunk-based file storage** with configurable block size
- **Node health monitoring** via heartbeat mechanism
- **Client-facing API** for file operations

This project is ideal for learning distributed systems concepts or as a foundation for distributed storage research.

## Architecture

### System Design

The system follows a **master-slave architecture** pattern:

```
┌─────────────────────────────────────────────────────────┐
│                    Client Layer (FastAPI)               │
│              Handles file upload/download requests       │
└─────────────┬───────────────────────────────────────────┘
              │
              ├─── Metadata Requests ──────────┐
              │                                 │
              │      Chunk Replication Data     │
              │                                 │
       ┌──────▼──────────────┐        ┌───────▼──────┐
       │   NameNode          │        │   DataNodes  │
       │  (Master)           │◄──────►│  (Slaves)    │
       │                     │        │              │
       │ - Metadata Store    │        │ - Chunk      │
       │ - File Registry     │        │   Storage    │
       │ - Replication Plan  │        │ - Heartbeats │
       │ - Node Health       │        │ - Pipelines  │
       └─────────────────────┘        └──────────────┘
```

### Read & Write Pipelines

**Write Pipeline** (`./images/write.png`):
- Client uploads file → Backend chunks the file → Sends chunks to primary datanode → Pipeline replication across replica nodes

![Mini HDFS write pipeline](./images/write.png)

**Read Pipeline** (`./images/read.png`):
- Client requests file → NameNode returns chunk locations → Client fetches chunks from nearest available datanode

![Mini HDFS read pipeline](./images/read.png)

## Features

 **Core Features:**
- File upload with automatic chunking (default 2MB)
- Distributed chunk storage across multiple datanodes
- Configurable replication factor (default: 2)
- Chunk-level pipeline replication
- File integrity verification via MD5 hashing
- File listing and metadata retrieval
- Chunk hash verification
- Datanode health monitoring via heartbeat mechanism

 **Web Dashboard:**
- Modern, responsive UI built with Next.js 14 and React
- Real-time file browser
- Upload/download functionality
- System status monitoring
- Datanode health visualization
- Styled with Tailwind CSS

 **API:**
- FastAPI-based REST endpoints
- Async file operations
- JSON-based communication between components

## System Components

### 1. **NameNode** (Master)
- **Port:** 5000 (heartbeat), 5050 (client requests)
- **Responsibility:** Metadata management and file registry
- **Functions:**
  - Stores file metadata (filename, size, chunks, replication info)
  - Tracks datanode status and availability
  - Plans chunk placement strategy for new writes
  - Monitors datanode heartbeats (default: every 3 seconds)
  - Marks nodes as dead after 9 seconds of silence

**Key Files:**
- `namenode/main.py` - Request handler
- `namenode/heartbeat_handler.py` - Node health monitoring
- `namenode/db_utils.py` - Metadata persistence

### 2. **DataNode** (Slave)
- **Ports:** 5001 (datanode1), 5002 (datanode2), configurable for additional nodes
- **Responsibility:** Chunk storage and replication
- **Functions:**
  - Stores file chunks locally
  - Sends periodic heartbeats to NameNode
  - Participates in replication pipelines
  - Handles chunk replication from upstream nodes
  - Verifies chunk integrity via hashing

**Key Files:**
- `datanode/main.py` - Chunk storage and pipeline handler
- `datanode/server.py` - Network communication
- `datanode/heartbeat_handler.py` - Heartbeat generation

### 3. **Backend Server** (FastAPI)
- **Port:** 8000
- **Responsibility:** Client-facing API and file orchestration
- **Functions:**
  - Accepts file uploads from frontend
  - Chunks files into blocks
  - Coordinates chunk placement via NameNode
  - Handles file reads by requesting chunks from datanodes
  - Returns files to client

**Key Files:**
- `client/central_server.py` - API endpoints and orchestration

### 4. **Frontend Dashboard** (Next.js)
- **Port:** 3000
- **Responsibility:** User-facing web interface
- **Features:**
  - File upload interface
  - File browser with download capability
  - System monitoring dashboard
  - Real-time status updates

**Key Files:**
- `client/frontend/app/page.tsx` - Main dashboard
- `client/frontend/app/dashboard/page.tsx` - Dashboard view

## Technology Stack

**Backend:**
- Python 3.x
- FastAPI 0.104.1 (REST API framework)
- Uvicorn 0.24.0 (ASGI server)
- Socket programming (TCP communication)
- Ping3 4.0.4 (Node health checks)

**Frontend:**
- Next.js 14.0.0
- React 18.2.0
- TypeScript 5.2.2
- Tailwind CSS 3.3.5
- Radix UI components
- Lucide React (icons)

**Infrastructure:**
- Docker & Docker Compose (containerization)
- Network: Bridge network (dfsnet)

## Installation & Setup

### Prerequisites
- Docker & Docker Compose
- Python 3.8+ (for standalone setup)
- Node.js 18+ (for frontend development)

### Option 1: Docker Compose (Recommended)

1. **Clone or navigate to project directory:**
   ```bash
   cd mini-HDFS
   ```

2. **Build and start all services:**
   ```bash
   docker-compose up --build
   ```

3. **Access services:**
   - Frontend Dashboard: http://localhost:3000
   - Backend API: http://localhost:8000
   - NameNode: http://localhost:5000
   - DataNode1: http://localhost:5001
   - DataNode2: http://localhost:5002

### Option 2: Manual Setup (Distributed Across Machines)

#### NameNode Setup
```bash
cd namenode
export NAMENODE_HOST=<this-machine-ip>
export DATANODE1_HOST=<datanode1-ip>
export DATANODE2_HOST=<datanode2-ip>
pip install -r requirements.txt
python main.py
```

#### DataNode Setup
```bash
cd datanode
export NAMENODE_HOST=<namenode-ip>
export DATANODE_HOST=<this-machine-ip>
export DATANODE_ID=<1 or 2>
pip install -r requirements.txt
sudo -E python main.py  # sudo required for storage access
```

#### Backend Server Setup
```bash
cd client
export NAMENODE_HOST=<namenode-ip>
export NAMENODE_PORT=5000
export DATANODE1_HOST=<datanode1-ip>
export DATANODE1_PORT=5001
export DATANODE2_HOST=<datanode2-ip>
export DATANODE2_PORT=5002
pip install -r requirements.txt
python central_server.py
```

#### Frontend Setup
```bash
cd client/frontend
npm install
# Edit .env.local with backend IP
NEXT_PUBLIC_API_URL=http://<backend-ip>:8000
npm run dev
```

## Usage

### Web Dashboard (Recommended)
1. Navigate to `http://localhost:3000`
2. Use the upload form to select and upload files
3. Monitor system status and datanodes
4. Click on files to download them

### API Endpoints

**File Operations:**
- `POST /upload/` - Upload a file
  - Parameters: `file` (multipart/form-data)
  - Returns: File ID, number of chunks, chunk locations

- `GET /download/{filename}` - Download a file
  - Parameters: `filename` (query)
  - Returns: File content (streaming)

- `GET /files/` - List all files in the system
  - Returns: Array of file metadata

- `GET /datanodes/status` - Get datanode status
  - Returns: Status of all datanodes and node health

## Configuration

All configuration is managed via environment variables in `shared/config.py`:

```python
CHUNK_SIZE = 2 * 1024 * 1024           # Default: 2MB
HEARTBEAT_INTERVAL = 3                 # Seconds between heartbeats
DEAD_NODE_THRESHOLD = 9                # Seconds before marking node as dead
MONITOR_INTERVAL = 5                   # Namenode monitoring interval
REPLICATION_FACTOR = 2                 # Copies per chunk
```

### Environment Variables

**Common:**
```bash
NAMENODE_HOST          # NameNode address
NAMENODE_PORT          # NameNode heartbeat port (5000)
NAMENODE_REQ_PORT      # NameNode request port (5050)
DATANODE_HOST          # Current DataNode address
DATANODE_PORT          # Current DataNode port
DATANODE_ID            # DataNode identifier (1 or 2)
BACKEND_HOST           # Backend server address
BACKEND_PORT           # Backend server port (8000)
```

## API Endpoints

### NameNode Requests (Port 5050)

**Write Request:**
```json
{
  "type": "write_req",
  "filename": "document.pdf",
  "num_chunks": 3
}

Response:
{
  "file_id": "b10a2f8a-97a5-4d30-becb...",
  "placements": {
    "chunk_1": ["datanode1:5001", "datanode2:5002"],
    "chunk_2": ["datanode2:5002", "datanode1:5001"],
    "chunk_3": ["datanode1:5001", "datanode2:5002"]
  }
}
```

**Metadata Write:**
```json
{
  "type": "metadata_write",
  "filename": "document.pdf",
  "file_hash": "a1b2c3d4...",
  "file_size": 6291456,
  "num_chunks": 3,
  "placements": {...},
  "chunk_hashes": ["hash1", "hash2", "hash3"]
}
```

**Read Request:**
```json
{
  "type": "read_req",
  "subtype": "list_files"
}

Response:
{
  "status": "ok",
  "files": [
    {
      "filename": "document.pdf",
      "file_size": 6291456,
      "num_chunks": 3,
      "file_hash": "a1b2c3d4..."
    }
  ]
}
```

## Known Limitations

 **Partial Features:**
- **Automatic Re-replication:** The system tracks dead nodes but does **not automatically re-replicate** chunks from failed nodes to healthy ones. Manual intervention or system restart is required to rebuild lost replicas.

 **Other Considerations:**
- Default to 2 datanodes; adding more requires configuration updates
- Replication factor fixed at build time
- No authentication/authorization implemented
- No encryption for data in transit
- Metadata not distributed (single point of failure if namenode goes down)
- No data consistency checks after corruption detection

## Project Structure

```
mini-HDFS/
├── docker-compose.yaml          # Service orchestration
├── README.md                     # This file
│
├── shared/                       # Common utilities
│   ├── commons.py               # Socket utilities
│   └── config.py                # Global configuration
│
├── namenode/                     # NameNode (Master)
│   ├── main.py                  # Request handler
│   ├── server.py                # Network server
│   ├── heartbeat_handler.py     # Health monitoring
│   ├── db_utils.py              # Metadata storage
│   ├── Dockerfile               # Container definition
│   └── requirements.txt          # Python dependencies
│
├── datanode/                     # DataNode (Slave)
│   ├── main.py                  # Storage handler
│   ├── server.py                # Network server
│   ├── heartbeat_handler.py     # Heartbeat sender
│   ├── Dockerfile               # Container definition
│   ├── requirements.txt          # Python dependencies
│   └── storage[1-2]/            # Local chunk storage
│
├── client/                       # Client Components
│   ├── central_server.py        # FastAPI backend
│   ├── Dockerfile               # Container definition
│   ├── requirements.txt          # Python dependencies
│   │
│   └── frontend/                # Next.js Dashboard
│       ├── package.json         # Node dependencies
│       ├── tsconfig.json        # TypeScript config
│       ├── next.config.js       # Next.js config
│       ├── tailwind.config.js   # Tailwind config
│       ├── app/
│       │   ├── page.tsx         # Main page
│       │   ├── layout.tsx       # Root layout
│       │   └── dashboard/       # Dashboard pages
│       ├── components/          # React components
│       └── lib/                 # Utilities
│
├── images/                       # Architecture diagrams
│   ├── read.png                 # Read pipeline
│   └── write.png                # Write pipeline
│
└── namenode/storage/            # NameNode metadata store
```

## Development & Contribution

This project serves as an educational implementation. To extend functionality:

1. **Add more datanodes:** Update `docker-compose.yaml` and `config.py`
2. **Implement auto-replication:** Extend `heartbeat_handler.py` with rebuild logic
3. **Add persistence:** Enhance `db_utils.py` for metadata backup
4. **Improve frontend:** Add real-time dashboard updates with WebSockets
5. **Add monitoring:** Integrate Prometheus/Grafana for metrics

## License

This project is for educational purposes.

---

**For questions or issues, refer to the inline code comments and architecture diagrams in the `./images/` directory.**
