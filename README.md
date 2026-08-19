# Graphify SDKs

Official SDK family for [Graphify](https://github.com/cawa0505/graphify) — access
knowledge graph capabilities over the MCP (Model Context Protocol) via Stdio/JSON-RPC.

## Available SDKs

| Language | Package | Repo | Status |
|----------|---------|------|--------|
| 🟢 **TypeScript** | `graphify-sdk` | [cawa0505/graphify-sdk-js](https://github.com/cawa0505/graphify-sdk-js) | ✅ Active |
| 🐹 **Go** | `github.com/cawa0505/graphify-sdk-go` | [cawa0505/graphify-sdk-golang](https://github.com/cawa0505/graphify-sdk-golang) | ✅ Active |
| 🐘 **PHP** | `graphify/sdk-php` | [cawa0505/graphify-sdk-php](https://github.com/cawa0505/graphify-sdk-php) | ✅ Active |
| 🐍 **Python** | `graphify-sdk-python` | [cawa0505/graphify-sdk-python](https://github.com/cawa0505/graphify-sdk-python) | ✅ Active |

## Architecture

All SDKs follow the same architecture — zero-dependency process wrappers that
communicate with the Graphify Core binary (`graphify`) over Stdio/JSON-RPC:

```
Your App → SDK Client → Transport (Stdio/JSON-RPC) → graphify (Rust)
```

### Key Design Principles

- **Zero external dependencies**: each SDK uses only its language's standard library
- **Bi-directional CLI interop**: all communication via subprocess Stdio/JSON-RPC
- **Lazy initialization**: the subprocess spawns on the first request, not at construction
- **Auto workspace key**: derived from project path (crc32 hash, consistent across SDKs)
- **Single native binary**: all SDKs compile to standalone executables for deployment

## SDK Capabilities

All SDKs expose the same tool set (24+ methods) covering:

| Domain | Tools |
|--------|-------|
| **Core Graph** | GraphSummary, QueryGraph, QueryNode, TracePath, ReindexFile |
| **Memory** | MemoryQuery |
| **Relay/Handoff** | RelayInit, RelaySave, RelayClose, RelaySwitch, RelayResume, RelayStatus, RelayAdd |
| **OpenDoc** | OpenDocIndex, OpenDocGetContext, OpenDocAuditDrift |
| **Review** | ReviewIngest, ReviewGetContext, ReviewResolve, ReviewSearchCrg |
| **Telemetry** | TelemetryIngest, TelemetryGetContext |
| **Coverage** | CoverageIngest, CoverageGetContext, CoverageBlindspots |
| **Plugin** | PluginNotify |

## Binary Compilation

Each SDK can be compiled to a standalone native binary for deployment:

| Language | Compilation | Startup | Size |
|----------|------------|---------|------|
| TypeScript | `tsc` + Bun Compile / SEA | < 10ms (Bun) | ~30-40MB |
| Go | `CGO_ENABLED=0 go build -ldflags="-s -w"` | < 10ms | ~5-15MB |
| PHP | static-php-cli + phpmicro / FrankenPHP | 5-15ms | ~15-25MB |
| Python | Nuitka `--standalone --onefile` | 10-30ms | ~20-40MB |

## Documentation

- [Ecosystem Architecture](docs/architecture/ecosystem-architecture.md) — bidirectional SDK/plugin design, WASM vs IPC comparison
- [Architecture & Distribution Spec](docs/architecture/graphify-2.0.0-architecture-spec.md) — GA compilation matrix, single-binary deployment
- [Node.js/TypeScript Ecosystem Spec](docs/architecture/graphify-2.0.0-nodejs-spec.md) — Bun/SEA compilation, bidirectional CLI interop
- [Rust API Reference](docs/api/graphify-rust-api.md) — full tool catalog, data types, protocol details

## Related Repositories

- [GraphifySDK](https://github.com/cawa0505/graphify) — Graphify Core and SDK development workspace
- [graphify](https://github.com/cawa0505/graphify) — Graphify Rust binary (Core Engine)

## License

Each SDK is licensed under MIT. See individual repos for details.