# Slot Previewer — 架構總覽

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 1. 系統總覽

**Slot Previewer** 是一款瀏覽器端的 Slot 遊戲動畫預覽工具，專為**美術人員**設計。美術可以在工具中：

- 匯入並預覽 **Spine 動畫**（拖拽 / URL 載入）
- 自由調整**圖層順序、混合模式、可見性與鎖定**
- 模擬**轉輪運轉**（含自訂盤面、Symbol 配置）
- 撰寫或載入**演出腳本**（JSON / TypeScript），自動播放完整遊戲流程
- 將整個專案**匯出為 .zip**，分享給其他同事直接匯入使用

### 核心技術組合

| 用途 | 技術選型 |
|------|---------|
| UI 框架 | React 18 |
| 渲染引擎 | PixiJS 8（WebGL2 / WebGPU） |
| Spine 執行時 | @esotericsoftware/spine-pixi |
| 全局狀態 | Zustand |

### 使用者與部署

- **目標使用者**：美術人員——UI 必須簡單直覺，避免過多技術術語
- **單人使用**，透過專案檔（.zip）進行分享
- **介面語言**：僅繁體中文
- **部署方式**：靜態部署（Vite build 產出），未來規劃 Electron 打包為桌面應用程式

---

## 2. 核心設計原則

| 原則 | 說明 |
|------|------|
| **效能優先** | 渲染完全由 PixiJS 管理，React 僅負責工具列與面板。React 永遠不干涉 Canvas 的更新頻率，確保 60 fps 穩定。 |
| **圖層驅動** | 所有視覺元素皆為可堆疊的圖層節點（Layer Node），各自擁有獨立的可見性、鎖定、混合模式等屬性。 |
| **Clipping 原生支援** | 透過 PixiJS MaskData 實現精確的轉輪視窗裁切，不依賴 CSS overflow。 |
| **腳本可擴充** | 以 JSON / TS 腳本描述轉輪行為與動畫演出，設計師可自訂任意流程，無須修改核心程式碼。 |
| **模組化** | 每個子系統（LayerManager、SpinePlayer、ReelController、ScriptEngine）獨立封裝，可單獨測試與未來擴充。 |
| **擴充性優先** | 架構必須支援未來新增：新的轉輪類型（如 Megaways 可變列數）、新的 Action 類型、新的面板類型。 |

---

## 3. 技術棧

| 類別 | 技術 | 版本 / 備註 |
|------|------|-------------|
| 框架 | React + TypeScript | React 18、TypeScript 5 |
| 渲染 | PixiJS | 8.x（WebGL2 為主，WebGPU fallback） |
| Spine 執行時 | @esotericsoftware/spine-pixi | 4.2.x |
| 狀態管理 | Zustand + Immer middleware | 4.x |
| 動畫補間 | GSAP | 3.x（僅用於腳本層，不介入渲染層） |
| 建置工具 | Vite + React plugin | 5.x |
| UI 元件 | shadcn/ui + Tailwind CSS | Glassmorphism 深色主題（採用 JR UI Design System） |
| 測試 | Vitest + Playwright | 單元測試 + E2E |
| 專案 I/O | JSZip | .zip 專案匯出 / 匯入 |

---

## 4. 系統架構圖

### 4.1 分層架構

```
┌──────────────────────────────────────────────────────────────┐
│                       React UI Layer                         │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐ │
│  │ LayerPanel │  │SpinePanel │  │ ReelPanel  │  │ScriptPanel│ │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘ │
│        │               │              │               │      │
│  ┌─────▼───────────────▼──────────────▼───────────────▼────┐ │
│  │            Zustand Store (Single Source of Truth)         │ │
│  │  layerStore │ spineStore │ reelStore │ scriptStore        │ │
│  │  previewStore │ uiStore │ projectStore                    │ │
│  └─────┬───────────────┬──────────────┬───────────────┬────┘ │
│        │               │              │               │      │
│  ┌─────▼───────────────▼──────────────▼───────────────▼────┐ │
│  │             PixiJS Engine Layer (Managers)                │ │
│  │  LayerManager │ ClipManager │ ReelEngine │ SpineLoader    │ │
│  └─────┬──────────────────────────────────────────────┬────┘ │
│        │                                              │      │
│  ┌─────▼──────────────────────────────────────────────▼────┐ │
│  │               PixiJS Application (Canvas)                │ │
│  │           WebGL2 / WebGPU Renderer + Ticker              │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 資料流向

```
使用者操作 → React UI → Zustand Store → PixiJS Managers → PixiJS Scene Graph → Canvas 渲染
```

**單向資料流**：UI 層透過 Zustand action 修改 Store，Managers 訂閱 Store 變化後更新 PixiJS Scene Graph，最終由 PixiJS Ticker 驅動 Canvas 渲染。UI 層永遠不直接操作 PixiJS 物件。

### 4.3 腳本引擎流程

```
JSON Script → ScriptRunner → Action Queue → ┬→ Zustand Store（狀態更新）
                                             └→ PixiJS Managers（直接指令）
```

ScriptRunner 解析腳本後，將動作推入 Action Queue 依序執行。每個 Action 可選擇：
- 更新 Zustand Store（觸發 UI 與 Managers 同步）
- 直接呼叫 Manager API（需要即時反應的低延遲操作）

---

## 5. 專案目錄結構

```
slot-previewer/
├── public/                    # 靜態資源
├── src/
│   ├── app/                   # React 頁面與路由
│   │   ├── App.tsx
│   │   └── main.tsx
│   │
│   ├── pixi/                  # PixiJS 核心
│   │   ├── PixiApp.ts              # Application 初始化與生命週期
│   │   ├── LayerManager.ts         # 圖層系統管理
│   │   ├── ClipManager.ts          # Clipping / Mask 系統
│   │   └── ReelContainer.ts        # 轉輪容器
│   │
│   ├── spine/                 # Spine 動畫封裝
│   │   ├── SpineLoader.ts          # 資源載入（拖拽 / URL）
│   │   └── SpineActor.ts           # 動畫實例高階 API
│   │
│   ├── reel/                  # 轉輪模擬
│   │   ├── ReelEngine.ts           # 轉輪運動邏輯
│   │   ├── ReelConfig.ts           # 盤面設定
│   │   ├── ReelGridEditor.ts       # 盤面網格編輯器
│   │   └── SymbolPool.ts           # Symbol 物件池
│   │
│   ├── script/                # 腳本引擎
│   │   ├── ScriptRunner.ts         # 腳本解析與執行
│   │   ├── EventBus.ts             # 事件匯流排
│   │   └── actions/                # 各 Action handler
│   │       ├── SpinAction.ts
│   │       ├── StopReelAction.ts
│   │       ├── PlaySpineAction.ts
│   │       ├── TweenAction.ts
│   │       └── index.ts
│   │
│   ├── store/                 # Zustand 全局狀態
│   │   ├── layerStore.ts           # 圖層狀態
│   │   ├── spineStore.ts           # Spine 動畫狀態
│   │   ├── reelStore.ts            # 轉輪狀態
│   │   ├── scriptStore.ts          # 腳本狀態
│   │   ├── previewStore.ts         # 預覽控制狀態
│   │   ├── uiStore.ts              # UI 面板狀態
│   │   └── projectStore.ts         # 專案 I/O 狀態
│   │
│   ├── io/                    # 專案匯出匯入
│   │   ├── ProjectExporter.ts      # .zip 匯出邏輯
│   │   ├── ProjectImporter.ts      # .zip 匯入邏輯
│   │   └── ProjectSchema.ts        # 專案檔 JSON Schema
│   │
│   ├── ui/                    # React 工具面板
│   │   ├── layout/                 # 整體佈局
│   │   ├── panels/                 # 各功能面板
│   │   ├── toolbar/                # 頂部工具列
│   │   └── components/             # 共用 UI 元件
│   │
│   ├── hooks/                 # React Hooks
│   ├── utils/                 # 工具函式
│   └── types/                 # 共用型別定義
│
├── docs/                      # 架構文件
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 6. 模組依賴關係圖

```
┌────────────────┐
│   React UI     │  （panels, toolbar, layout）
│   Components   │
└───────┬────────┘
        │ 讀取 / 寫入
        ▼
┌────────────────┐
│  Zustand Store │  （layerStore, spineStore, reelStore, scriptStore ...）
│  (狀態中樞)     │
└───────┬────────┘
        │ 訂閱變化
        ▼
┌────────────────┐
│  PixiJS        │  （LayerManager, ClipManager, ReelEngine, SpineLoader）
│  Managers      │
└───────┬────────┘
        │ 操作 Scene Graph
        ▼
┌────────────────┐
│  PixiJS Core   │  （Application, Container, Sprite, Ticker ...）
│  (渲染引擎)     │
└────────────────┘
```

### 關鍵規則

| 規則 | 說明 |
|------|------|
| **UI → Store → Managers** | UI 層透過 Store 間接驅動 Managers，永遠不直接 import Manager 模組。 |
| **Managers ⛔ UI** | Managers **永遠不得**匯入 UI 層的任何模組，確保引擎層可獨立運作。 |
| **Store 是唯一橋梁** | Store 是 UI 層與 Engine 層之間的唯一通訊管道，所有狀態同步皆透過 Store 完成。 |
| **Managers → PixiJS** | Managers 直接操作 PixiJS Scene Graph，對外透過 Store subscription 接收指令。 |

---

## 7. 擴充性設計要點

### 7.1 Plugin-like Action 註冊

ScriptEngine 採用**註冊制**管理 Action handler：

```typescript
// 註冊新的 Action 類型
ActionRegistry.register('customAction', CustomActionHandler);
```

新增演出動作時，只需實作 `ActionHandler` 介面並註冊，無須修改 ScriptRunner 核心邏輯。

### 7.2 可變列數轉輪（Megaways 風格）

ReelEngine 透過 `ReelConfig` 支援**每軸可變列數**：

```typescript
interface ReelConfig {
  reelCount: number;          // 軸數
  rowsPerReel: number[];      // 每軸列數（如 [3, 4, 5, 4, 3] 即 Megaways 風格）
  symbolSize: { w: number; h: number };
  // ...
}
```

### 7.3 自訂圖層類型

LayerManager 使用 **type discriminator** 區分圖層類型：

```typescript
type LayerNode =
  | { type: 'sprite'; /* ... */ }
  | { type: 'spine';  /* ... */ }
  | { type: 'reel';   /* ... */ }
  | { type: 'group';  /* ... */ }
  | { type: 'custom'; handler: string; /* ... */ };
```

未來新增圖層類型只需擴充聯合型別並實作對應的渲染邏輯。

### 7.4 Observer 模式

所有 Managers 遵循 **Observer 模式**，訂閱 Zustand Store 變化：

```typescript
class LayerManager {
  constructor() {
    layerStore.subscribe(
      (state) => state.layers,
      (layers) => this.syncSceneGraph(layers)
    );
  }
}
```

Manager 不主動輪詢，而是被動響應 Store 狀態變化，降低耦合並提升效能。

### 7.5 專案 I/O Schema 版本化

專案檔的 JSON Schema 包含版本號，確保**向前相容**：

```json
{
  "schemaVersion": "1.0.0",
  "project": { /* ... */ }
}
```

匯入時根據 `schemaVersion` 選擇對應的 migration 邏輯，確保舊版專案檔可在新版工具中正確載入。

---

> **下一份文件**：`02-layer-system.md` — 圖層系統設計規格
