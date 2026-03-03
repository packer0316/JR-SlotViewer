# Slot Previewer — 模組設計規格

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置文件**：`01-architecture-overview.md`
> **適用對象**：開發人員、技術主管

---

## 1. 模組總覽

| 模組 | 路徑 | 職責 |
|------|------|------|
| **PixiApp** | `src/pixi/PixiApp.ts` | PixiJS Application 生命週期管理（初始化 / 銷毀 / 視窗縮放） |
| **LayerManager** | `src/pixi/LayerManager.ts` | 圖層樹 CRUD 操作，同步 Zustand Store 與 PixiJS Container 樹 |
| **ClipManager** | `src/pixi/ClipManager.ts` | 遮罩系統——支援 Graphics / Sprite / Stencil 三種遮罩模式 |
| **SpineLoader** | `src/spine/SpineLoader.ts` | Spine 資源載入（拖拽 / URL），快取管理 |
| **SpineActor** | `src/spine/SpineActor.ts` | Spine 動畫實例高階 API（播放 / 混合 / 軌道 / 皮膚 / 附件） |
| **ReelEngine** | `src/reel/ReelEngine.ts` | 轉輪運動模擬（啟動 / 停止 / 彈跳 / 快停） |
| **ReelConfig** | `src/reel/ReelConfig.ts` | 盤面設定介面——NxM 可變列數、Symbol 尺寸、間距、對齊方式 |
| **ReelGridEditor** | `src/reel/ReelGridEditor.ts` | 視覺化盤面網格編輯器，即時預覽盤面佈局 |
| **SymbolPool** | `src/reel/SymbolPool.ts` | Symbol 物件池——減少 GC 壓力，提升轉輪渲染效能 |
| **ScriptRunner** | `src/script/ScriptRunner.ts` | JSON 腳本解析與依序執行 |
| **EventBus** | `src/script/EventBus.ts` | 事件發射器——跨模組鬆耦合通訊 |
| **ProjectExporter** | `src/io/ProjectExporter.ts` | 將專案狀態與資源打包匯出為 `.zip` |
| **ProjectImporter** | `src/io/ProjectImporter.ts` | 從 `.zip` 匯入專案，還原狀態與資源 |

---

## 2. 各模組詳細設計

### 2.1 PixiApp

#### 職責

管理 PixiJS `Application` 的完整生命週期，包括 Canvas 初始化、Ticker 啟動、視窗大小變更與資源銷毀。所有其他 Manager 都依賴此模組取得 `Application`、`Stage`、`Ticker` 的參考。

#### 公開 API

```typescript
class PixiApp {
  /**
   * 初始化 PixiJS Application 並掛載至指定 Canvas。
   * 會自動設定 WebGL2 渲染器與 Ticker。
   */
  async init(canvas: HTMLCanvasElement, width: number, height: number): Promise<void>

  /** 銷毀 Application 並釋放所有 GPU 資源 */
  destroy(): void

  /** 調整 Canvas 與 Renderer 的尺寸 */
  resize(width: number, height: number): void

  /** 取得 PixiJS Application 實例 */
  getApp(): Application

  /** 取得 Ticker 實例（用於自訂 update loop） */
  getTicker(): Ticker

  /** 取得 Stage 根容器 */
  getStage(): Container
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| PixiJS 8 | `Application`, `Container`, `Ticker` |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `previewStore` | `canvasSize` | 監聽畫布尺寸變化，自動呼叫 `resize()` |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `pixi:ready` | `{ app: Application }` | Application 初始化完成 |
| `pixi:resize` | `{ width: number; height: number }` | Canvas 尺寸已變更 |
| `pixi:destroyed` | — | Application 已銷毀 |

---

### 2.2 LayerManager

#### 職責

維護圖層樹結構，將 `layerStore` 中的 `LayerNode[]` 同步至 PixiJS Container 樹。負責圖層的新增、刪除、移動、屬性更新，並確保 Container 層級與 Store 狀態一致。

#### 公開 API

```typescript
class LayerManager {
  constructor(app: PixiApp)

  /** 新增圖層節點，同時建立對應的 PixiJS Container */
  addLayer(node: LayerNode): void

  /** 刪除圖層及其對應的 Container */
  removeLayer(id: string): void

  /** 移動圖層至新的排序位置 */
  moveLayer(id: string, newIndex: number): void

  /** 更新圖層屬性（可見性、鎖定、混合模式等） */
  updateLayer(id: string, props: Partial<LayerNode>): void

  /** 取得指定圖層對應的 PixiJS Container */
  getContainer(id: string): Container | null

  /** 從 Store 同步完整圖層樹（Store 變化時自動呼叫） */
  syncFromStore(): void
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| `PixiApp` | 取得 Stage 根容器以掛載圖層 Container |
| `ClipManager` | 當圖層綁定遮罩時，委託 ClipManager 處理 |
| PixiJS 8 | `Container`, `Sprite`, `BlendMode` |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `layerStore` | `layers` | 監聽圖層樹變化，觸發 `syncFromStore()` |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `layer:added` | `{ id: string; node: LayerNode }` | 圖層已新增 |
| `layer:removed` | `{ id: string }` | 圖層已刪除 |
| `layer:moved` | `{ id: string; newIndex: number }` | 圖層順序已變更 |
| `layer:updated` | `{ id: string; props: Partial<LayerNode> }` | 圖層屬性已更新 |

---

### 2.3 ClipManager

#### 職責

管理所有遮罩（Mask）物件的建立、套用與移除。支援三種遮罩類型：矩形 / 多邊形（Graphics）、Sprite 遮罩、Stencil 遮罩。轉輪視窗的裁切即透過本模組實現。

#### 公開 API

```typescript
class ClipManager {
  /** 建立矩形遮罩 */
  buildRectMask(id: string, rect: { x: number; y: number; w: number; h: number }): void

  /** 建立多邊形遮罩 */
  buildPolyMask(id: string, points: number[]): void

  /** 建立 Sprite 遮罩（以灰階貼圖控制透明度） */
  buildSpriteMask(id: string, texture: Texture): void

  /** 將指定遮罩套用至圖層 */
  applyClip(layerId: string, clipId: string): void

  /** 移除圖層上的遮罩 */
  removeClip(layerId: string): void

  /** 更新既有遮罩的參數 */
  updateMask(id: string, rect: Partial<Rect>): void

  /** 取得遮罩資料 */
  getClip(id: string): MaskData | null
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| PixiJS 8 | `Graphics`, `Sprite`, `Texture`, `MaskData` |
| `LayerManager` | 需取得目標圖層的 Container 以套用遮罩 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `layerStore` | `layers[].clip` | 監聽圖層遮罩設定變化 |

#### 發出的事件

無。ClipManager 為被動模組，僅回應 LayerManager 或 Store 的呼叫。

---

### 2.4 SpineLoader

#### 職責

負責 Spine 動畫資源的載入與快取。支援兩種載入方式：URL 路徑載入與本地檔案拖拽載入。載入完成後回傳 `SpineActor` 實例。

#### 公開 API

```typescript
class SpineLoader {
  /** 從 URL 載入 Spine 資源 */
  static async load(config: {
    skelPath: string;
    atlasPath: string;
  }): Promise<SpineActor>

  /** 從本地檔案載入 Spine 資源（拖拽匯入） */
  static async loadFromFiles(
    skelFile: File,
    atlasFile: File,
    pngFiles: File[]
  ): Promise<SpineActor>

  /** 取得目前的快取 Map */
  static getCache(): Map<string, SpineActor>

  /** 清除所有快取 */
  static clearCache(): void
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| `@esotericsoftware/spine-pixi` | Spine 執行時核心 |
| PixiJS 8 | `Assets` loader |
| `SpineActor` | 載入完成後包裝為 SpineActor 實例 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `spineStore` | `loadRequests` | 監聽新的載入請求 |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `spine:loaded` | `{ id: string; actor: SpineActor }` | Spine 資源載入完成 |
| `spine:loadError` | `{ id: string; error: Error }` | 載入失敗 |

---

### 2.5 SpineActor

#### 職責

封裝單一 Spine 動畫實例的高階 API。提供播放、混合、軌道控制、皮膚切換、附件切換等功能，遮蔽底層 `@esotericsoftware/spine-pixi` 的實作細節。

#### 公開 API

```typescript
class SpineActor {
  /** 播放指定動畫 */
  play(animName: string, loop?: boolean): void

  /** 混合切換至指定動畫 */
  mixTo(animName: string, mixDuration: number): void

  /** 在指定軌道播放動畫 */
  setTrack(trackIndex: number, animName: string, loop?: boolean): void

  /** 切換皮膚 */
  setSkin(skinName: string): void

  /** 設定指定 Slot 的附件 */
  setAttachment(slotName: string, attachmentName: string): void

  /** 動畫播放完成回呼 */
  onComplete(callback: () => void): void

  /** 暫停播放 */
  pause(): void

  /** 恢復播放 */
  resume(): void

  /** 設定播放速度（1.0 為正常速度） */
  setSpeed(speed: number): void

  /** 取得所有可用動畫名稱 */
  getAnimationNames(): string[]

  /** 取得所有可用皮膚名稱 */
  getSkinNames(): string[]

  /** 取得所有 Slot 名稱 */
  getSlotNames(): string[]

  /** 銷毀實例並釋放資源 */
  dispose(): void
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| `@esotericsoftware/spine-pixi` | `Spine`, `AnimationState`, `Skeleton` |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `spineStore` | `actors[id].playback` | 監聽播放控制指令（播放 / 暫停 / 速度） |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `spine:animComplete` | `{ id: string; animName: string; trackIndex: number }` | 動畫播放完成 |
| `spine:animEvent` | `{ id: string; eventName: string; data: any }` | Spine 自訂事件觸發 |

---

### 2.6 ReelEngine

#### 職責

轉輪運動模擬引擎。根據 `ReelConfig` 建立轉輪容器並控制運動行為——包括啟動旋轉、逐軸停止（含彈跳動畫）、快停、設定結果等。透過 PixiJS Ticker 驅動逐幀更新。

#### 公開 API

```typescript
class ReelEngine {
  constructor(config: ReelConfig, app: PixiApp)

  /** 啟動所有轉輪旋轉，可指定各軸速度 */
  spin(speeds?: number[]): void

  /** 停止指定軸並滾至目標位置（可配置彈跳效果） */
  stopReel(index: number, targetPos: number, bounce?: BounceConfig): void

  /** 依序停止所有軸（可設定軸間延遲） */
  stopAll(delay?: number): void

  /** 快停——所有軸立即停止至目標位置 */
  quickStop(): void

  /** 設定最終結果（停輪後顯示的 Symbol 排列） */
  setResult(result: number[][]): void

  /** 取得所有轉輪的 PixiJS Container */
  getReelContainers(): Container[]

  /** 銷毀轉輪引擎並釋放資源 */
  dispose(): void
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| `PixiApp` | 取得 Ticker 用於逐幀更新 |
| `ReelConfig` | 盤面設定參數 |
| `SymbolPool` | 從物件池取得 / 歸還 Symbol |
| PixiJS 8 | `Container`, `Ticker` |
| GSAP | 彈跳動畫補間 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `reelStore` | `command` | 監聽轉輪操作指令（spin / stop / quickStop） |
| `reelStore` | `config` | 監聽盤面設定變更，重建轉輪佈局 |
| `reelStore` | `result` | 監聽結果設定 |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `reel:spinStart` | — | 所有軸開始旋轉 |
| `reel:reelStopped` | `{ index: number }` | 指定軸已停止 |
| `reel:allStopped` | — | 所有軸皆已停止 |
| `reel:quickStopped` | — | 快停完成 |

---

### 2.7 ReelConfig

#### 職責

定義轉輪盤面的完整設定——軸數、每軸列數（支援可變列數）、Symbol 尺寸、間距、對齊方式、Symbol 帶定義、轉速與緩動參數。此為純資料介面（interface），不包含邏輯。

#### 型別定義

```typescript
interface ReelConfig {
  /** 轉輪軸數（欄數） */
  reels: number

  /** 每軸列數，支援可變列數（如 [3, 4, 5, 4, 3] 即 Megaways 風格） */
  rowsPerReel: number[]

  /** 單一 Symbol 的尺寸 */
  symbolSize: { w: number; h: number }

  /** Symbol 間的間距 */
  symbolGap: { x: number; y: number }

  /** 垂直對齊方式 */
  alignment: 'top' | 'center' | 'bottom'

  /** 各軸的 Symbol 帶（數字陣列代表 Symbol ID） */
  reelStrips: number[][]

  /** Symbol 定義對照表 */
  symbols: Record<string, SymbolDef>

  /** 基礎轉速（像素 / 幀） */
  spinSpeed: number

  /** 緩動設定 */
  easeConfig: EaseConfig

  /** 彈跳設定 */
  bounceConfig: BounceConfig
}

interface SymbolDef {
  id: string
  name: string
  texture: string          // 貼圖路徑或 SpineActor ID
  type: 'static' | 'spine' // 靜態圖片或 Spine 動畫
}

interface EaseConfig {
  startEase: string        // GSAP ease 字串（如 'power2.in'）
  stopEase: string         // GSAP ease 字串（如 'power2.out'）
  startDuration: number    // 加速階段時長（秒）
  stopDuration: number     // 減速階段時長（秒）
}

interface BounceConfig {
  enabled: boolean
  amplitude: number        // 彈跳幅度（像素）
  duration: number         // 彈跳時長（秒）
  ease: string             // GSAP ease 字串
}
```

#### 依賴

無。純資料介面。

#### 訂閱的 Store

不適用（介面定義，非類別）。

#### 發出的事件

不適用。

---

### 2.8 ReelGridEditor

#### 職責

提供視覺化的盤面網格編輯器，讓美術人員可以直覺地設定轉輪佈局。支援即時預覽，所有變更即時反映至 `reelStore` 並同步至 `ReelEngine`。

#### 功能項目

| 功能 | 說明 |
|------|------|
| 設定軸數 | 使用者可增減轉輪軸數（欄數） |
| 設定每軸列數 | 每軸可獨立設定列數，支援可變列數（如 Megaways 風格） |
| Symbol 尺寸設定 | 拖拽或數值輸入調整 Symbol 寬高 |
| 間距設定 | 設定 Symbol 間的水平與垂直間距 |
| 對齊方式 | 選擇 `top` / `center` / `bottom` 垂直對齊 |
| 即時預覽 | 以網格線框即時顯示盤面佈局 |

#### 互動流程

```
使用者調整設定 → ReelGridEditor 更新 UI
                → 寫入 reelStore.config
                → ReelEngine 監聽到變化
                → 重建轉輪佈局
                → Canvas 即時渲染新佈局
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| React | UI 元件渲染 |
| Zustand | 讀寫 `reelStore` |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `reelStore` | `config` | 讀取並顯示目前的盤面設定 |

#### 發出的事件

無。透過 Store 與 `ReelEngine` 間接通訊。

---

### 2.9 SymbolPool

#### 職責

Symbol 物件池——以池化模式管理轉輪中的 Symbol 節點，避免頻繁建立與銷毀 PixiJS 顯示物件，降低 GC 壓力並提升轉輪高速旋轉時的渲染效能。

#### 公開 API

```typescript
class SymbolPool {
  /** 從池中取得一個 Symbol 節點（若池為空則新建） */
  acquire(symbolId: string): SymbolNode

  /** 將 Symbol 節點歸還至池中（重置狀態） */
  release(node: SymbolNode): void

  /** 預載指定 Symbol 類型的節點至池中 */
  preload(symbolIds: string[]): void

  /** 清空池中所有節點並釋放資源 */
  clear(): void

  /** 取得目前池中的節點總數 */
  getPoolSize(): number
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| PixiJS 8 | `Container`, `Sprite` |
| `ReelConfig` | 取得 `symbols` 定義以建立對應的顯示物件 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `reelStore` | `config.symbols` | 當 Symbol 定義變更時，清除並重建池 |

#### 發出的事件

無。SymbolPool 為被動工具模組。

---

### 2.10 ScriptRunner

#### 職責

JSON 腳本的解析與執行引擎。接收 `ScriptDefinition`，解析為 `ActionQueue`，然後依序執行每個 Action。支援暫停、恢復、中止，以及執行進度查詢。

#### 公開 API

```typescript
class ScriptRunner {
  /** 解析 JSON 腳本為 Action 佇列 */
  parseScript(json: ScriptDefinition): ActionQueue

  /** 開始執行 Action 佇列 */
  async run(): Promise<void>

  /** 暫停執行 */
  pause(): void

  /** 恢復執行 */
  resume(): void

  /** 中止執行（不可恢復） */
  abort(): void

  /** 取得目前執行到的步驟索引 */
  getCurrentStep(): number

  /** 取得總步驟數 */
  getTotalSteps(): number

  /** 取得執行器狀態 */
  getState(): 'idle' | 'running' | 'paused' | 'aborted'
}
```

#### Action 處理機制

ScriptRunner 本身不包含任何 Action 的實作邏輯，而是透過 `ActionRegistry` 查詢對應的 `ActionHandler`：

```typescript
interface ActionHandler {
  type: string
  execute(params: Record<string, any>, context: ActionContext): Promise<void>
}

interface ActionContext {
  store: StoreAPI
  eventBus: EventBus
  getManager<T>(name: string): T
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| `EventBus` | 發出腳本生命週期事件 |
| `ActionRegistry` | 查詢 Action handler |
| Zustand Store | 透過 `ActionContext` 存取 Store |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `scriptStore` | `currentScript` | 監聽載入的腳本 |
| `scriptStore` | `command` | 監聽播放控制指令（run / pause / resume / abort） |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `script:start` | `{ totalSteps: number }` | 腳本開始執行 |
| `script:step` | `{ step: number; action: string }` | 進入下一步驟 |
| `script:pause` | `{ step: number }` | 腳本已暫停 |
| `script:resume` | `{ step: number }` | 腳本已恢復 |
| `script:abort` | `{ step: number }` | 腳本已中止 |
| `script:complete` | — | 腳本執行完畢 |
| `script:error` | `{ step: number; error: Error }` | 步驟執行發生錯誤 |

---

### 2.11 EventBus

#### 職責

全域事件發射器，提供跨模組的鬆耦合通訊管道。各 Manager 可透過 EventBus 發出事件，其他模組（或 ScriptRunner 的 Action）可訂閱這些事件以執行後續邏輯。

#### 公開 API

```typescript
class EventBus {
  /** 訂閱事件 */
  on(event: string, handler: Function): void

  /** 取消訂閱 */
  off(event: string, handler: Function): void

  /** 訂閱事件（僅觸發一次） */
  once(event: string, handler: Function): void

  /** 發出事件 */
  emit(event: string, payload?: any): void

  /** 移除所有事件監聽 */
  removeAllListeners(): void
}
```

#### 依賴

無。EventBus 為獨立的基礎設施模組。

#### 訂閱的 Store

無。EventBus 不依賴任何 Store。

#### 發出的事件

EventBus 本身不定義事件，僅提供事件傳遞機制。各模組自行定義並發出事件（參見各模組的「發出的事件」段落）。

---

### 2.12 ProjectExporter

#### 職責

將完整專案狀態（Zustand Store 快照）與所有引用的資源檔案（Spine 骨骼、圖集、貼圖等）打包為 `.zip` 檔案，供使用者下載或分享。

#### 公開 API

```typescript
class ProjectExporter {
  /**
   * 匯出專案為 .zip Blob。
   * 包含 project.json（狀態快照）與 assets/ 目錄（所有資源檔）。
   */
  static async export(state: ProjectState): Promise<Blob>
}
```

#### .zip 結構

```
project.zip
├── project.json          # 專案狀態（含 schemaVersion）
└── assets/
    ├── spines/
    │   ├── character.skel
    │   ├── character.atlas
    │   └── character.png
    ├── symbols/
    │   ├── sym_h1.png
    │   └── sym_h2.png
    └── textures/
        └── background.png
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| JSZip | .zip 檔案建立 |
| Zustand Store | 讀取所有 Store 的狀態快照 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `projectStore` | `exportRequest` | 監聽匯出請求 |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `project:exportStart` | — | 開始匯出 |
| `project:exportComplete` | `{ blob: Blob; size: number }` | 匯出完成 |
| `project:exportError` | `{ error: Error }` | 匯出失敗 |

---

### 2.13 ProjectImporter

#### 職責

從使用者選取的 `.zip` 檔案還原專案。解析 `project.json`，驗證 Schema 版本並執行必要的 migration，然後還原所有 Store 狀態並重新載入引用的資源。

#### 公開 API

```typescript
class ProjectImporter {
  /**
   * 從 .zip 檔案匯入專案。
   * 解析 project.json、驗證版本、還原 Store 狀態、重新載入資源。
   */
  static async import(file: File): Promise<ProjectState>
}
```

#### 依賴

| 依賴項 | 說明 |
|--------|------|
| JSZip | .zip 檔案解壓 |
| `ProjectSchema` | JSON Schema 驗證與版本 migration |
| `SpineLoader` | 重新載入 Spine 資源 |
| Zustand Store | 還原所有 Store 狀態 |

#### 訂閱的 Store

| Store | Slice | 用途 |
|-------|-------|------|
| `projectStore` | `importRequest` | 監聽匯入請求 |

#### 發出的事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `project:importStart` | — | 開始匯入 |
| `project:importComplete` | `{ state: ProjectState }` | 匯入完成 |
| `project:importError` | `{ error: Error }` | 匯入失敗 |

---

## 3. 模組間通訊方式

### 3.1 通訊模式總覽

```
┌──────────┐   Zustand    ┌──────────┐  subscribe()  ┌──────────┐
│  React   │──actions()──▶│  Store   │──────────────▶│ Managers │
│  UI      │◀──selector──│  (狀態)   │◀──setState──│ (引擎)    │
└──────────┘              └──────────┘               └────┬─────┘
                                                          │
                                                    EventBus
                                                          │
                                              ┌───────────┴───────────┐
                                              │    ScriptRunner       │
                                              │  (腳本指揮所有模組)     │
                                              └───────────────────────┘
```

### 3.2 各通訊路徑

| 路徑 | 方式 | 說明 |
|------|------|------|
| **UI → Store** | React 元件呼叫 Zustand actions | 使用者操作（如拖拽圖層、點擊播放）轉化為 Store action 呼叫 |
| **Store → Managers** | Managers 透過 `store.subscribe()` 監聽 | Manager 在建構時訂閱相關 slice，當資料變化時自動同步至 PixiJS Scene Graph |
| **Manager → Manager** | 透過 EventBus（鬆耦合） | Manager 之間不直接引用，而是透過 EventBus 事件進行通訊，確保可獨立測試 |
| **Manager → UI** | 透過更新 Store 狀態（單向） | Manager 將計算結果或狀態回寫至 Store，UI 透過 selector 自動更新 |
| **ScriptRunner → 一切** | 透過 `ActionContext` 存取 Store 與 Manager | ScriptRunner 的 Action handler 可呼叫 Store actions 或直接操作 Manager API |

### 3.3 通訊規則

| 規則 | 詳細說明 |
|------|----------|
| **禁止 UI 直接呼叫 Manager** | UI 層永遠不得 import 或直接呼叫 Manager 方法。所有指令必須透過 Store action 間接傳遞。 |
| **禁止 Manager 引用 UI** | Manager 永遠不得引用 React 元件或 Hook，確保引擎層可獨立運作。 |
| **Store 是唯一的 UI ↔ Engine 橋梁** | Store 承擔 UI 層與 Engine 層之間的唯一通訊管道，保證狀態一致性。 |
| **EventBus 僅用於 Manager 間通訊** | EventBus 用於 Manager 之間的鬆耦合通訊，UI 層不應直接訂閱 EventBus 事件。 |
| **ScriptRunner 是唯一可跨層操作的模組** | ScriptRunner 透過 `ActionContext` 可存取 Store 與 Manager，這是設計上的例外，用以支援複雜的腳本流程。 |

---

## 4. 模組擴充指南

### 4.1 新增圖層類型

1. 擴充 `LayerNode` 聯合型別：

```typescript
// src/types/layer.ts
type LayerNode =
  | { type: 'sprite'; /* ... */ }
  | { type: 'spine';  /* ... */ }
  | { type: 'reel';   /* ... */ }
  | { type: 'group';  /* ... */ }
  | { type: 'video';  src: string; /* ... */ }  // ← 新增
```

2. 在 `LayerManager.syncFromStore()` 中新增對應的建立邏輯：

```typescript
// src/pixi/LayerManager.ts
private createContainer(node: LayerNode): Container {
  switch (node.type) {
    case 'sprite': return this.createSpriteContainer(node);
    case 'spine':  return this.createSpineContainer(node);
    case 'video':  return this.createVideoContainer(node);  // ← 新增
    // ...
  }
}
```

3. 在 UI 面板新增對應的屬性編輯器（若需要）。

---

### 4.2 新增腳本 Action

1. 在 `src/script/actions/` 建立新的 Action handler 檔案：

```typescript
// src/script/actions/WaitAction.ts
import type { ActionHandler, ActionContext } from '../types';

export const WaitAction: ActionHandler = {
  type: 'wait',
  async execute(params: { duration: number }, context: ActionContext) {
    await new Promise(resolve => setTimeout(resolve, params.duration));
  }
};
```

2. 在 `ActionRegistry` 中註冊：

```typescript
// src/script/actions/index.ts
import { WaitAction } from './WaitAction';

ActionRegistry.register(WaitAction.type, WaitAction);
```

3. 即可在 JSON 腳本中使用：

```json
{
  "actions": [
    { "type": "spin" },
    { "type": "wait", "params": { "duration": 2000 } },
    { "type": "stopAll", "params": { "delay": 300 } }
  ]
}
```

---

### 4.3 新增轉輪類型

1. 擴充 `ReelConfig` 介面，加入新的參數：

```typescript
interface ReelConfig {
  // ...既有欄位
  reelType: 'standard' | 'cascade' | 'cluster'  // ← 新增
  cascadeConfig?: CascadeConfig                   // ← 新增
}

interface CascadeConfig {
  fallSpeed: number
  fallDelay: number
  refillDelay: number
}
```

2. 在 `ReelEngine` 中新增對應的運動邏輯：

```typescript
// src/reel/ReelEngine.ts
class ReelEngine {
  spin(speeds?: number[]): void {
    switch (this.config.reelType) {
      case 'standard': this.standardSpin(speeds); break;
      case 'cascade':  this.cascadeSpin();        break;
      case 'cluster':  this.clusterSpin();        break;
    }
  }
}
```

3. 在 `ReelGridEditor` 中新增對應的 UI 選項。

---

### 4.4 新增 UI 面板

1. 在 `src/ui/panels/` 建立新的面板元件：

```typescript
// src/ui/panels/HistoryPanel.tsx
export function HistoryPanel() {
  const history = useHistoryStore((s) => s.entries);
  return (
    <div className="panel">
      {/* 面板內容 */}
    </div>
  );
}
```

2. 在右側面板的 Tab 列表中註冊新面板：

```typescript
// src/ui/layout/RightPanel.tsx
const PANEL_TABS = [
  { id: 'layers',  label: '圖層',   component: LayerPanel },
  { id: 'spine',   label: 'Spine',  component: SpinePanel },
  { id: 'reel',    label: '轉輪',   component: ReelPanel },
  { id: 'script',  label: '腳本',   component: ScriptPanel },
  { id: 'history', label: '歷史紀錄', component: HistoryPanel },  // ← 新增
];
```

3. 若面板需要獨立狀態，在 `src/store/` 建立對應的 Zustand Store。

---

> **下一份文件**：`03-data-models.md` — 資料模型與 Store 設計規格
