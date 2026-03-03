# Slot Previewer — 資料模型定義

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管

---

本文件定義 Slot Previewer 全部 TypeScript 型別與介面。所有型別集中於 `src/types/` 目錄，各 Store、Manager 與元件皆從此處匯入，確保整個專案使用**單一型別來源（Single Source of Type Truth）**。

---

## 目錄

1. [圖層系統型別](#1-圖層系統型別)
2. [Clipping 系統型別](#2-clipping-系統型別)
3. [轉輪系統型別](#3-轉輪系統型別)
4. [Spine 系統型別](#4-spine-系統型別)
5. [腳本系統型別](#5-腳本系統型別)
6. [專案 I/O 型別](#6-專案-io-型別)
7. [Store Slice 型別](#7-store-slice-型別)
8. [型別擴充指南](#8-型別擴充指南)

---

## 1. 圖層系統型別

圖層系統是 Slot Previewer 的核心資料結構。所有可見元素——Spine 動畫、靜態圖片、轉輪、群組——皆以 `LayerNode` 節點表示，形成樹狀結構供 `LayerManager` 管理。

### 1.1 LayerType

| 類型 | 說明 |
|------|------|
| `spine` | Spine 骨骼動畫圖層，引用 `SpineLayerData` |
| `sprite` | 靜態圖片圖層，引用 `SpriteLayerData` |
| `reel` | 轉輪容器圖層，引用 `ReelLayerData` |
| `clip` | 裁切遮罩圖層，不直接渲染，作為其他圖層的遮罩來源 |
| `group` | 群組圖層，可包含子節點，用於組織圖層結構 |

```typescript
type LayerType = 'spine' | 'sprite' | 'reel' | 'clip' | 'group';
```

### 1.2 LayerNode

`LayerNode` 是圖層樹的基本節點。透過 `children` 與 `parentId` 組成遞迴樹狀結構，`clipId` 可選地綁定一個 `ClipMask` 進行遮罩裁切。

```typescript
interface LayerNode {
  id: string;                          // UUID，唯一識別碼
  name: string;                        // 使用者可編輯的顯示名稱
  type: LayerType;                     // 圖層類型
  visible: boolean;                    // 是否可見
  locked: boolean;                     // 是否鎖定（鎖定後不可選取與編輯）
  alpha: number;                       // 透明度 0–1
  blendMode: BlendMode;               // PixiJS BlendMode
  zIndex: number;                      // 渲染排序優先級
  position: { x: number; y: number };  // 位置（相對於父節點）
  scale: { x: number; y: number };     // 縮放
  rotation: number;                    // 旋轉角度（弧度）
  clipId?: string;                     // 綁定的 ClipMask 節點 id
  children?: LayerNode[];              // 子節點（僅 group 類型使用）
  parentId?: string;                   // 父節點 id（根節點為 undefined）

  // 以下為各類型專屬資料，根據 type 決定使用哪一個
  spineData?: SpineLayerData;
  spriteData?: SpriteLayerData;
  reelData?: ReelLayerData;
}
```

### 1.3 SpineLayerData

Spine 圖層的專屬資料，描述動畫檔案路徑與播放狀態。

```typescript
interface SpineLayerData {
  skeletonPath: string;       // .skel 或 .json 骨骼檔路徑
  atlasPath: string;          // .atlas 圖集路徑
  currentAnimation: string;   // 當前播放的動畫名稱
  currentTrack: number;       // 動畫軌道索引（支援多軌混合）
  loop: boolean;              // 是否循環播放
  speed: number;              // 播放速度倍率（1.0 = 正常速度）
  skin?: string;              // 當前使用的 Skin 名稱
}
```

### 1.4 SpriteLayerData

靜態圖片圖層的專屬資料。

```typescript
interface SpriteLayerData {
  texturePath: string;                 // 圖片檔路徑
  anchor: { x: number; y: number };    // 錨點（0–1），預設 { x: 0.5, y: 0.5 }
  tint?: number;                       // 色彩疊加（16 進位色碼，如 0xff0000）
}
```

### 1.5 ReelLayerData

轉輪圖層的專屬資料，引用一個 `ReelConfig` 設定檔。

```typescript
interface ReelLayerData {
  configId: string;  // 對應 ReelConfig.id
}
```

### 1.6 欄位約束總覽

| 欄位 | 型別 | 預設值 | 約束 |
|------|------|--------|------|
| `id` | `string` | — | UUID v4，不可重複 |
| `alpha` | `number` | `1` | 0 ≤ alpha ≤ 1 |
| `zIndex` | `number` | `0` | 整數，值越大越上層 |
| `rotation` | `number` | `0` | 弧度制 |
| `scale.x / y` | `number` | `1` | 可為負值（鏡像） |
| `children` | `LayerNode[]` | `undefined` | 僅 `group` 類型有效 |

---

## 2. Clipping 系統型別

Clipping 系統提供遮罩裁切功能，主要用於轉輪視窗的顯示區域限制。`ClipMask` 節點不直接渲染，而是作為其他圖層的 `clipId` 參照目標。

### 2.1 ClipType

| 類型 | 說明 | 使用場景 |
|------|------|----------|
| `rect` | 矩形遮罩 | 最常見，用於轉輪視窗 |
| `polygon` | 多邊形遮罩 | 不規則形狀裁切 |
| `sprite` | 圖片遮罩 | 使用灰階圖作為 alpha 遮罩 |
| `stencil` | Stencil 遮罩 | GPU stencil buffer，適合複雜形狀 |

```typescript
type ClipType = 'rect' | 'polygon' | 'sprite' | 'stencil';
```

### 2.2 ClipMask

```typescript
interface ClipMask {
  id: string;                                              // UUID，唯一識別碼
  type: ClipType;                                          // 遮罩類型
  rect?: { x: number; y: number; w: number; h: number };  // rect 類型的區域定義
  points?: number[];                                       // polygon 類型的頂點座標 [x1,y1,x2,y2,...]
  texturePath?: string;                                    // sprite 類型的遮罩圖片路徑
}
```

### 2.3 各遮罩類型必填欄位

| 類型 | 必填 | 選填 |
|------|------|------|
| `rect` | `rect` | — |
| `polygon` | `points` | — |
| `sprite` | `texturePath` | — |
| `stencil` | `rect` 或 `points` | — |

> **注意**：`ClipMask` 採用聯合式結構而非 discriminated union，實際上各類型的必填欄位由 runtime 驗證保證。未來可考慮改為嚴格的 discriminated union 以獲得編譯期型別安全。

---

## 3. 轉輪系統型別

轉輪系統支援**每軸可變列數**的完整盤面配置（如 Megaways 風格的 `[3, 4, 5, 4, 3]` 佈局），涵蓋 Symbol 定義、轉動動畫的緩動與彈跳效果。

### 3.1 ReelConfig

轉輪的主要設定檔，描述盤面結構、符號配置與動畫參數。

```typescript
interface ReelConfig {
  id: string;                          // UUID，唯一識別碼
  reels: number;                       // 軸數（columns）
  rowsPerReel: number[];               // 每軸的列數，長度必須等於 reels
  symbolSize: { w: number; h: number };// 單一 Symbol 的尺寸（px）
  symbolGap: { x: number; y: number }; // Symbol 間距（px）
  alignment: 'top' | 'center' | 'bottom'; // 垂直對齊方式
  reelStrips: number[][];              // 每軸的符號帶（symbol ID 陣列）
  symbols: Record<string, SymbolDefinition>; // Symbol 定義映射表
  spinSpeed: number;                   // 轉動速度（px/sec）
  easeConfig: EaseConfig;             // 緩動設定
  bounceConfig: BounceConfig;         // 彈跳設定
}
```

#### 欄位約束

| 欄位 | 約束 |
|------|------|
| `reels` | 正整數，通常 3–6 |
| `rowsPerReel` | 長度必須 === `reels`，每個元素為正整數 |
| `reelStrips[i]` | 長度至少 > `rowsPerReel[i]`，以確保有足夠的滾動空間 |
| `spinSpeed` | 正數 |

### 3.2 SymbolDefinition

定義單一 Symbol 的資源與顯示方式。

```typescript
interface SymbolDefinition {
  id: string;                    // 唯一識別碼
  name: string;                  // 顯示名稱（如「Wild」、「Scatter」）
  type: 'spine' | 'sprite';     // 資源類型
  spinePath?: string;            // Spine 動畫路徑（type === 'spine' 時必填）
  spritePath?: string;           // 靜態圖片路徑（type === 'sprite' 時必填）
  preview?: string;              // 縮圖路徑（用於面板預覽）
}
```

### 3.3 EaseConfig

控制轉輪啟動與停止的緩動效果。

```typescript
interface EaseConfig {
  easeIn: string;     // GSAP 緩動函式名稱，如 'power2.in'
  easeOut: string;    // GSAP 緩動函式名稱，如 'power2.out'
  duration: number;   // 緩動持續時間（秒）
}
```

### 3.4 BounceConfig

控制轉輪停止時的彈跳動畫效果。

```typescript
interface BounceConfig {
  enabled: boolean;    // 是否啟用彈跳
  amplitude: number;   // 彈跳幅度（px）
  count: number;       // 彈跳次數
  damping: number;     // 阻尼係數 0–1（越接近 0 衰減越快）
}
```

### 3.5 GridConfig

通用網格配置，用於更精細的盤面佈局控制。

```typescript
interface GridConfig {
  columns: number;                // 行數
  rowsPerColumn: number[];        // 每行的列數
  cellWidth: number;              // 單元格寬度（px）
  cellHeight: number;             // 單元格高度（px）
  gapX: number;                   // 水平間距（px）
  gapY: number;                   // 垂直間距（px）
  alignment: 'top' | 'center' | 'bottom'; // 垂直對齊
  padding: {                      // 外邊距
    top: number;
    right: number;
    bottom: number;
    left: number;
  };
}
```

### 3.6 ReelConfig 與 GridConfig 的關係

`ReelConfig` 專注於**轉輪邏輯**（符號帶、轉動速度、緩動），而 `GridConfig` 專注於**視覺佈局**（網格尺寸、間距、對齊）。兩者可透過以下方式對應：

| ReelConfig 欄位 | GridConfig 對應 |
|-----------------|----------------|
| `reels` | `columns` |
| `rowsPerReel` | `rowsPerColumn` |
| `symbolSize.w` | `cellWidth` |
| `symbolSize.h` | `cellHeight` |
| `symbolGap.x` | `gapX` |
| `symbolGap.y` | `gapY` |
| `alignment` | `alignment` |

---

## 4. Spine 系統型別

Spine 系統管理骨骼動畫資源的載入與播放狀態。`SpineResource` 描述靜態資源資訊，`SpineActorState` 描述運行時的播放狀態。

### 4.1 SpineResource

描述一組 Spine 動畫資源（骨骼檔 + 圖集 + 貼圖）的靜態資訊。

```typescript
interface SpineResource {
  id: string;            // UUID，唯一識別碼
  name: string;          // 資源顯示名稱
  skelPath: string;      // .skel 或 .json 骨骼檔路徑
  atlasPath: string;     // .atlas 圖集路徑
  pngPaths: string[];    // 所有相關 .png 貼圖路徑
  animations: string[];  // 可用動畫名稱列表
  skins: string[];       // 可用 Skin 列表
  slots: string[];       // Slot 名稱列表
  loaded: boolean;       // 資源是否已載入完成
}
```

### 4.2 SpineActorState

描述一個 Spine 動畫實例的運行時播放狀態。

```typescript
interface SpineActorState {
  resourceId: string;          // 引用的 SpineResource.id
  currentAnimation: string;    // 當前播放的動畫名稱
  currentTrack: number;        // 動畫軌道索引
  loop: boolean;               // 是否循環播放
  speed: number;               // 播放速度倍率
  paused: boolean;             // 是否暫停
  skin: string;                // 當前使用的 Skin
}
```

### 4.3 資源與實例的關係

```
SpineResource（靜態資源）  1 ──→ N  SpineActorState（運行時實例）
       ↑                              ↑
  SpineLoader 載入管理            SpineActor 封裝播放 API
```

同一個 `SpineResource` 可被多個 `SpineActorState` 引用（例如同一角色出現在多個圖層）。`SpineLoader` 負責資源的載入與快取，`SpineActor` 封裝 PixiJS Spine 物件的高階播放 API。

---

## 5. 腳本系統型別

腳本系統允許美術人員以 JSON 定義完整的遊戲演出流程。腳本由一系列**步驟（Step）**組成，支援順序執行與並行執行。

### 5.1 ScriptDefinition

腳本的頂層結構定義。

```typescript
interface ScriptDefinition {
  name: string;             // 腳本名稱（唯一識別）
  version: number;          // 腳本版本號
  description?: string;     // 腳本描述（可選）
  steps: ScriptStep[];      // 步驟列表
}
```

### 5.2 ScriptStep

腳本步驟採用**順序 / 並行**二元結構：

```typescript
type ScriptStep = SequentialStep | ParallelStep;

interface SequentialStep {
  action: ActionType;           // 動作類型
  [key: string]: any;           // 各動作的專屬參數
}

interface ParallelStep {
  parallel: SequentialStep[];   // 並行執行的動作列表
}
```

> **設計說明**：`SequentialStep` 採用 index signature 而非嚴格的 discriminated union，是為了讓 JSON 格式更簡潔。型別安全由 `ActionDefinition` 聯合型別在程式碼層確保。

### 5.3 ActionType

所有支援的動作類型列舉：

```typescript
type ActionType =
  | 'spin'             // 開始轉動
  | 'stopReel'         // 停止指定軸
  | 'stopAll'          // 停止所有軸
  | 'playSpine'        // 播放 Spine 動畫
  | 'mixSpine'         // 混合 Spine 動畫
  | 'setSkin'          // 切換 Spine Skin
  | 'setLayerVisible'  // 設定圖層可見性
  | 'setLayerAlpha'    // 設定圖層透明度
  | 'tween'            // GSAP 補間動畫
  | 'wait'             // 等待指定時間
  | 'waitEvent'        // 等待事件觸發
  | 'emit'             // 發送事件
  | 'callScript';      // 呼叫另一個腳本
```

### 5.4 各動作介面

每種動作類型皆有對應的介面定義，提供強型別支援：

```typescript
// 轉輪控制
interface SpinAction       { action: 'spin';       duration: number; speed?: number }
interface StopReelAction   { action: 'stopReel';   reel: number; delay?: number; result?: number[] }
interface StopAllAction    { action: 'stopAll';     delay?: number }

// Spine 動畫控制
interface PlaySpineAction  { action: 'playSpine';  actor: string; anim: string; track?: number; loop?: boolean }
interface MixSpineAction   { action: 'mixSpine';   actor: string; anim: string; mix: number }
interface SetSkinAction    { action: 'setSkin';     actor: string; skin: string }

// 圖層控制
interface SetLayerVisibleAction { action: 'setLayerVisible'; layer: string; visible: boolean }
interface SetLayerAlphaAction   { action: 'setLayerAlpha';   layer: string; alpha: number }

// 通用動畫與流程控制
interface TweenAction      { action: 'tween';      target: string; props: Record<string, number>; duration: number; ease?: string }
interface WaitAction        { action: 'wait';       duration: number }
interface WaitEventAction   { action: 'waitEvent';  event: string }
interface EmitAction        { action: 'emit';       event: string; payload?: any }
interface CallScriptAction  { action: 'callScript'; name: string }
```

### 5.5 ActionDefinition 聯合型別

所有動作介面的聯合型別，用於程式碼層的型別安全檢查：

```typescript
type ActionDefinition =
  | SpinAction
  | StopReelAction
  | StopAllAction
  | PlaySpineAction
  | MixSpineAction
  | SetSkinAction
  | SetLayerVisibleAction
  | SetLayerAlphaAction
  | TweenAction
  | WaitAction
  | WaitEventAction
  | EmitAction
  | CallScriptAction;
```

### 5.6 ScriptState

腳本執行狀態機：

```typescript
type ScriptState = 'idle' | 'running' | 'paused' | 'aborted' | 'completed';
```

| 狀態 | 說明 | 可轉換至 |
|------|------|----------|
| `idle` | 尚未執行 | `running` |
| `running` | 執行中 | `paused`、`aborted`、`completed` |
| `paused` | 暫停中 | `running`、`aborted` |
| `aborted` | 已中止 | `idle` |
| `completed` | 已完成 | `idle` |

### 5.7 腳本 JSON 範例

```json
{
  "name": "bigwin_演出",
  "version": 1,
  "description": "Big Win 完整流程",
  "steps": [
    { "action": "spin", "duration": 2 },
    { "action": "stopReel", "reel": 0, "result": [1, 2, 3] },
    { "action": "stopReel", "reel": 1, "delay": 0.3, "result": [1, 2, 3] },
    { "action": "stopReel", "reel": 2, "delay": 0.3, "result": [1, 2, 3] },
    { "action": "stopReel", "reel": 3, "delay": 0.3, "result": [1, 2, 3] },
    { "action": "stopReel", "reel": 4, "delay": 0.3, "result": [1, 2, 3] },
    { "action": "wait", "duration": 0.5 },
    {
      "parallel": [
        { "action": "playSpine", "actor": "bigwin_fx", "anim": "start", "loop": false },
        { "action": "setLayerVisible", "layer": "dimmer", "visible": true },
        { "action": "tween", "target": "dimmer", "props": { "alpha": 0.7 }, "duration": 0.3 }
      ]
    },
    { "action": "waitEvent", "event": "bigwin_complete" },
    { "action": "setLayerVisible", "layer": "dimmer", "visible": false }
  ]
}
```

---

## 6. 專案 I/O 型別

專案以 `.zip` 格式匯出 / 匯入，包含所有資源檔案與狀態資訊。`ProjectManifest` 為 zip 內的根描述檔，`ProjectState` 為完整的應用程式狀態快照。

### 6.1 ProjectManifest

專案描述檔，儲存於 zip 根目錄的 `manifest.json`。

```typescript
interface ProjectManifest {
  version: string;           // Schema 版本號（用於向前相容與 migration）
  name: string;              // 專案名稱
  createdAt: string;         // 建立時間（ISO 8601）
  updatedAt: string;         // 最後更新時間（ISO 8601）
  preview: {
    width: number;           // 畫布寬度（px）
    height: number;          // 畫布高度（px）
    backgroundColor: string; // 背景色（CSS 色碼，如 '#1a1a2e'）
    zoom: number;            // 縮放倍率
  };
  assets: AssetEntry[];      // 資源清單
}
```

### 6.2 AssetEntry

描述 zip 內的單一資源檔案。

```typescript
interface AssetEntry {
  id: string;                // UUID，唯一識別碼
  type: 'spine' | 'sprite' | 'script' | 'config'; // 資源類型
  relativePath: string;      // zip 內的相對路徑
  originalName: string;      // 原始檔案名稱
}
```

### 6.3 ProjectState

完整的應用程式狀態快照，儲存於 zip 內的 `state.json`。

```typescript
interface ProjectState {
  manifest: ProjectManifest;
  layers: LayerNode[];
  clips: ClipMask[];
  reelConfigs: ReelConfig[];
  spineResources: SpineResource[];
  scripts: ScriptDefinition[];
  preview: PreviewState;
}
```

### 6.4 ZIP 目錄結構

```
project.zip/
├── manifest.json              ← ProjectManifest
├── state.json                 ← ProjectState（layers, clips, reelConfigs, scripts, preview）
└── assets/
    ├── spine/
    │   ├── hero.skel
    │   ├── hero.atlas
    │   └── hero.png
    ├── sprites/
    │   └── bg.png
    └── scripts/
        └── bigwin.json
```

### 6.5 匯出 / 匯入流程

```
匯出流程：
  Zustand Store → ProjectState → JSON 序列化 → JSZip 打包 → .zip 下載

匯入流程：
  .zip 上傳 → JSZip 解壓 → manifest.json 版本檢查
    → migration（若需要）→ state.json 解析 → Zustand Store 還原
    → assets/ 載入至記憶體 → PixiJS Managers 初始化
```

---

## 7. Store Slice 型別

Zustand Store 採用 **Slice 模式** 拆分，每個 Slice 管理獨立的狀態領域。以下定義各 Slice 的 State 與 Action 介面。

### 7.1 LayerStoreState

管理圖層樹的 CRUD 與選取狀態。

```typescript
interface LayerStoreState {
  layers: LayerNode[];
  selectedLayerId: string | null;

  addLayer: (node: LayerNode) => void;
  removeLayer: (id: string) => void;
  moveLayer: (id: string, newIndex: number) => void;
  updateLayer: (id: string, props: Partial<LayerNode>) => void;
  selectLayer: (id: string | null) => void;
}
```

### 7.2 SpineStoreState

管理 Spine 資源與動畫實例狀態。

```typescript
interface SpineStoreState {
  resources: SpineResource[];
  actors: Map<string, SpineActorState>;

  addResource: (resource: SpineResource) => void;
  removeResource: (id: string) => void;
  updateActor: (id: string, state: Partial<SpineActorState>) => void;
}
```

### 7.3 ReelStoreState

管理轉輪設定與轉動狀態。

```typescript
interface ReelStoreState {
  configs: ReelConfig[];
  currentResult: number[][] | null;
  spinning: boolean;
  activeConfig: string | null;

  setConfig: (config: ReelConfig) => void;
  setResult: (result: number[][]) => void;
  setSpinning: (spinning: boolean) => void;
}
```

### 7.4 ScriptStoreState

管理腳本載入、執行進度與狀態機。

```typescript
interface ScriptStoreState {
  scripts: ScriptDefinition[];
  activeScript: string | null;
  state: ScriptState;
  currentStep: number;
  totalSteps: number;

  loadScript: (script: ScriptDefinition) => void;
  setActive: (name: string | null) => void;
  updateProgress: (step: number, total: number) => void;
  setState: (state: ScriptState) => void;
}
```

### 7.5 PreviewStoreState

管理預覽畫布的顯示設定。

```typescript
interface PreviewStoreState {
  width: number;
  height: number;
  backgroundColor: string;
  zoom: number;
  panX: number;
  panY: number;

  setResolution: (w: number, h: number) => void;
  setBackgroundColor: (color: string) => void;
  setZoom: (zoom: number) => void;
  setPan: (x: number, y: number) => void;
}
```

### 7.6 UIStoreState

管理 UI 面板的開關與尺寸狀態。

```typescript
interface UIStoreState {
  leftPanelOpen: boolean;
  rightPanelOpen: boolean;
  rightPanelWidth: number;
  activeRightTab: string;

  toggleLeftPanel: () => void;
  toggleRightPanel: () => void;
  setRightPanelWidth: (w: number) => void;
  setActiveRightTab: (tab: string) => void;
}
```

### 7.7 ProjectStoreState

管理專案層級的元資訊與狀態追蹤。

```typescript
interface ProjectStoreState {
  projectName: string;
  dirty: boolean;
  lastSaved: string | null;

  setProjectName: (name: string) => void;
  markDirty: () => void;
  markClean: () => void;
}
```

### 7.8 Store Slice 總覽

| Slice | 檔案 | 主要職責 |
|-------|------|----------|
| `LayerStoreState` | `store/layerStore.ts` | 圖層樹的 CRUD、選取、排序 |
| `SpineStoreState` | `store/spineStore.ts` | Spine 資源管理、動畫實例狀態 |
| `ReelStoreState` | `store/reelStore.ts` | 轉輪設定、轉動狀態、結果盤面 |
| `ScriptStoreState` | `store/scriptStore.ts` | 腳本載入、執行進度、狀態機 |
| `PreviewStoreState` | `store/previewStore.ts` | 畫布解析度、背景色、縮放與平移 |
| `UIStoreState` | `store/uiStore.ts` | 面板開關、尺寸、活動分頁 |
| `ProjectStoreState` | `store/projectStore.ts` | 專案名稱、dirty 狀態、儲存時間 |

---

## 8. 型別擴充指南

### 8.1 新增圖層類型

當需要支援新的圖層類型（例如 `video` 影片圖層）時，依照以下步驟：

**步驟 1**：擴充 `LayerType` 聯合型別

```typescript
// 修改前
type LayerType = 'spine' | 'sprite' | 'reel' | 'clip' | 'group';

// 修改後
type LayerType = 'spine' | 'sprite' | 'reel' | 'clip' | 'group' | 'video';
```

**步驟 2**：定義新類型的專屬資料介面

```typescript
interface VideoLayerData {
  videoPath: string;
  autoplay: boolean;
  loop: boolean;
  volume: number;
}
```

**步驟 3**：在 `LayerNode` 中新增可選欄位

```typescript
interface LayerNode {
  // ... 既有欄位 ...
  videoData?: VideoLayerData;
}
```

**步驟 4**：在 `LayerManager` 中新增對應的渲染邏輯

```typescript
// LayerManager.ts
private createDisplayObject(node: LayerNode): DisplayObject {
  switch (node.type) {
    // ... 既有類型 ...
    case 'video':
      return this.createVideoLayer(node.videoData!);
  }
}
```

### 8.2 新增腳本動作類型

當需要支援新的腳本動作（例如 `playSound` 音效播放）時：

**步驟 1**：擴充 `ActionType` 聯合型別

```typescript
type ActionType =
  | 'spin' | 'stopReel' | 'stopAll'
  // ... 既有類型 ...
  | 'playSound';  // 新增
```

**步驟 2**：定義新動作的介面

```typescript
interface PlaySoundAction {
  action: 'playSound';
  src: string;
  volume?: number;
  loop?: boolean;
}
```

**步驟 3**：加入 `ActionDefinition` 聯合型別

```typescript
type ActionDefinition =
  | SpinAction
  // ... 既有類型 ...
  | PlaySoundAction;  // 新增
```

**步驟 4**：實作 Action Handler 並註冊

```typescript
// actions/PlaySoundAction.ts
const playSoundHandler: ActionHandler = async (params) => {
  const audio = new Audio(params.src);
  audio.volume = params.volume ?? 1;
  audio.loop = params.loop ?? false;
  await audio.play();
};

// actions/index.ts
ActionRegistry.register('playSound', playSoundHandler);
```

### 8.3 擴充專案 Schema

當需要變更 `state.json` 或 `manifest.json` 的結構時：

**步驟 1**：在 `ProjectManifest.version` 中遞增版本號

```typescript
// 從 "1.0.0" → "1.1.0"
```

**步驟 2**：在 `ProjectImporter` 中新增 migration 邏輯

```typescript
// io/ProjectImporter.ts
const migrations: Record<string, (state: any) => any> = {
  '1.0.0→1.1.0': (state) => {
    // 為所有 LayerNode 補上新增的 videoData 欄位
    state.layers = state.layers.map((layer: any) => ({
      ...layer,
      videoData: undefined,
    }));
    return state;
  },
};
```

**步驟 3**：確保匯入時自動偵測版本並依序執行 migration

```typescript
function migrateState(state: any, fromVersion: string, toVersion: string): ProjectState {
  let current = fromVersion;
  while (current !== toVersion) {
    const key = `${current}→${nextVersion(current)}`;
    if (migrations[key]) {
      state = migrations[key](state);
    }
    current = nextVersion(current);
  }
  return state as ProjectState;
}
```

> **原則**：永遠只往前加欄位、不刪除既有欄位，確保舊版專案檔可在新版工具中正確載入。

---

> **下一份文件**：`04-ui-layout.md` — UI 佈局與面板設計規格
