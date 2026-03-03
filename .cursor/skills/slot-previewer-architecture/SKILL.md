---
name: slot-previewer-architecture
description: Core architecture rules for Slot Previewer. Enforces layered architecture, dependency direction, data flow, module boundaries, and extensibility patterns. Use when creating new modules, files, imports, or making structural decisions.
---

# Slot Previewer — 架構核心規則

## 分層架構（嚴格單向）

```
React UI → Zustand Store → PixiJS Managers → Scene Graph → Canvas
```

| 層 | 職責 | 可引用 |
|---|------|--------|
| **UI** (`src/ui/`, `src/app/`) | 面板、工具列 | Store, hooks, types |
| **Store** (`src/store/`) | 單一真相來源 | types only |
| **Manager** (`src/pixi/`, `src/spine/`, `src/reel/`) | 場景管理 | Store (subscribe), PixiJS, types |
| **Script** (`src/script/`) | 腳本執行（唯一允許跨層） | Store, Manager, types |

## 禁止規則

- UI **禁止** import Manager 或直接操作 PixiJS 物件
- Manager **禁止** import UI 元件或 React Hook
- Store 之間 **禁止** 互相 import（跨 Store 用 `getState()` 或組合 action）
- UI **不訂閱** EventBus（EventBus 僅限 Manager 間通訊）
- React **不進入** PixiJS Ticker 迴圈

## 模組通訊

| 路徑 | 方式 |
|------|------|
| UI → Store | Zustand action |
| Store → Manager | `store.subscribe()` |
| Manager ↔ Manager | EventBus |
| Manager → UI | 透過 Store 狀態更新（間接） |
| ScriptRunner → 任意 | 透過 ActionContext（設計例外） |

## 目錄結構

```
src/
├── app/          # React 入口
├── pixi/         # PixiApp, LayerManager, ClipManager
├── spine/        # SpineLoader, SpineActor
├── reel/         # ReelEngine, ReelConfig, SymbolPool
├── script/       # ScriptRunner, EventBus, actions/
├── store/        # Zustand slices（7 個）
├── io/           # ProjectExporter, ProjectImporter
├── ui/           # layout/, panels/, toolbar/, components/
├── hooks/        # React hooks
├── utils/        # 工具函式
└── types/        # 所有 TypeScript 型別（單一來源）
```

## 擴充性模式

- **新 Action**：實作 `ActionHandler` 介面 → 註冊至 `ActionRegistry`（不改 ScriptRunner）
- **新圖層類型**：擴充 `LayerType` 聯合型別 → 加對應 data interface → LayerManager 加 handler
- **新面板**：`src/ui/panels/` 加元件 → 右側 Tab 註冊
- **Schema 升級**：只加欄位（`?` 可選）→ 加 Migration 函式 → 鏈式遷移

## 技術棧

React 18 + TS 5 | PixiJS 7.4.x（WebGL2）| @pixi-spine/all-3.8（Spine 3.8.99）| Zustand + Immer | GSAP 3（僅腳本層）| Vite 5 | shadcn/ui + Tailwind | Vitest + Playwright

## 詳細設計

完整介面定義與實作細節見 [docs/](../../docs/README.md)。
