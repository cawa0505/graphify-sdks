# Graphify 生態系架構 — 雙向整合設計

> 釐清 Graphify SDK 生態的整體架構方向，從單向查詢走向雙向整合。
> 2026-08-19

---

## 1. 現狀：單向架構

```
External Code ──── MCP (JSON-RPC over stdio) ──── Graphify Core
(PHP/Python SDK)                                    (Rust)
```

SDK 的角色僅是 **Client**：透過 MCP 協議向 Graphify 發起查詢，取得知識圖譜資料。

- SDK 能做的：`query_graph`、`get_context`、`ingest data`
- SDK 不能做的：被 Graphify 呼叫、註冊為 Graphify 的資料來源

---

## 2. 目標：雙向架構

```
Plugin (PHP/Node/Python) ←── SDK ──→ Graphify Core (Rust)
  寫業務邏輯                橋接          只管圖譜 + MCP
```

SDK 同時扮演 **兩種角色**：

| 角色 | 方向 | 說明 | 現有例子 |
|------|------|------|---------|
| **Client SDK** | SDK → Core | 外部程式查詢 Graphify | sdk-php, sdk-python |
| **Plugin SDK** | Core → SDK | Graphify 載入外部工具當 plugin | 設計中 |

核心關鍵：**SDK = Plugin SDK**。同一包 API 同時服務兩個方向：

- 當 Client 用：`$client->queryGraph(...)` — 查圖譜
- 當 Plugin 用：`$plugin->registerTool('analyze_schema', $handler)` — 註冊工具給 Graphify

### 2.1 為什麼這很重要

Graphify Core（Rust）保持極致乾淨：

- 不須懂 PHP 怎麼跑、Laravel Migration 長怎樣
- 不須懂 Node.js Stream、Python asyncio
- 只負責管圖譜（Graph）與 MCP 協議

Plugin 開發者用熟悉的語言寫業務邏輯：

- PHP 開發者寫 Laravel DB 關聯解析器
- Python 開發者寫 ML 模型評分器
- Node.js 開發者寫 GraphQL schema 分析器

通訊方式（輕量化，無綁定）：

- **JSON-RPC over stdio**：任何語言原生支援
- **WASM 模組**：輕量沙箱，適合 Rust/C/Zig

### 2.2 運作流程

```
User MCP Request: "trace:column users legacy_status"
       │
       ▼
Graphify Core (Rust)
  └── 查 AST graph，找出相關節點
  └── 轉發給 php-db-analyzer plugin（IPC subprocess）
       │
       ▼
PHP Plugin（透過 SDK）
  └── 收到 AST node + context
  └── 執行 Laravel Migration 解析邏輯
  └── 回傳 JSON 結果
       │
       ▼
Graphify Core
  └── 合併結果到 MCP response
  └── 回傳給 User
```

核心不須知道 Laravel 是什麼。它只負責：收到查詢 → 找出相關節點 → 轉給對應 plugin → 回傳結果。

---

## 3. 兩種 Plugin Host 方案

### A. WASM Plugin Runtime

```
Graphify Core
  └── wasmtime / wasmer
        ├── plugin_a.wasm   (Rust → WASM)
        ├── plugin_b.wasm   (Go  → WASM, via tinygo)
        └── plugin_c.wasm   (C   → WASM, via emcc)
```

| 面向 | 評價 |
|------|------|
| **安全** | 沙箱天生隔離，無 fs/net/subprocess（除非 WASI 明確開） |
| **效能** | 接近原生，無序列化開銷（WASM memory 可直接共享） |
| **跨平台** | 單一 .wasm 二進位，無平台依賴 |
| **開發體驗** | 需編譯成 WASM，非所有語言都支援（Python/PHP 困難） |
| **能力限制** | 無 POSIX API、無 thread、無 GPU（除非 WASIX 擴充） |
| **適合** | 輕量計算：schema 轉換、資料驗證、規則引擎 |

### B. IPC Plugin Host

```
Graphify Core
  └── Plugin Host (subprocess manager)
        ├── php plugin   (proc_open → php plugin.php)
        ├── python plugin (proc_open → python plugin.py)
        └── go plugin     (proc_open → ./go-plugin)
```

每個 plugin 是一個獨立行程，透過 stdin/stdout JSON-RPC 通訊（同 MCP 協議）。

| 面向 | 評價 |
|------|------|
| **安全** | 行程級隔離，作業系統負責 |
| **能力** | 完整語言 runtime：fs、net、subprocess、GPU |
| **語言支援** | 任何語言，無編譯限制 |
| **開發體驗** | 寫一個 CLI 程式即可，零門檻 |
| **成本** | 每個 plugin 一個行程，記憶體開銷較高 |
| **序列化** | JSON-RPC 每次呼叫有序列化成本 |
| **適合** | 重度工具：AST parser、DB analyzer、ML inference |

### 對比總表

| | WASM Plugin | IPC Plugin |
|---|---|---|
| 語言支援 | Rust/Go/C/Zig → WASM | 任何語言 |
| 開發門檻 | 中（需 WASM toolchain） | 低（寫 CLI 即可） |
| 能力完整度 | 受限（沙箱） | 完整 |
| 啟動速度 | μs 級 | ms 級 |
| 資源隔離 | 記憶體級 | 行程級 |
| 序列化開銷 | 無（shared memory） | 有（JSON-RPC） |
| 部署 | 單一 .wasm | 需 runtime + dependencies |

---

## 4. 建議策略：兩層共存

不互斥，分層使用：

```
Graphify Core
  ├── WASM Runtime (wasmtime)
  │     ├── graphql-to-schema.wasm     # 輕量轉換
  │     └── validation-rules.wasm       # 規則引擎
  │
  └── IPC Plugin Host (subprocess)
        ├── php-db-analyzer             # 重度資料庫分析
        ├── python-ml-scorer            # ML 模型推論
        └── go-network-scanner          # 網路操作
```

### 選擇原則

| 工具類型 | 方案 |
|---------|------|
| 純計算、無 I/O | WASM |
| 需要檔案系統 | IPC |
| 需要網路請求 | IPC |
| 需要 subprocess | IPC |
| 需要 GPU/ML | IPC |
| 需要跨平台輕量部署 | WASM |
| 語言是 Python/PHP/Go | IPC |
| 語言是 Rust/C/Zig | 兩者皆可 |

---

## 5. SDK = Plugin SDK：同一包 API，兩個面向

### 5.1 核心概念

**不是 Client SDK 和 Plugin SDK 兩套不同的東西。** 同一語言的 SDK 同時提供兩個面向：

```
graphify-sdk-php/
├── Client/          # Outbound：查詢 Graphify
│   └── GraphifyClient.php
│
└── Plugin/          # Inbound：被 Graphify 載入
    ├── PluginInterface.php     # 註冊 tools、處理 requests
    └── PluginHost.php          # 負責 IPC stdio 通訊
```

開發者只需要 `composer require graphify/sdk-php`，就能：

- 寫一般 PHP 程式時：`$client->queryGraph(...)`
- 寫 Graphify plugin 時：`$plugin->registerTool(...)` → 同一個 composer package

### 5.2 Plugin 運作方式

```
Graphify Core (Rust)
  │
  ├── 啟動時掃描 plugin 設定檔
  │     e.g. graphify.plugins.json:
  │     { "php-db-analyzer": { "command": "php analyzer.php", "language": "php" } }
  │
  ├── IPC Plugin Host: spawn subprocess
  │     proc_open("php analyzer.php", [stdin, stdout, stderr])
  │
  ├── Plugin 啟動後回報能力:
  │     {"jsonrpc":"2.0","method":"initialize","params":{"tools":[
  │       {"name":"analyze_schema","inputSchema":{...}},
  │       {"name":"trace_column","inputSchema":{...}}
  │     ]}}
  │
  └── 收到 MCP request 時轉發給 plugin
        {"jsonrpc":"2.0","method":"tools/call","params":{"name":"analyze_schema","arguments":{...}}}
        ──→ PHP Plugin 處理 → 回 JSON → Core 合併進 response
```

### 5.3 Plugin 開發者體驗

```php
<?php
// analyzer.php — 一個 PHP Graphify Plugin
use Graphify\Sdk\Plugin\PluginInterface;
use Graphify\Sdk\Plugin\PluginHost;

$host = new PluginHost();

$host->registerTool('analyze_schema', [
    'description' => 'Analyze Laravel database schema',
    'inputSchema' => [
        'type' => 'object',
        'properties' => [
            'migration_path' => ['type' => 'string']
        ]
    ]
], function (array $args): array {
    // 用 PHP 熟悉的 Laravel Migration 解析邏輯
    return ['columns' => [...], 'relations' => [...]];
});

$host->run(); // 啟動 JSON-RPC stdio listener
```

不用 Rust。不用 WASM toolchain。`composer install` 就開工。

### 5.4 Graphify Core 端負責

```
Graphify Core 的 Plugin Host 元件：
  1. 讀設定 → 知道有哪些 plugin、用什麼指令啟動
  2. spawn → 啟動 subprocess（IPC）或載入 .wasm（WASM）
  3. initialize → 跟 plugin 握手，取得 tool list
  4. route → 收到相關 MCP request 時轉發給 plugin
  5. lifecycle → crash restart、idle timeout、graceful shutdown
```

核心不須懂 plugin 用什麼語言寫的。**Plugin Host 只看協議，不看語言。**

---

## 6. 問題與待討論

- Plugin Host 協議是否直接沿用 MCP JSON-RPC（initialize / tools/list / tools/call）？還是定義一套更輕量的子集？
- IPC Plugin 的生命週期管理：lazy spawn（第一次呼叫才啟動）、keep-alive pool、crash restart with backoff？
- Plugin 如何宣告自己的 capabilities？透過 `initialize` response 的 `tools` 陣列就夠了嗎？還是需要 resource / prompt 等 MCP 進階功能？
- 安全邊界：IPC plugin 是 full trust 還是 sandbox（seccomp / landlock / container）？
- 現有 graphify-plugin-*（handoff/opendoc/review/coverage/telemetry）是保持原生 Rust plugin，還是也可用 SDK 重寫？**建議：保持 Rust，因為它們是核心基礎設施，不需要跨語言的好處。**
- Plugin 註冊方式：config file（graphify.plugins.json）還是 Graphify Core 啟動時掃描某個目錄？還是兩者都支援？

---

## 7. 建議下一步

1. 確定 Plugin Host 協議 = MCP JSON-RPC（零新協議）
2. 在 Graphify Core 中實作 **IPC Plugin Host**（優先，因為最實用）
3. 定義 Plugin SDK 的介面規格（tool registration, capability discovery）
4. 用一個非 Rust plugin 做 PoC（例如 Python 寫的 git-blame plugin）
5. 視需求加入 WASM Runtime（當有純計算場景時）

---

*本文件為方向探討，非最終規格。*
