# Graphify 2.0.0 GA Architecture & Distribution Spec

> 跨語言 SDK、Plugin 與二進位編譯全貌規格書
> 日期：2026-08-19

## 1. 系統核心哲學：雙向 CLI 互查 (Bi-directional CLI Interop)

Graphify 放棄了複雜且需常駐背景進程（Daemon）的 RPC 綁定，全面採用
「雙向 CLI 互查」與「純文字/JSON 標準流（Stdio）」作為跨語言與跨工具鏈溝通的神經總線。

```
Plaintext                               Graphify Core (Rust 2.0.0 GA)
                               └── Standard NodeId & .toon Engine
                                                ▲
                                                │ 雙向 CLI 互查 (Process I/O + JSON)
         ┌──────────────────────────────────────┼──────────────────────────────────────┐
         │                                      │                                      │
         ▼                                      ▼                                      ▼
graphify-sdk-php                     graphify-sdk-golang                   Nuitka Compiled Python
(Laravel / Pest / Artisan)             (Cobra / Gin / Go Test)                (code-review-graph / Draco)
```

### 核心原則

- **0ms 實體對齊**：所有語言端的 Data Points 最終統一升維並綁定至標準 Canonical NodeId（`{file_path}:{kind}:{name}`）。
- **無環境依賴發布**：所有工具鏈與 SDK 驅動的產物均導向「單一靜態二進位檔（Single Native Binary）」，達成零環境開銷與源碼保護。
- **無鎖高並發 Process I/O**：語言端 SDK 本質為零開銷 Process Wrapper，透過 OS 級別 Subprocess 進行瞬發（10~30ms）調用。

## 2. 跨語言 SDK 與二進位編譯矩陣 (Compilation Matrix)

| 語言 / 生態 | SDK 專案倉庫 | 最佳二進位編譯方案 | 平均啟動延遲 | 框架相容性 | 源碼保護力 |
|-------------|-------------|-------------------|-------------|-----------|-----------|
| Rust (Core) | Core 本體 | `cargo build --release` (Native) | < 5ms | 100% 原生 | 100% 機器碼 |
| PHP | cawa0505/graphify-sdk-php | static-php-cli (spc) + phpmicro / FrankenPHP | 5~15ms | 100% (含 Laravel/Pest) | 100% (無 Plaintext 落盤) |
| Go | cawa0505/graphify-sdk-golang | `go build -ldflags="-s -w"` (Static) | < 10ms | 100% 原生 | 100% 機器碼 |
| Python | Python Tools / Bridge | Nuitka (C Transpiler / AOT) | 10~30ms | 100% | 100% (轉譯 C/C++ 機器碼) |

## 3. 各語言 SDK 與編譯架構技術拆解

### 🦀 Rust (Core Engine)

**角色**：圖譜與 Context 彙整樞紐（AST Extraction, Petgraph BFS, .toon Compression, graphify.db WAL 託管）。

**編譯產物**：`graphify` 靜態原生二進位檔（Single Binary）。

**溝通介面**：提供標準 CLI 命令（如 `graphify coverage`, `graphify query`）供外部 SDK 與 Plugin 呼叫。

### 🐘 PHP (graphify-sdk-php)

**角色**：將 PHP / Laravel 領域資料（Pest / PHPUnit Cobertura 報告、Eloquent ORM Schema）導向 Graphify，或從 Laravel 向 Graphify 反向查詢 AST 衝擊半徑。

**二進位編譯技術**：
- **微型靜態 SAPI (static-php-cli + phpmicro)**：將 PHP 8.x Zend Engine 靜態編譯，與專案 Phar 打包成單一靜態執行檔。
- **FrankenPHP Embed**：將 PHP 專案與 Web Daemon 嵌入 Go 二進位檔。

**優勢**：啟動延遲僅 10ms，且目標主機硬碟上完全不需要放任何 .php 原始碼，100% 原始碼保密。

### 🐹 Go (graphify-sdk-golang)

**角色**：提供雲原生工具鏈、Cobra CLI 與 Go 服務（Gin/Echo）對接 Graphify 的 Goroutine-Safe Process Wrapper。

**二進位編譯技術**：`CGO_ENABLED=0 go build -ldflags="-s -w"`。

**優勢**：產出極小、無動態庫依賴（Distroless 友好）的靜態二進位檔，適合在高併發場景下背景觸發 Graphify 雙向查詢。

### 🐍 Python (Tools / Bridges, e.g., code-review-graph, Draco)

**角色**：外部動態分析、Review 數據與 AI Agent 輔助工具。

**二進位編譯技術**：Nuitka AOT Compiler (`nuitka --standalone --onefile`)。

**不採用 PyInstaller**：避免單一檔運行時解壓縮導致的 300ms~1s 啟動延遲。

**Nuitka 原理**：將 Python 程式碼轉譯為 C/C++ 代碼後呼叫 GCC/Clang 編譯成原生機器碼。

**優勢**：將 Python CLI 啟動延遲壓低至 10~30ms，達成媲美 Rust/Go 的極速雙向互查體驗。

## 4. Pure Bridge 外掛生態矩陣 (Plugins)

Graphify GA 堅持「Core 算拓撲，Plugin 做 Binding」的 Pure Bridge 架構，目前四大 Plugin 狀態如下：

| Plugin | 功能 |
|--------|------|
| 🤝 **graphify-plugin-handoff** | 心智快照 / Session 焦點 → NodeId 綁定 |
| 📄 **graphify-plugin-opendoc** | 業務規範 / OpenAPI Spec → NodeId 綁定 |
| 🔍 **graphify-plugin-review** | code-review-graph 評語與歷史 Drift 防禦 → NodeId 綁定 |
| 🧪 **graphify-plugin-test-coverage** | lcov.info / cobertura.json → NodeId 盲區綁定 |

## 5. 終極部署與容器發布範式 (Distroless / Single Binary Container)

利用各語言編譯出的單一二進位檔（Single Binary），發布與容器化時可採用
Distroless (`gcr.io/distroless/static-debian12`) 極致瘦身架構：

```dockerfile
# 終極瘦身、零攻擊面與保密 Dockerfile 範例
FROM gcr.io/distroless/static-debian12

# 1. 複製各語言編譯出的單一靜態 Binary
COPY my-php-tool /usr/local/bin/my-php-tool
COPY my-python-tool /usr/local/bin/my-python-tool

# 2. 複製 Graphify Core Rust Binary
COPY graphify /usr/local/bin/graphify

ENTRYPOINT ["/usr/local/bin/my-php-tool"]
```

### 架構效益

- **極致瘦身**：整體 Image 大小控制在 30MB ~ 80MB
- **零資安攻擊面**：容器內無 Shell (`/bin/sh`)、無套件管理器，通過 100% 資安稽核
- **商業保密**：無 Plaintext 腳本落盤，商業邏輯已被鎖入 Machine Code