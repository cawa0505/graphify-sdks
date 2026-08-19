# Graphify 2.0.0 GA — Node.js / TypeScript SDK 技術規格書

> 跨語言 SDK、Plugin 與二進位編譯全貌規格書 — Node.js/TS 補全篇

## 1. 系統核心哲學：雙向 CLI 互查 (Bi-directional CLI Interop)

Graphify 放棄了複雜且需常駐背景進程（Daemon）的 RPC 綁定，全面採用 **「雙向 CLI 互查」** 與 **「純文字/JSON 標準流（Stdio）」** 作為跨語言與跨工具鏈溝通的神經總線。

```
                               Graphify Core (Rust 2.0.0 GA)
                               └── Standard NodeId & .toon Engine
                                                ▲
                                                │ 雙向 CLI 互查 (Process I/O + JSON)
         ┌──────────────────────────────────────┼──────────────────────────────────────┐
         │                                      │                                      │
         ▼                                      ▼                                      ▼
graphify-sdk-php                     graphify-sdk-golang                   graphify-sdk-js / ts
(Laravel / Pest)                      (Cobra / Go Test)                    (Bun / SEA Single Binary)
```

## 2. Node.js / TS 二進位編譯與打包方案評估

| 方案 / 技術 | 運作原理 | 啟動延遲 | 單一 Binary | 源碼保護力 | 適合場景 |
|------------|----------|---------|------------|-----------|---------|
| 🥇 **Bun Compile** (`bun build --compile`) | Bun 執行期 + Zig 編譯器，將 TS/JS 靜態編譯成獨立二進位檔 | < 10ms | 是 | 極高 | 雙向 CLI 工具首選 |
| 📦 Node.js SEA (Single Executable Applications) | 官方方案。將 JS Bundle (esbuild) 注入 Node.js Binary 資源區段 | 30-50ms | 是 | 高 | 標準 Node.js 跨平台工具發布 |
| ❌ pkg | 已廢棄，不推薦 | 慢 (~200ms+) | 是 | 低 | 不推薦 |

### 🚀 首選方案：Bun Compile

```
bun build --compile --minify ./src/index.ts --outfile graphify-tools-js
```

- 瞬間啟動 (< 10ms)
- TypeScript 原生支援（不需先跑 tsc）
- 零 node_modules：打包後約 30~40MB 獨立 Binary

### 📦 備選方案：esbuild + Node.js SEA

```
esbuild src/index.ts --bundle --minify --platform=node --outfile=bundle.js
node --experimental-sea-config sea-config.json
```

## 3. Node.js/TS 與 Graphify 雙向互查架構

### ➡️ 正向：Graphify 呼叫 JS/TS 工具

Node.js 專案執行 vitest coverage 產出 coverage-final.json → Graphify 透過 `graphify coverage ingest-json` 讀取並將 JS/TS 函數升維對齊至標準 NodeId。

### ⬅️ 反向：JS/TS 工具鏈呼叫 Graphify

透過 `child_process` 封裝呼叫 Graphify CLI：

```typescript
// graphify-sdk-js — Zero Dependency Transport
import { spawn } from 'child_process';
import { createHash } from 'crypto';

export class GraphifyClient {
  async queryNode(filePath: string): Promise<NodeInfo> {
    const { stdout } = await execa('graphify', ['query', '--file', filePath]);
    return JSON.parse(stdout);
  }
}
```

## 4. 補齊後的全語言 SDK 與二進位編譯全貌矩陣

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               Graphify Core 2.0.0 GA                                   │
│                        (Rust Single Native Binary, < 5ms)                              │
└───────────────────────────────────────────▲────────────────────────────────────────────┘
                                            │
                             雙向 CLI 互查 (Process I/O + JSON)
                                            │
 ┌──────────────────┬───────────────────────┼───────────────────────┬──────────────────┐
 │                  │                       │                       │                  │
 ▼                  ▼                       ▼                       ▼                  ▼
Rust (Core)      PHP SDK                 Go SDK                  Python (AOT)       Node.js / TS SDK
`cargo build`   `static-php-cli`        `go build`              **Nuitka**          **Bun Compile / SEA**
< 5ms           5 ~ 15ms                < 10ms                  10 ~ 30ms           < 10ms (Bun)
```

## 5. 總結

- **全語言二進位化**：Rust、PHP、Go、Python、Node.js/TS 五大生態系全數導向單一靜態 Binary
- **極致流暢度**：所有語言的二進位檔啟動時間全數控制在 < 30ms 內
- **商業級防護與容器瘦身**：所有語言打包後均可直接放入 Distroless Container（< 80MB）