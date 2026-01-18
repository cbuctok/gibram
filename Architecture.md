# GibRAM Architecture

**GibRAM** (Graph in-Buffer Retrieval & Associative Memory) is a high-performance, in-memory knowledge graph server designed for Retrieval Augmented Generation (RAG) workflows.

## System Overview

```mermaid
flowchart TB
    subgraph Clients
        CLI[CLI Client]
        SDK[Python SDK]
        Custom[Custom Go Client]
    end

    subgraph Server["Server Layer"]
        TCP[TCP Server]
        Proto[Protobuf Codec]
        Auth[RBAC Auth]
        Rate[Rate Limiter]
    end

    subgraph Engine["Query Engine"]
        Eng[Engine]
        QLog[Query Log LRU]
    end

    subgraph Storage["Session Storage"]
        SM[Session Manager]
        SS1[Session Store 1]
        SS2[Session Store 2]
        SSN[Session Store N]
    end

    subgraph Indices["Per-Session Indices"]
        Doc[Documents]
        TU[TextUnits]
        Ent[Entities]
        Rel[Relationships]
        Com[Communities]
        VecTU[HNSW Index<br/>TextUnits]
        VecEnt[HNSW Index<br/>Entities]
        VecCom[HNSW Index<br/>Communities]
    end

    subgraph Persistence["Persistence Layer"]
        WAL[Write-Ahead Log]
        Snap[Snapshots]
        Rec[Recovery]
    end

    CLI --> TCP
    SDK --> TCP
    Custom --> TCP
    TCP --> Proto
    Proto --> Auth
    Auth --> Rate
    Rate --> Eng
    Eng --> SM
    SM --> SS1
    SM --> SS2
    SM --> SSN
    SS1 --> Doc
    SS1 --> TU
    SS1 --> Ent
    SS1 --> Rel
    SS1 --> Com
    TU --> VecTU
    Ent --> VecEnt
    Com --> VecCom
    Eng --> WAL
    WAL --> Snap
    Snap --> Rec
```

## Core Components

### 1. Server (`pkg/server/`)

The TCP server handles client connections using a custom binary protocol over TLS or plain TCP.

**Responsibilities:**
- Connection lifecycle management
- Protocol framing (4-byte length prefix + protobuf payload)
- Session binding per client
- Authentication and authorization
- Rate limiting and connection limits

**Wire Protocol:**
```
┌─────────┬────────────┬──────────────┬─────────┬────────────┐
│ Version │ Request ID │ Command Type │ Payload │ Session ID │
│ (1 byte)│ (4 bytes)  │ (2 bytes)    │ (var)   │ (var)      │
└─────────┴────────────┴──────────────┴─────────┴────────────┘
```

### 2. Engine (`pkg/engine/`)

The query execution engine coordinates vector search and graph traversal across sessions.

**Responsibilities:**
- Session lifecycle (creation, TTL, cleanup)
- Query execution combining vector similarity and graph traversal
- Query logging with LRU eviction (10K entries)
- Idle session detection and expiration

```mermaid
flowchart LR
    Q[Query] --> VS[Vector Search]
    Q --> GT[Graph Traversal]
    VS --> TUI[TextUnit Index]
    VS --> EI[Entity Index]
    VS --> CI[Community Index]
    GT --> Seeds[Seed Entities]
    Seeds --> Hops[K-Hop Expansion]
    TUI --> Agg[Aggregation]
    EI --> Agg
    CI --> Agg
    Hops --> Agg
    Agg --> Dedup[Deduplication]
    Dedup --> Results[ContextPack]
```

### 3. Store (`pkg/store/`)

Per-session isolated storage with multiple indices for different data types.

**Data Structures:**
- Documents: Full text with metadata
- TextUnits: Chunked text with embeddings
- Entities: Named entities with types and descriptions
- Relationships: Typed edges between entities with weights
- Communities: Hierarchical entity clusters

**Index Types:**
- Primary: ID-based lookup (O(1))
- Secondary: External ID, title, document ID lookups
- Vector: HNSW indices for semantic search

### 4. Vector (`pkg/vector/`)

HNSW (Hierarchical Navigable Small World) implementation for approximate nearest neighbor search.

**Configuration:**
- `M`: Maximum connections per node (default: 16)
- `Ef`: Search candidate list size (default: 200)
- Dimension: Configurable (typically 1536 for OpenAI embeddings)

**Operations:**
- Add/Remove vectors with IDs
- K-nearest neighbor search by cosine similarity
- Serialization for snapshots

**Performance:**
- O(log N) search complexity
- SIMD-optimized distance calculations (AMD64)

### 5. Graph (`pkg/graph/`)

Community detection using the Leiden algorithm.

**Features:**
- Resolution-tunable clustering
- Hierarchical community levels
- Weighted edge support
- Incremental updates

```mermaid
flowchart TB
    E1[Entity 1] --- R1[knows] --- E2[Entity 2]
    E2 --- R2[works_at] --- E3[Entity 3]
    E1 --- R3[located_in] --- E4[Entity 4]
    E3 --- R4[located_in] --- E4

    subgraph C1[Community 1]
        E1
        E2
    end

    subgraph C2[Community 2]
        E3
        E4
    end
```

### 6. Backup (`pkg/backup/`)

Durability through WAL and snapshots.

**Write-Ahead Log:**
- Entry types: Insert, Update, Delete, Checkpoint
- 64MB segment rotation
- Configurable sync modes (every write, periodic, buffered)
- LSN (Log Sequence Number) tracking

**Snapshots:**
- Point-in-time compressed backups (gzip)
- Atomic write-to-temp pattern
- CRC32 checksums
- LSN correlation for recovery

**Recovery Flow:**
```mermaid
flowchart LR
    Start[Start Recovery] --> Load[Load Latest Snapshot]
    Load --> LSN[Get Snapshot LSN]
    LSN --> Replay[Replay WAL from LSN]
    Replay --> Verify[Verify Integrity]
    Verify --> Ready[Server Ready]
```

### 7. Client (`pkg/client/`)

Go client library with connection pooling.

**Features:**
- Configurable connection pool with health checks
- TLS support with certificate verification
- API key authentication
- Request ID correlation
- Automatic reconnection

## Data Flow

### Indexing Pipeline

```mermaid
flowchart TB
    subgraph Input
        Docs[Raw Documents]
    end

    subgraph Processing["Processing (Python SDK)"]
        Chunk[TokenChunker]
        Extract[LLM Extractor]
        Embed[Embedder]
    end

    subgraph Transport
        Client[GibRAM Client]
        Proto[Protobuf]
    end

    subgraph Server["Server Storage"]
        DocIdx[Document Index]
        TUIdx[TextUnit Index]
        EntIdx[Entity Index]
        RelIdx[Relationship Index]
        ComIdx[Community Index]
        VecIdx[Vector Indices]
    end

    Docs --> Chunk
    Chunk --> |Text Units| Extract
    Extract --> |Entities + Relationships| Embed
    Embed --> |Vectors| Client
    Client --> Proto
    Proto --> DocIdx
    Proto --> TUIdx
    Proto --> EntIdx
    Proto --> RelIdx
    TUIdx --> VecIdx
    EntIdx --> VecIdx
    ComIdx --> VecIdx
```

### Query Execution

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant E as Engine
    participant V as Vector Index
    participant G as Graph Store

    C->>S: Query(session_id, vector, k_hops)
    S->>E: Execute Query

    par Vector Search
        E->>V: Search TextUnit Index
        E->>V: Search Entity Index
        E->>V: Search Community Index
    end

    V-->>E: Similarity Results

    loop K-Hop Traversal
        E->>G: Get Relationships
        G-->>E: Connected Entities
    end

    E->>E: Aggregate & Deduplicate
    E-->>S: ContextPack
    S-->>C: Results + Stats
```

## Session Isolation

Each client operates in a fully isolated session with independent:
- Data storage (documents, entities, relationships)
- Vector indices
- ID generators
- TTL and idle timeout

```mermaid
flowchart TB
    subgraph Client1["Client A"]
        S1[Session abc-123]
    end

    subgraph Client2["Client B"]
        S2[Session xyz-789]
    end

    subgraph Engine
        SM[Session Manager]
    end

    subgraph Storage
        SS1[SessionStore<br/>abc-123]
        SS2[SessionStore<br/>xyz-789]
    end

    S1 --> SM
    S2 --> SM
    SM --> SS1
    SM --> SS2

    SS1 -.- |No Access| SS2
    SS2 -.- |No Access| SS1
```

## Security Model

### Authentication
- API key-based authentication
- Keys configured in YAML with permission levels

### Authorization (RBAC)
- **read**: Query, get operations
- **write**: Add, update, delete operations
- **admin**: Backup, restore, session management

### Rate Limiting
- Per-IP connection limits
- Global request rate limiting
- Configurable burst allowance

### TLS
- Auto-generated certificates for development
- Custom certificate support for production
- Optional skip-verify for testing

## Protocol Commands

130+ command types organized by category:

| Category | Range | Examples |
|----------|-------|----------|
| Basic | 1-9 | PING, INFO, HEALTH |
| Document | 10-19 | ADD_DOC, GET_DOC, DEL_DOC |
| TextUnit | 20-29 | ADD_TU, GET_TU, LINK_TU |
| Entity | 30-39 | ADD_ENT, GET_ENT, UPDATE_ENT |
| Relationship | 40-49 | ADD_REL, GET_REL, DEL_REL |
| Community | 50-59 | ADD_COM, COMPUTE_LEIDEN |
| Query | 60-69 | QUERY, EXPLAIN |
| Session | 70-79 | LIST, DELETE, SET_TTL |
| Bulk | 80-91 | MSET_*, MGET_* |
| Pipeline | 100-101 | PIPELINE_EXEC |
| Backup | 110-119 | SAVE, RESTORE, WAL_* |
| Auth | 120-121 | AUTH |

## Directory Structure

```
gibram/
├── cmd/
│   ├── server/main.go          # Server entry point
│   └── cli/main.go             # Interactive CLI
├── pkg/
│   ├── engine/                 # Query execution
│   ├── store/                  # Session storage
│   ├── vector/                 # HNSW implementation
│   ├── graph/                  # Leiden algorithm
│   ├── server/                 # TCP server
│   ├── client/                 # Go client
│   ├── backup/                 # WAL & snapshots
│   ├── types/                  # Data structures
│   ├── config/                 # Configuration
│   ├── codec/                  # Protocol codec
│   ├── logging/                # Structured logging
│   ├── metrics/                # Performance metrics
│   ├── memory/                 # Memory tracking
│   ├── resilience/             # Circuit breaker
│   ├── pool/                   # Object pooling
│   ├── simd/                   # SIMD optimizations
│   └── shutdown/               # Graceful shutdown
├── proto/
│   ├── gibram.proto            # Protocol definition
│   └── gibrampb/               # Generated code
├── sdk/
│   └── python/                 # Python SDK
└── examples/                   # Usage examples
```

## Configuration

```yaml
server:
  addr: ":6161"
  data_dir: "./data"
  vector_dim: 1536

tls:
  cert_file: ""
  key_file: ""
  auto_cert: true

auth:
  keys:
    - id: "admin"
      key: "secret"
      permissions: ["admin"]

security:
  max_frame_size: 4194304  # 4MB
  rate_limit: 1000         # req/sec
  rate_burst: 100
  idle_timeout: 300s
```

## Performance Characteristics

| Aspect | Characteristic |
|--------|----------------|
| Vector Search | O(log N) via HNSW |
| Graph Traversal | O(edges × k-hops) |
| Session Lookup | O(1) map access |
| Memory Model | In-memory only (ephemeral without persistence) |
| Concurrency | Session-level isolation, no cross-session locks |
| Max Sessions | 10,000 (configurable DoS protection) |
| Query Log | 10,000 entries LRU |

## Graceful Shutdown

Shutdown follows a prioritized sequence:

```mermaid
flowchart TB
    Signal[SIGTERM/SIGINT] --> Stop[Stop Accepting Connections]
    Stop --> Drain[Drain Active Requests]
    Drain --> Sessions[Clean Up Sessions]
    Sessions --> Metrics[Snapshot Metrics]
    Metrics --> WAL[Close WAL]
    WAL --> Done[Exit]
```

Timeout: 30 seconds maximum.

## External Dependencies

### Go
- `google.golang.org/protobuf` - Protocol Buffers
- `golang.org/x/crypto` - TLS, bcrypt
- `golang.org/x/time/rate` - Rate limiting
- `gopkg.in/yaml.v3` - Configuration
- `github.com/cespare/xxhash/v2` - Fast checksums

### Python SDK
- `openai` - LLM and embeddings
- `grpcio-tools` - Protocol compilation
- `click` - CLI interface

## Integration Points

| Integration | Purpose |
|-------------|---------|
| OpenAI API | Entity extraction, embedding generation |
| TLS Certificates | Transport security |
| File System | WAL segments, snapshots |
| Docker | Container deployment |
