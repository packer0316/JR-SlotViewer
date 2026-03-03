# Slot Previewer — 狀態管理

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置閱讀**：`01-architecture-overview.md`、`03-data-models.md`
> **適用對象**：開發人員、技術主管

---

## 1. 狀態管理架構

### 1.1 核心原則

Slot Previewer 的狀態管理遵循 **Single Source of Truth** 模式——所有業務狀態皆存放在 Zustand Store 中，PixiJS Scene Graph **純粹為渲染產物**，不儲存任何業務狀態。

| 原則 | 說明 |
|------|------|
| **Zustand 為唯一真相來源** | 所有 UI 操作與業務邏輯的狀態皆由 Zustand Store 管理，任何元件或模組都從 Store 讀取 |
| **Immer 不可變更新** | 透過 Immer middleware 以可變語法撰寫不可變更新，降低深層巢狀物件的操作複雜度 |
| **領域切片（Domain Slice）** | 每個業務領域擁有獨立的 Store Slice，避免單體式 Store 造成的耦合與效能問題 |
| **精細化 Selector** | React 元件透過精細的 selector 僅訂閱所需欄位，避免不必要的重新渲染 |
| **Manager 被動訂閱** | PixiJS Managers 透過 `store.subscribe()` 監聽 Store 變化，被動同步至 Scene Graph |

### 1.2 資料流

```
使用者操作 → React UI → Zustand Action → Store 狀態更新
                                              │
                                     store.subscribe()
                                              │
                                       PixiJS Manager
                                              │
                                    Scene Graph 更新
                                              │
                                       Canvas 渲染
```

**單向資料流**保證了狀態的可預測性與可追蹤性：

1. **使用者操作**（點擊按鈕、拖曳圖層、輸入數值）觸發 React 事件
2. **React 元件**呼叫 Zustand Store 的 action 方法
3. **Zustand Store** 透過 Immer 執行不可變更新
4. **PixiJS Manager** 透過 `store.subscribe()` 收到狀態變化通知
5. **Manager** 將新狀態同步至 PixiJS Scene Graph
6. **PixiJS Ticker** 驅動 Canvas 渲染最終畫面

> **關鍵規則**：React UI 永遠不直接操作 PixiJS 物件；PixiJS Manager 永遠不直接修改 React 狀態。Store 是兩者之間的唯一橋梁。

---

## 2. Store Slice 設計

### 2.1 Slice 總覽

| Slice | 檔案 | 職責 | 訂閱者（Manager） | 消費者（UI） |
|-------|------|------|-------------------|-------------|
| `layerStore` | `store/layerStore.ts` | 圖層樹 CRUD、選取、排序 | LayerManager | LayerPanel, PropertyPanel |
| `spineStore` | `store/spineStore.ts` | Spine 資源管理、動畫實例狀態 | SpineLoader | SpinePanel, LayerManager |
| `reelStore` | `store/reelStore.ts` | 轉輪設定、轉動狀態、結果盤面 | ReelEngine | ReelPanel, ReelGridEditor |
| `scriptStore` | `store/scriptStore.ts` | 腳本載入、執行進度、狀態機 | ScriptRunner | ScriptPanel |
| `previewStore` | `store/previewStore.ts` | 畫布解析度、背景色、縮放與平移 | PixiApp, ViewportManager | Toolbar, StatusBar |
| `uiStore` | `store/uiStore.ts` | 面板開關、尺寸、活動分頁 | Layout 元件 | 所有面板 |
| `projectStore` | `store/projectStore.ts` | 專案名稱、dirty 狀態、儲存時間 | Title bar, Auto-save | ProjectExporter/Importer, Toolbar |

### 2.2 layerStore

管理圖層樹的完整狀態——包含圖層節點的 CRUD、選取與排序。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `layers` | `LayerNode[]` | 圖層樹（扁平化或巢狀結構） |
| `selectedLayerId` | `string \| null` | 當前選取的圖層 ID |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `addLayer` | `(node: LayerNode) => void` | 新增圖層節點 |
| `removeLayer` | `(id: string) => void` | 刪除圖層及其子節點 |
| `moveLayer` | `(id: string, newIndex: number) => void` | 移動圖層至新的排序位置 |
| `updateLayer` | `(id: string, props: Partial<LayerNode>) => void` | 更新圖層屬性 |
| `selectLayer` | `(id: string \| null) => void` | 選取或取消選取圖層 |
| `reorderLayers` | `(orderedIds: string[]) => void` | 批次重新排序圖層 |

#### 訂閱與消費

- **訂閱者**：`LayerManager`——監聽 `layers` 變化，同步至 PixiJS Scene Graph
- **消費者**：`LayerPanel`（顯示圖層列表）、`PropertyPanel`（編輯選取圖層屬性）

---

### 2.3 spineStore

管理 Spine 動畫資源的載入狀態與運行時實例。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `resources` | `SpineResource[]` | 已載入的 Spine 資源列表 |
| `actors` | `Map<string, SpineActorState>` | 活動中的 Spine 動畫實例 |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `addResource` | `(resource: SpineResource) => void` | 註冊新的 Spine 資源 |
| `removeResource` | `(id: string) => void` | 移除 Spine 資源與相關快取 |
| `updateActor` | `(id: string, state: Partial<SpineActorState>) => void` | 更新動畫實例狀態 |
| `clearAll` | `() => void` | 清除所有資源與實例 |

#### 訂閱與消費

- **訂閱者**：`SpineLoader`——監聽資源列表變化，同步快取
- **消費者**：`SpinePanel`（資源瀏覽與播放控制）、`LayerManager`（建立 Spine 圖層）

---

### 2.4 reelStore

管理轉輪盤面設定、轉動狀態與開獎結果。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `configs` | `ReelConfig[]` | 所有盤面設定檔 |
| `activeConfig` | `string \| null` | 當前啟用的盤面設定 ID |
| `currentResult` | `number[][] \| null` | 當前的開獎結果 |
| `spinning` | `boolean` | 轉輪是否正在旋轉 |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `setConfig` | `(config: ReelConfig) => void` | 設定或替換盤面設定 |
| `updateConfig` | `(id: string, partial: Partial<ReelConfig>) => void` | 部分更新盤面設定 |
| `setResult` | `(result: number[][]) => void` | 設定開獎結果 |
| `setSpinning` | `(spinning: boolean) => void` | 設定轉動狀態 |
| `addConfig` | `(config: ReelConfig) => void` | 新增盤面設定 |
| `removeConfig` | `(id: string) => void` | 移除盤面設定 |

#### 訂閱與消費

- **訂閱者**：`ReelEngine`——監聽設定變化重建佈局、監聯轉動指令
- **消費者**：`ReelPanel`（轉輪控制面板）、`ReelGridEditor`（視覺化盤面編輯器）

---

### 2.5 scriptStore

管理腳本的載入、選取與執行狀態機。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `scripts` | `ScriptDefinition[]` | 已載入的腳本列表 |
| `activeScript` | `string \| null` | 當前啟用的腳本名稱 |
| `state` | `ScriptState` | 執行狀態（`idle` / `running` / `paused` / `aborted` / `completed`） |
| `currentStep` | `number` | 當前執行到的步驟索引 |
| `totalSteps` | `number` | 總步驟數 |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `loadScript` | `(script: ScriptDefinition) => void` | 載入腳本 |
| `removeScript` | `(name: string) => void` | 移除腳本 |
| `setActive` | `(name: string \| null) => void` | 設定啟用的腳本 |
| `updateProgress` | `(step: number, total: number) => void` | 更新執行進度 |
| `setState` | `(state: ScriptState) => void` | 更新執行狀態 |

#### 訂閱與消費

- **訂閱者**：`ScriptRunner`——監聽啟用腳本與控制指令
- **消費者**：`ScriptPanel`（腳本列表、播放控制、進度條）

---

### 2.6 previewStore

管理預覽畫布的顯示參數。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `width` | `number` | 畫布寬度（px） |
| `height` | `number` | 畫布高度（px） |
| `backgroundColor` | `string` | 背景色（CSS 色碼） |
| `zoom` | `number` | 縮放倍率 |
| `panX` | `number` | 水平平移量 |
| `panY` | `number` | 垂直平移量 |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `setResolution` | `(w: number, h: number) => void` | 設定畫布解析度 |
| `setBackgroundColor` | `(color: string) => void` | 設定背景色 |
| `setZoom` | `(zoom: number) => void` | 設定縮放倍率 |
| `setPan` | `(x: number, y: number) => void` | 設定平移量 |
| `resetView` | `() => void` | 重置視角至預設值 |

#### 訂閱與消費

- **訂閱者**：`PixiApp`（監聽解析度變化觸發 resize）、`ViewportManager`（監聽縮放與平移）
- **消費者**：`Toolbar`（解析度選單、縮放控制）、`StatusBar`（顯示當前縮放比例與座標）

---

### 2.7 uiStore

管理 UI 佈局面板的開關狀態與尺寸。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `leftPanelOpen` | `boolean` | 左側面板是否展開 |
| `rightPanelOpen` | `boolean` | 右側面板是否展開 |
| `rightPanelWidth` | `number` | 右側面板寬度（px） |
| `activeRightTab` | `string` | 右側面板活動中的分頁 ID |
| `bottomPanelOpen` | `boolean` | 底部面板是否展開 |
| `bottomPanelHeight` | `number` | 底部面板高度（px） |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `toggleLeftPanel` | `() => void` | 切換左側面板開關 |
| `toggleRightPanel` | `() => void` | 切換右側面板開關 |
| `setRightPanelWidth` | `(w: number) => void` | 設定右側面板寬度 |
| `setActiveRightTab` | `(tab: string) => void` | 切換右側面板活動分頁 |

#### 訂閱與消費

- **訂閱者**：Layout 元件——根據面板狀態重新計算佈局
- **消費者**：所有面板元件

---

### 2.8 projectStore

管理專案層級的元資訊與未儲存變更追蹤。

#### State

| 欄位 | 型別 | 說明 |
|------|------|------|
| `projectName` | `string` | 專案名稱 |
| `dirty` | `boolean` | 是否有未儲存的變更 |
| `lastSaved` | `string \| null` | 最後儲存時間（ISO 8601） |
| `version` | `string` | Schema 版本號 |

#### Actions

| Action | 簽章 | 說明 |
|--------|------|------|
| `setProjectName` | `(name: string) => void` | 設定專案名稱 |
| `markDirty` | `() => void` | 標記有未儲存變更 |
| `markClean` | `() => void` | 標記所有變更已儲存 |
| `setLastSaved` | `(time: string) => void` | 更新最後儲存時間 |

#### 訂閱與消費

- **訂閱者**：Title bar（顯示專案名稱與 dirty 指示器）、Auto-save 邏輯
- **消費者**：`ProjectExporter` / `ProjectImporter`（匯出 / 匯入）、`Toolbar`（專案操作按鈕）

---

## 3. Immer Middleware 使用方式

### 3.1 為何使用 Immer

Zustand 預設要求以不可變方式更新狀態（如展開運算子 `...state`）。當狀態結構較深時（例如圖層樹的巢狀子節點），手動撰寫不可變更新極為繁瑣且容易出錯。Immer middleware 允許以**可變語法**撰寫更新邏輯，Immer 會在背後自動產生不可變的新物件。

### 3.2 建立 Store 的標準範式

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface LayerStoreState {
  layers: LayerNode[];
  selectedLayerId: string | null;

  addLayer: (node: LayerNode) => void;
  removeLayer: (id: string) => void;
  updateLayer: (id: string, props: Partial<LayerNode>) => void;
  selectLayer: (id: string | null) => void;
}

const useLayerStore = create(
  immer<LayerStoreState>((set) => ({
    layers: [],
    selectedLayerId: null,

    addLayer: (node) =>
      set((state) => {
        state.layers.push(node);
      }),

    removeLayer: (id) =>
      set((state) => {
        state.layers = state.layers.filter((l) => l.id !== id);
        if (state.selectedLayerId === id) {
          state.selectedLayerId = null;
        }
      }),

    updateLayer: (id, props) =>
      set((state) => {
        const layer = state.layers.find((l) => l.id === id);
        if (layer) Object.assign(layer, props);
      }),

    selectLayer: (id) =>
      set((state) => {
        state.selectedLayerId = id;
      }),
  }))
);
```

### 3.3 注意事項

| 項目 | 說明 |
|------|------|
| **不可在 `set()` 外使用草稿** | Immer 的 draft 僅在 `set()` callback 內有效，不可將 draft 傳出 |
| **避免回傳值** | `set()` callback 不應有 `return` 語句（Immer 會自動產生新狀態） |
| **深層巢狀安全** | 可直接 `state.layers[0].children[2].visible = false`，Immer 會正確追蹤 |
| **陣列操作** | 可直接使用 `push`、`splice`、`sort` 等可變方法 |

---

## 4. Selector 最佳實踐

### 4.1 精細化 Selector

React 元件應透過 selector 僅訂閱所需的**最小狀態片段**，避免無關狀態變化觸發重新渲染：

```typescript
// ✅ 正確：僅訂閱 selectedLayerId
const selectedId = useLayerStore((s) => s.selectedLayerId);

// ✅ 正確：僅訂閱 layers 陣列
const layers = useLayerStore((s) => s.layers);

// ❌ 錯誤：訂閱整個 Store，任何欄位變化都會觸發重新渲染
const store = useLayerStore();
```

### 4.2 使用 useShallow 進行物件選取

當 selector 回傳物件或陣列時，預設的嚴格相等比較 (`===`) 會在每次 `set()` 呼叫後認為值已改變。使用 `useShallow` 進行淺比較以避免不必要的重新渲染：

```typescript
import { useShallow } from 'zustand/react/shallow';

// ✅ 正確：淺比較多個欄位
const { width, height, zoom } = usePreviewStore(
  useShallow((s) => ({
    width: s.width,
    height: s.height,
    zoom: s.zoom,
  }))
);
```

### 4.3 衍生資料 Memoize

對於需要計算的衍生資料，使用 `useMemo` 避免每次渲染都重新計算：

```typescript
const layers = useLayerStore((s) => s.layers);
const selectedId = useLayerStore((s) => s.selectedLayerId);

const selectedLayer = useMemo(
  () => layers.find((l) => l.id === selectedId),
  [layers, selectedId]
);
```

### 4.4 Selector 規則總覽

| 規則 | 說明 |
|------|------|
| 僅選取所需欄位 | 避免訂閱整個 Store 或不需要的欄位 |
| 物件選取用 `useShallow` | 防止淺層物件的假陽性重新渲染 |
| 衍生資料用 `useMemo` | 避免重複計算 |
| 一個元件多個 selector | 分開呼叫 `useStore`，各自獨立訂閱 |
| 不在 selector 中產生副作用 | Selector 必須是純函式 |

---

## 5. Manager 訂閱模式

### 5.1 訂閱機制

PixiJS Managers 在初始化時透過 `store.subscribe()` 訂閱相關 Store Slice 的變化。當 Store 狀態更新時，Manager 會收到新舊狀態通知，執行差異比對後更新 PixiJS Scene Graph。

```typescript
import { useLayerStore } from '../store/layerStore';
import { shallow } from 'zustand/shallow';

class LayerManager {
  private unsubscribe: () => void;

  init(): void {
    this.unsubscribe = useLayerStore.subscribe(
      (state) => state.layers,
      (layers, prevLayers) => {
        this.syncFromStore(layers, prevLayers);
      },
      { equalityFn: shallow }
    );
  }

  destroy(): void {
    this.unsubscribe();
  }

  private syncFromStore(layers: LayerNode[], prevLayers: LayerNode[]): void {
    // 差異比對並更新 PixiJS Scene Graph
  }
}
```

### 5.2 訂閱流程圖

```
┌──────────────────────────────────────────────────────────────┐
│                     Manager 訂閱流程                          │
│                                                              │
│  Manager.init()                                              │
│       │                                                      │
│       ▼                                                      │
│  store.subscribe(selector, callback, { equalityFn })         │
│       │                                                      │
│       ▼                                                      │
│  Store 狀態變化                                               │
│       │                                                      │
│       ▼                                                      │
│  selector 擷取關注的 Slice                                     │
│       │                                                      │
│       ▼                                                      │
│  equalityFn 比較新舊值                                         │
│       │                                                      │
│       ├── 相同 → 跳過                                         │
│       └── 不同 → 呼叫 callback(newValue, prevValue)           │
│                    │                                         │
│                    ▼                                         │
│              syncFromStore()                                  │
│                    │                                         │
│                    ▼                                         │
│           更新 PixiJS Scene Graph                              │
│                                                              │
│  Manager.destroy()                                           │
│       │                                                      │
│       ▼                                                      │
│  unsubscribe() — 移除監聽                                      │
└──────────────────────────────────────────────────────────────┘
```

### 5.3 各 Manager 訂閱對照

| Manager | 訂閱的 Store | 監聽的 Slice | 觸發行為 |
|---------|-------------|-------------|---------|
| `LayerManager` | `layerStore` | `layers` | 同步圖層樹至 Scene Graph |
| `SpineLoader` | `spineStore` | `resources` | 同步資源快取 |
| `ReelEngine` | `reelStore` | `configs`, `activeConfig`, `currentResult`, `spinning` | 重建轉輪佈局、控制轉動 |
| `ScriptRunner` | `scriptStore` | `activeScript`, `state` | 載入並執行腳本 |
| `PixiApp` | `previewStore` | `width`, `height` | 調整 Renderer 尺寸 |
| `ViewportManager` | `previewStore` | `zoom`, `panX`, `panY` | 更新 Stage 變換矩陣 |

### 5.4 注意事項

| 項目 | 說明 |
|------|------|
| **務必取消訂閱** | `destroy()` 時呼叫 `unsubscribe()`，防止記憶體洩漏 |
| **使用 `equalityFn`** | 使用 `shallow` 避免無意義的重複同步 |
| **不可反向寫入** | Manager 不應在 `subscribe` callback 中再次呼叫 `set()`，避免無限迴圈 |
| **批次處理** | 若同一幀內 Store 連續更新多次，考慮以 `requestAnimationFrame` 合併處理 |

---

## 6. 狀態持久化策略

### 6.1 匯出（Serialize）

專案匯出時，將所有 Store 狀態序列化為 JSON，連同資源檔案一併打包為 `.zip`：

```
匯出流程：
  ┌────────────────┐
  │ Zustand Stores │
  │ (全部 Slice)    │
  └───────┬────────┘
          │ 讀取各 Store 的 getState()
          ▼
  ┌────────────────┐
  │ ProjectState   │
  │ (JSON 物件)     │
  └───────┬────────┘
          │ JSON.stringify()
          ▼
  ┌────────────────┐
  │ state.json     │ + manifest.json + assets/
  └───────┬────────┘
          │ JSZip 打包
          ▼
  ┌────────────────┐
  │  project.zip   │
  └────────────────┘
```

### 6.2 匯入（Deserialize）

匯入 `.zip` 時，解壓並還原所有 Store 狀態：

```
匯入流程：
  ┌────────────────┐
  │  project.zip   │
  └───────┬────────┘
          │ JSZip 解壓
          ▼
  ┌────────────────┐
  │ manifest.json  │ → 版本檢查 + migration
  └───────┬────────┘
          │
          ▼
  ┌────────────────┐
  │ state.json     │ → JSON.parse()
  └───────┬────────┘
          │ 還原至各 Store
          ▼
  ┌────────────────┐
  │ Zustand Stores │ → 觸發所有 Manager 重新同步
  └───────┬────────┘
          │
          ▼
  ┌────────────────┐
  │ assets/        │ → 載入至記憶體（Spine、圖片等）
  └────────────────┘
```

### 6.3 持久化規則

| 規則 | 說明 |
|------|------|
| **無 LocalStorage 自動儲存** | 依使用者需求，僅透過 `.zip` 匯出進行持久化 |
| **Dirty 標記** | 任何 Store 變更後自動標記 `projectStore.dirty = true` |
| **離開提醒** | 當 `dirty === true` 時，`beforeunload` 事件會提示使用者尚有未儲存的變更 |
| **Schema 版本化** | 匯出的 `manifest.json` 包含 `version` 欄位，匯入時進行版本比對與 migration |

### 6.4 Dirty 標記實作

```typescript
function subscribeDirtyTracking(): void {
  const stores = [
    useLayerStore,
    useSpineStore,
    useReelStore,
    useScriptStore,
    usePreviewStore,
  ];

  stores.forEach((store) => {
    store.subscribe(() => {
      useProjectStore.getState().markDirty();
    });
  });
}

window.addEventListener('beforeunload', (e) => {
  if (useProjectStore.getState().dirty) {
    e.preventDefault();
  }
});
```

---

## 7. 跨 Store 協調

### 7.1 問題場景

某些操作會牽涉多個 Store 的狀態變更。例如：

| 操作 | 涉及的 Store | 說明 |
|------|-------------|------|
| 刪除含 Spine 的圖層 | `layerStore` + `spineStore` | 需同時移除圖層節點與 Spine actor |
| 匯入專案 | 所有 Store | 需依序還原所有 Store 狀態 |
| 重設專案 | 所有 Store | 需將所有 Store 重置為初始值 |
| 刪除轉輪設定 | `reelStore` + `layerStore` | 需同時移除設定與引用該設定的轉輪圖層 |

### 7.2 組合動作（Composite Actions）

對於跨 Store 操作，建立**組合動作函式**統一管理：

```typescript
function deleteLayerWithCleanup(layerId: string): void {
  const layer = useLayerStore.getState().layers.find((l) => l.id === layerId);

  if (!layer) return;

  if (layer.type === 'spine' && layer.spineData) {
    const actorId = layerId;
    useSpineStore.getState().updateActor(actorId, { paused: true });
  }

  if (layer.type === 'reel' && layer.reelData) {
    // 轉輪圖層可能需要停止轉動
    useReelStore.getState().setSpinning(false);
  }

  useLayerStore.getState().removeLayer(layerId);
}
```

### 7.3 協調規則

| 規則 | 說明 |
|------|------|
| **Store 之間禁止直接 import** | Store A 不可 import Store B 的 hook 或 module，避免循環依賴 |
| **使用 `getState()` 讀取外部 Store** | 組合動作中透過 `store.getState()` 存取其他 Store 的狀態 |
| **EventBus 鬆耦合** | 當組合邏輯複雜時，改用 EventBus 發送事件，由各 Store 自行響應 |
| **組合動作集中管理** | 所有跨 Store 操作集中於 `src/store/actions/` 目錄 |

### 7.4 EventBus 輔助模式

```typescript
// 發送端
eventBus.emit('layer:deleted', { layerId, layerType: 'spine', spineData });

// spineStore 側監聽
eventBus.on('layer:deleted', ({ layerId, layerType, spineData }) => {
  if (layerType === 'spine') {
    useSpineStore.getState().updateActor(layerId, { paused: true });
  }
});
```

> **注意**：EventBus 僅用於 Manager 之間以及跨 Store 的鬆耦合通訊。React UI 元件不應直接訂閱 EventBus 事件。

---

## 8. DevTools 整合

### 8.1 Zustand DevTools Middleware

透過 `devtools` middleware 將 Store 狀態接入 **Redux DevTools** 瀏覽器擴充套件，方便開發期間即時檢查與追蹤狀態變化：

```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

const useLayerStore = create(
  devtools(
    immer<LayerStoreState>((set) => ({
      // ... state & actions
    })),
    {
      name: 'LayerStore',
      enabled: import.meta.env.DEV,
    }
  )
);
```

### 8.2 DevTools 功能

| 功能 | 說明 |
|------|------|
| **狀態快照** | 即時查看各 Store 的完整狀態樹 |
| **時間旅行** | 回到任意歷史狀態點，方便除錯 |
| **Action 追蹤** | 記錄每次 action 呼叫及其引起的狀態差異 |
| **匯出 / 匯入狀態** | 將狀態快照匯出為 JSON，可在另一台裝置上匯入還原 |

### 8.3 生產環境禁用

透過 `enabled: import.meta.env.DEV` 確保 DevTools middleware 僅在開發模式下啟用，**不會影響**生產環境的效能與套件大小。

### 8.4 Middleware 堆疊順序

當同時使用多個 middleware 時，注意堆疊順序（由外而內）：

```typescript
const useLayerStore = create(
  devtools(          // ← 最外層：DevTools 記錄
    immer(           // ← 內層：Immer 不可變更新
      (set) => ({
        // ... state & actions
      })
    ),
    { name: 'LayerStore', enabled: import.meta.env.DEV }
  )
);
```

> `devtools` 應包在 `immer` 外層，確保 DevTools 記錄的是 Immer 產生的最終不可變狀態差異，而非 draft 物件。

---

> **下一份文件**：`09-project-io.md` — 專案匯出 / 匯入設計規格
