# GraphifyRust API Catalog

> Complete API reference for GraphifyRust, extracted from `graphify-rust` repository
> Last updated: 2026-08-19
> Purpose: Building `graphify-sdk-php` - a PHP SDK for Graphify MCP

---

## Table of Contents

1. [MCP Tools (graphify)](#mcp-tools)
2. [Core Data Types (graphify-core)](#core-data-types)
3. [Plugin API (graphify-core)](#plugin-api)
4. [Plugin Memory (graphify-core)](#plugin-memory)
5. [TOON Format](#toon-format)
6. [Communication Protocol](#communication-protocol)

---

## MCP Tools (graphify)

### Core Graph Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_graph_summary` | High-level topology metrics | - | `{"total_nodes": N, "total_edges": N, "languages": [...]}` |
| `graphify_graph_query` | BFS traversal by question | `{"question": string}` | `{"nodes": [...], "edges": [...]}` |
| `graphify_graph_query_node` | Query nodes by ID | `{"node_id": string, "depth": int?}` | `{"nodes": [...], "edges": [...]}` |
| `graphify_graph_trace_path` | Shortest path between nodes | `{"from": string, "to": string}` | `{"path": [node_ids]}` |
| `graphify_graph_reindex` | Reindex a file | `{"file_path": string}` | `{"status": "success", "total_nodes": N, "total_edges": N}` |

### Memory Query Tool

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_memory_query` | Semantic memory search | `{"workspace_key": string, "query": string, "limit": int?}` | `{"status": "found\|not_found\|error", "nodes": [...]}` |

### Relay/Handoff Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_relay_init` | Initialize relay session | `{"project_context": string, "kind": string?}` | Relay init confirmation |
| `graphify_relay_save` | Save session state | `{"repo": string, "role": string, "phase": string, "volatile": string, "conf": float?, "next": string, "debt": string, "kind": string}` | Saved state summary |
| `graphify_relay_close` | Close relay session | `{"repo": string, "next": string}` | Consistency status |
| `graphify_relay_switch` | Switch to another repo | `{"repo": string, "kind": string?}` | Switch confirmation |
| `graphify_relay_resume` | Resume relay session | `{"repo": string, "kind": string?}` | Resumed state |
| `graphify_relay_status` | Show relay summary | - | Relay status object |
| `graphify_relay_add` | Ingest TODO/handoff doc | `{"file": string, "repo": string}` | Ingested count |

### OpenDoc Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_opendoc_index` | Index spec blocks | `{"doc_paths": [string]}` | `[indexed] N link rows` |
| `graphify_opendoc_get_context` | Get spec blocks for symbol | `{"symbol": string}` | Spec block rows or "no coverage" |
| `graphify_opendoc_audit_drift` | Audit doc-side drift | - | Drift items list |

### Review Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_review_ingest` | Import CRG payload | `{"payload": string}` | `[bound] N, [orphan] N` |
| `graphify_review_get_context` | Query unresolved reviews | `{"node": string}` | Review rows or "no reviews" |
| `graphify_review_resolve` | Mark review resolved | `{"review_id": string, "reason": string}` | Resolved status |
| `graphify_review_search_crg` | Search CRG for changed functions | `{"base": string}` | Changed function list |

### Telemetry Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_telemetry_ingest` | Import telemetry metrics | `{"source": "file\|draco-mcp", "path_or_draco_params": string}` | Metrics summary |
| `graphify_telemetry_get_context` | Query telemetry bindings | `{"node": string, "include_impact_radius": bool}` | Telemetry bindings list |

### Coverage Tools

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_coverage_ingest` | Import coverage data | `{"format": "lcov\|json", "data": string}` | Coverage summary |
| `graphify_coverage_get_context` | Query coverage for node | `{"node": string}` | Coverage percentage |
| `graphify_coverage_blindspots` | List low-coverage nodes | - | Blindspot list |

### Plugin Gateway

| Tool Name | Description | Parameters | Returns |
|-----------|-------------|------------|---------|
| `graphify_plugin_notify` | Broadcast graph update | `{"kind": "indexed\|extracted\|manual"}` | Broadcast status |

---

## Core Data Types (graphify-core)

### Node

```rust
pub struct Node {
    pub id: NodeId,           // String
    pub label: String,        // Symbol name (e.g., "MyFunction")
    pub file_type: FileType,  // code\|document\|paper\|image\|rationale\|concept
    pub kind: String,         // module\|struct\|class\|function\|method\|trait\|interface\|test\|etc.
    pub language: String,     // Language name (e.g., "rust", "python")
    pub source_file: String,  // Absolute or relative path
    pub start_line: usize,    // Line number (0-indexed)
    pub end_line: usize,      // Line number (0-indexed)
    pub doc_comment: Option<String>,  // Optional docstring
    pub description: Option<String>,  // Optional description
    pub metadata: Option<serde_json::Map<String, serde_json::Value>> // Optional custom metadata
}
```

### Edge

```rust
pub struct Edge {
    pub source: NodeId,       // Source node ID
    pub target: NodeId,       // Target node ID
    pub relation: String,     // Relationship type (e.g., "calls", "imports", "contains")
    pub source_file: String,  // Source file path
    pub confidence: String,   // Confidence level (default: "EXTRACTED")
    pub source_location: String, // Location string (e.g., "src/lib.rs:12")
    pub description: Option<String> // Optional description
}
```

### NodeId

```rust
pub struct NodeId(pub String);
// Example: "./graphify-core/src/types.rs:function:MyFunction"
```

### FileType

```rust
pub enum FileType {
    Code,
    Document,
    Paper,
    Image,
    Rationale,
    Concept,
}
```

### GraphMetadata

```rust
pub struct GraphMetadata {
    pub version: String,              // Format version (e.g., "1.0.0")
    pub generated_at: String,         // Timestamp string
    pub total_nodes: usize,           // Total node count
    pub total_edges: usize,           // Total edge count
    pub languages: Vec<String>,       // Languages present
    pub input_tokens: usize,          // LLM input token count
    pub output_tokens: usize,         // LLM output token count
    pub plugin_data: BTreeMap<String, serde_json::Value> // Plugin metadata (optional)
}
```

### GraphOutput

```rust
pub struct GraphOutput {
    pub nodes: Vec<Node>,
    pub edges: Vec<Edge>,
    pub metadata: GraphMetadata,
}
```

### WorkspaceContext

```rust
pub struct WorkspaceContext {
    pub workspace_key: String,        // e.g., "w-9f8a2b1c-8e7d-4c3b"
    pub workspace_name: String,       // e.g., "graphify-monorepo"
    pub root_path: String,            // Absolute filesystem root
    pub timestamp: i64,               // Unix epoch seconds
}
```

### MemoryQueryCriteria

```rust
pub struct MemoryQueryCriteria {
    pub target_symbols: Vec<String>,  // Symbols to search for
    pub domain_categories: Vec<String>, // Domain categories (e.g., "mcp", "review")
    pub search_terms: Vec<String>,   // Natural language search terms
}
```

### HandoffPayload

```rust
pub struct HandoffPayload {
    pub schema_version: u32,              // Always 1
    pub task_goal: String,                // Task description
    pub pinned_node_ids: Vec<String>,    // Important node IDs
    pub focused_subgraph_toon: String,    // Subgraph in TOON format
    pub reconstructable_query_metadata: MemoryQueryCriteria // Query conditions
}
```

---

## Plugin API (graphify-core)

### GraphifyPlugin Trait

```rust
pub trait GraphifyPlugin {
    /// Returns the plugin's unique identifier
    fn get_id(&self) -> &str;
    
    /// Binds the plugin to a workspace context
    fn bind(&mut self, ctx: WorkspaceContext);
    
    /// Returns the workspace UUID (empty if not bound)
    fn get_workspace_key(&self) -> &str;
    
    /// Synchronizes TOON payload and returns processed output
    /// - Some(payload): passive sync (consume external TOON)
    /// - None: proactive sync (produce output from context)
    fn sync_toon(&mut self, opt_toon: Option<Vec<u8>>) -> Vec<u8>;
    
    /// Receives graph update notification
    fn on_graph_updated(&mut self, _event: &GraphUpdateEvent) {}
    
    /// Sets host-injected notify callback (v1.1, optional)
    fn set_notify_callback(&mut self, _cb: Option<NotifyCallback>) {}
    
    /// Performs health check
    fn on_health_check(&self) -> bool { false }
}
```

### GraphUpdateEvent

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct GraphUpdateEvent {
    pub workspace_key: String,
    pub modified_nodes: Vec<NodeId>,
    pub event: GraphUpdateKind,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[non_exhaustive]
pub enum GraphUpdateKind {
    Indexed,      // graphify index completed
    Extracted,    // graphify extract completed
    Manual,       // Explicit trigger
}
```

### NotifyCallback

```rust
pub type NotifyCallback = Box<dyn Fn(serde_json::Value) + Send + Sync>;
```

---

## Plugin Memory (graphify-core)

### PluginMemoryEnvelope

```rust
pub struct PluginMemoryEnvelope<T> {
    pub format_version: u32,                // Always 1
    pub workspace_key: String,
    pub plugin_id: String,
    pub record_id: String,
    pub record_kind: String,
    pub created_at: i64,                    // Unix timestamp
    pub source_refs: Vec<String>,           // Source file references
    pub payload: T,                         // Plugin-specific payload
}
```

### OpenDocPayload

```rust
pub struct OpenDocPayload {
    pub schema_version: u32,         // Always 1
    pub doc_identity: String,        // Document identity
    pub doc_version: String,         // Document version
    pub chunk_index: usize,          // Chunk index
    pub raw_content: String,         // Chunk content
    pub linked_symbols: Vec<String>, // Linked symbol IDs
}
```

### ReviewPayload

```rust
pub struct ReviewPayload {
    pub schema_version: u32,         // Always 1
    pub review_id: String,
    pub git_commit_sha: Option<String>,
    pub affected_symbols: Vec<String>,
    pub finding_severity: String,    // e.g., "high", "medium", "low"
    pub resolution_status: String,   // e.g., "open", "resolved"
    pub review_comment: String,
}
```

### HandoffSnapshot

```rust
pub struct HandoffSnapshot {
    pub snapshot_id: String,
    pub session_id: String,
    pub workspace_key: String,
    pub created_at: i64,
    pub expires_at: i64,             // TTL expiry timestamp
    pub payload: HandoffPayload,
}
```

---

## TOON Format

### Structure

```
metadata:
  version: "1.0.0"
  generated_at: "2026-08-19"
  total_nodes: 608
  total_edges: 1793
  languages[1]: rust
  input_tokens: 0
  output_tokens: 0
  plugin_data: {"opendoc": {...}, "review": {...}}

nodes[608,]{id,label,file_type,kind,language,source_file,start_line,end_line,doc_comment,description,metadata}:
  "file:function:name","label",type,kind,lang,path,start,end,doc,desc,meta
  ...

edges[1793,]{source,targets,relation,source_file,confidence,source_location,description}:
  "src1.rs|src2.rs|src3.rs",relation,src,path,conf,location,desc
  ...
```

### Key Features

- **Header-Declared Columns**: Column names defined once
- **Virtual Hyperedge Aggregation**: Multiple edges with same source/relation aggregated using `|` delimiter
- **Escaped Strings**: Double quotes escaped as `\"`, newlines as `\n`
- **Optional Fields**: `doc_comment`, `description`, `metadata` may be `null`

### Example Node Line

```
"src/lib.rs:function:extract","extract",code,function,rust,"src/extract.rs",10,51,"fn extract doc", "Function description",null
```

### Example Edge Line (Aggregated)

```
  "src/app.rs","src/db.rs|src/cache.rs|src/config.rs","imports","src/app.rs","EXTRACTED","src/app.rs:5",null
```

---

## Communication Protocol

### MCP Protocol (Stdio/JSON-RPC)

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "graphify_graph_summary",
    "arguments": {}
  }
}

// Response (Success)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{"type": "text", "text": "summary data"}]
  }
}

// Response (Error)
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Missing required argument"
  }
}
```

### Workspace Key Derivation

```rust
// Derives a stable hash from the workspace root path
pub fn derive_workspace_key<P: AsRef<Path>>(root_path: P) -> String {
    let canonical = root_path.as_ref().canonicalize().unwrap_or_else(|_| root_path.as_ref().to_path_buf());
    let mut hasher = DefaultHasher::new();
    canonical.hash(&mut hasher);
    format!("{:x}", hasher.finish())  // Hex-encoded hash
}
```

---

## SDK Development Notes for PHP

### 1. Required Dependencies

```json
{
  "require": {
    "php": ">=8.0",
    "php-mcp": "^1.0",           // MCP client library
    "guzzlehttp/guzzle": "^7.0"  // HTTP client (if using JSON-RPC HTTP)
  }
}
```

### 2. Stdio MCP Connection

```php
class GraphifyMcpClient {
    private $process;
    
    public function __construct(string $path = 'graphify') {
        $this->process = proc_open(
            $path,
            ['pipe' => ['r', 'w', 'w']],
            $descriptors
        );
        $this->stdin = stream_socket_accept($descriptors[1]);
    }
    
    public function sendRequest(string $method, array $params): array {
        $request = json_encode([
            'jsonrpc' => '2.0',
            'id' => uniqid(),
            'method' => $method,
            'params' => $params
        ]);
        
        fwrite($this->stdin, $request . "\n");
        $response = stream_get_line($this->stdout, 8192);
        
        return json_decode($response, true);
    }
}
```

### 3. Plugin Registration

Plugins should be registered in `graphify.toml`:

```toml
[plugins.opendoc]
command = "php"
args = ["-m", "graphify_opendoc"]
cwd = "./plugins/opendoc"

[plugins.review]
command = "php"
args = ["-m", "graphify_review"]
cwd = "./plugins/review"
```

---

## References

- `graphify-core/src/types.rs` - Data type definitions
- `graphify-core/src/plugin.rs` - Plugin API trait
- `graphify-core/src/toon.rs` - TOON serialization
- `graphify-core/src/plugin_memory.rs` - Memory contracts
- `graphify/src/main.rs` - MCP server implementation
- `graphify/src/memory_query.rs` - Memory query service
- `openspec/specs/toon-plugin-data/spec.md` - Plugin data format
- `openspec/specs/sync-toon-packet/spec.md` - sync_toon contract
- `docs/plugin_system.md` - Plugin architecture
- `docs/architecture-memory-plugin.md` - Memory boundary

---

*Generated for graphify-sdk-php development*
