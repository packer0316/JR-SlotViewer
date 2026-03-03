# Slot Previewer — 效能策略

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 1. 效能目標

Slot Previewer 必須在瀏覽器中同時驅動 **5 條轉輪 × 3+ 列 Symbol** 與 **2 組 Spine 動畫**，並維持流暢的 60 fps。以下為各項量化指標：

| 指標 | 目標值 | 測量方式 |
|------|--------|----------|
| 目標幀率 | 60 FPS | PixiJS Ticker + 自訂 FPS Counter |
| DrawCall | < 10 DC / 幀 | `renderer.renderPipes` 統計 |
| JS Heap | < 200 MB | `performance.measureUserAgentSpecificMemory()` |
| 初始載入 | < 3 秒 | Vite dev mode（首次 HTML → 畫面可互動） |
| Spine 切換延遲 | < 16 ms | `performance.now()` 前後差值 |

> **為何 < 10 DC？**
> 每個 DrawCall 對應一次 GPU 狀態切換。在同時渲染 5 軸 Symbol + Spine 動畫 + UI 圖層的場景下，若不做任何合批，DrawCall 可能輕易超過 30。透過 Atlas 合批與 CacheAsTexture，將 DrawCall 壓在 10 以內，可確保低階 GPU 也能維持 60 fps。

---

## 2. 渲染效能策略

### 2.1 CacheAsTexture（靜態圖層快取）

對於不經常變動的圖層（如背景、邊框裝飾），將其快取為 `RenderTexture`，後續幀只需繪製單張貼圖，大幅減少 DrawCall。

**策略：**

- 靜態圖層在首次渲染後自動快取為 RenderTexture
- 當圖層屬性（位置、透明度、混合模式）發生變更時，標記 dirty 並在下一幀重新快取
- 自動管理：連續 N 幀無變更時自動啟用快取

```typescript
// CacheAsTexture 使用方式（PixiJS 8）
const staticLayer = new Container();
staticLayer.cacheAsTexture(true);

// 當屬性變更時，需手動更新快取
function onPropertyChange(layer: Container) {
  layer.cacheAsTexture(false);
  // 修改屬性...
  layer.cacheAsTexture(true);
}
```

**適用場景：**

| 場景 | 是否快取 | 原因 |
|------|----------|------|
| 背景圖層 | ✅ 是 | 幾乎不變動 |
| Symbol 靜止顯示 | ✅ 是 | 停輪後靜止 |
| 轉輪滾動中 | ❌ 否 | 每幀都在變動 |
| Spine 動畫播放中 | ❌ 否 | 每幀更新骨骼 |

### 2.2 Spine 合批（Batching）

同材質（Atlas + Blend Mode）的 Spine 骨骼可合併為單一 DrawCall，大幅降低渲染開銷。

**合批條件：**

1. 使用**相同 Atlas 貼圖**
2. 使用**相同混合模式**（Normal / Additive / ...）
3. **無 Stencil / Mask 切換**

**破壞合批的常見原因：**

```
❌ 不同 Spine 使用各自獨立的 Atlas → 貼圖切換 → DrawCall +1
❌ 同一 Spine 中混用 Normal + Additive 混合模式 → 混合模式切換
❌ 穿插非 Spine 元素 → 中斷批次連續性
```

**最佳實踐：**

- 相關 Spine（如同場景的角色與特效）盡量共用同一張 Atlas
- 在 Scene Graph 中將同 Atlas 的 Spine 節點排列在一起，避免穿插其他元素
- 利用 PixiJS 8 的 `renderPipes` 統計確認合批效果

### 2.3 符號圖集（Symbol Atlas）

所有轉輪符號打包為**單張貼圖集**，消除轉輪滾動時的貼圖切換開銷。

```
┌───────────────────────────────────────┐
│  Symbol Atlas (2048 × 2048)           │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │
│  │ SYM1│ │ SYM2│ │ SYM3│ │ SYM4│    │
│  └─────┘ └─────┘ └─────┘ └─────┘    │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │
│  │ SYM5│ │ SYM6│ │ SYM7│ │ SYM8│    │
│  └─────┘ └─────┘ └─────┘ └─────┘    │
│  ...                                  │
└───────────────────────────────────────┘
```

**工具與流程：**

- 使用 **TexturePacker** 或 **Free Texture Packer** 產生 Spritesheet JSON + 合併 PNG
- 單張 Atlas 建議不超過 **2048 × 2048**（相容低階裝置）
- 若 Symbol 數量超出單張限制，按**使用頻率**分組，優先將高頻 Symbol 放在同一張

### 2.4 Object Pool（物件池）

轉輪滾動時 Symbol 節點會頻繁進出可見區域。若每次都 `new Sprite()` + `destroy()`，將產生大量短命物件，觸發 GC 暫停（GC Jank）。

**SymbolPool 設計：**

```typescript
class SymbolPool {
  private pool: Map<string, Sprite[]> = new Map();

  acquire(symbolId: string): Sprite {
    const stack = this.pool.get(symbolId);
    if (stack && stack.length > 0) {
      return stack.pop()!;
    }
    return this.createSymbolSprite(symbolId);
  }

  release(symbolId: string, sprite: Sprite): void {
    sprite.visible = false;
    const stack = this.pool.get(symbolId) ?? [];
    stack.push(sprite);
    this.pool.set(symbolId, stack);
  }
}
```

**池大小計算：**

```
poolSize = rows × reels + scrollBuffer

範例：3 rows × 5 reels + 10 buffer = 25 個 Symbol Sprite
```

**監控指標：**

| 指標 | 說明 |
|------|------|
| `poolSize` | 池中所有物件數（含使用中） |
| `available` | 可用（閒置）物件數 |
| `utilization` | `(poolSize - available) / poolSize` |
| `missCount` | 池空時新建物件的次數（應趨近 0） |

---

## 3. React 效能策略

### 3.1 Zustand Selector 細粒度訂閱

Zustand 預設行為是：任何 store 屬性變更都會觸發所有訂閱者 re-render。透過 **Selector** 可精確訂閱需要的欄位，避免不必要的重繪。

```typescript
// ❌ 反模式：訂閱整個 store
const store = useLayerStore();

// ✅ 正確：只訂閱需要的欄位
const selectedLayerId = useLayerStore((s) => s.selectedLayerId);

// ✅ 物件選取時使用 useShallow 避免參考不等的 re-render
import { useShallow } from 'zustand/react/shallow';

const { visible, locked } = useLayerStore(
  useShallow((s) => ({
    visible: s.layers[id]?.visible,
    locked: s.layers[id]?.locked,
  }))
);
```

**原則：**

- 每個元件只訂閱**自身需要的最小欄位集**
- 物件 / 陣列類型的選取結果必須搭配 `useShallow`
- 絕對不要在 Selector 中 `return { ...state }`

### 3.2 React.memo & useMemo

```typescript
// 純展示元件使用 React.memo 防止父層 re-render 連鎖
const LayerItem = React.memo(({ layer }: { layer: LayerNode }) => {
  return <div>{layer.name}</div>;
});

// 昂貴運算結果快取
const sortedLayers = useMemo(
  () => layers.slice().sort((a, b) => a.order - b.order),
  [layers]
);

// 回調函式穩定化
const handleClick = useCallback(() => {
  selectLayer(layer.id);
}, [layer.id]);
```

### 3.3 React 與 PixiJS 隔離

這是 Slot Previewer 最關鍵的效能架構決策：**React 永遠不進入 PixiJS 的 Ticker 迴圈**。

```
┌──────────────────────────────────────────────────────┐
│  React 領域（16ms+ 更新週期，由使用者操作觸發）         │
│  • UI 面板（圖層列表、屬性面板、控制按鈕）               │
│  • Zustand Store（狀態容器）                            │
└────────────────────┬─────────────────────────────────┘
                     │ 訂閱 Store 變化
┌────────────────────▼─────────────────────────────────┐
│  PixiJS 領域（16.6ms 更新週期，Ticker 驅動）             │
│  • LayerManager → 直接操作 Scene Graph                  │
│  • ReelEngine → 直接更新 Symbol 位置                    │
│  • SpinePlayer → 直接驅動骨骼動畫                       │
└──────────────────────────────────────────────────────┘
```

**規則：**

| 規則 | 說明 |
|------|------|
| React 不碰 Canvas | React 元件不直接操作任何 PixiJS 物件 |
| Manager 不觸發 re-render | PixiJS Manager 不呼叫 `setState`，不使用 React hooks |
| Store 是唯一橋樑 | React → Store → Manager 是唯一的跨領域通訊路徑 |

---

## 4. 記憶體管理

### 4.1 Texture 管理

貼圖（Texture）是 Slot Previewer 中最大的記憶體消費者。必須嚴格追蹤貼圖生命週期。

**策略：**

- 移除圖層或切換場景時，**主動呼叫 `texture.destroy(true)`** 釋放 GPU 與 CPU 記憶體
- 維護 `TextureRefCounter` 追蹤每張貼圖的參考數，歸零時自動釋放
- 使用 `WeakRef` 搭配 `FinalizationRegistry` 監控貼圖物件回收狀態

```typescript
class TextureRefCounter {
  private refs = new Map<string, { texture: Texture; count: number }>();

  retain(key: string, texture: Texture): void {
    const entry = this.refs.get(key);
    if (entry) {
      entry.count++;
    } else {
      this.refs.set(key, { texture, count: 1 });
    }
  }

  release(key: string): void {
    const entry = this.refs.get(key);
    if (!entry) return;
    entry.count--;
    if (entry.count <= 0) {
      entry.texture.destroy(true);
      this.refs.delete(key);
    }
  }
}
```

### 4.2 Spine 資源管理

Spine 資源分為**骨架資料（Skeleton Data）** 與**骨架實例（Skeleton Instance）**，兩者的記憶體策略不同：

| 資源類型 | 快取策略 | 生命週期 |
|----------|----------|----------|
| SkeletonData | ✅ 全域快取 | 隨專案載入 / 卸載 |
| Skeleton Instance | ❌ 不快取 | 隨圖層建立 / 移除 |
| Atlas Texture | ✅ 參考計數 | 最後一個使用者釋放時銷毀 |

**關鍵措施：**

- 快取 `SkeletonData`（骨架定義），多個圖層可共享同一份資料
- 圖層移除時立即 `dispose()` 對應的 Skeleton Instance（Spine Actor）
- `.skel` 二進位解析移至 **Web Worker**，避免主執行緒出現分配尖峰

```typescript
// Web Worker 解析 .skel 檔案
// worker.ts
self.onmessage = async (e: MessageEvent<ArrayBuffer>) => {
  const skelData = parseSkelBinary(e.data);
  self.postMessage(skelData, [skelData.buffer]);
};
```

### 4.3 Object Pool 管理

| 操作 | 策略 |
|------|------|
| 預分配 | 應用啟動時根據 `rows × reels + buffer` 預建物件 |
| 複用 | 滾動時從池中取出，離開可見區域時歸還 |
| 縮容 | 閒置超過 N 秒後，釋放超額物件，降低記憶體佔用 |
| 禁止動態分配 | 動畫進行中絕不 `new Sprite()`，僅從池中取用 |

---

## 5. 載入效能

### 5.1 Vite 開發模式

| 特性 | 效果 |
|------|------|
| HMR（Hot Module Replacement） | 修改程式碼後瞬間反映，無需整頁重載 |
| 按需編譯 | 只編譯當前請求的模組，啟動速度快 |
| Lazy Import | 非關鍵模組（如 ScriptEditor）延遲載入 |

```typescript
// 非關鍵模組使用 lazy import
const ScriptEditor = React.lazy(() => import('./panels/ScriptEditor'));
```

### 5.2 生產建置

```typescript
// vite.config.ts — 分包策略
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'zustand'],
          pixi: ['pixi.js'],
          spine: ['@esotericsoftware/spine-pixi'],
        },
      },
    },
  },
});
```

**最佳化項目：**

| 項目 | 策略 |
|------|------|
| Code Splitting | 將 vendor / pixi / spine 分離為獨立 chunk |
| Tree Shaking | 只匯入使用到的 PixiJS 功能模組 |
| 資源預載 | 關鍵資源（Symbol Atlas、基底 Spine）優先載入 |
| 壓縮 | 使用 gzip / brotli 壓縮靜態資源 |

### 5.3 Web Worker 離線解析

大型 Spine `.skel` 檔案的解析可能耗時數十毫秒，足以造成可感知的卡頓。將解析工作移至 Web Worker：

```
主執行緒                          Web Worker
────────                          ──────────
fetch(.skel) ─→ ArrayBuffer
                    │
                    ├─→ postMessage(buffer) ─→ parseSkelBinary()
                    │                              │
                    │   ←─ postMessage(skelData) ──┘
                    │
建立 SkeletonData ←─┘
```

**未來擴充：** `.zip` 專案檔案的解壓縮也可移至 Worker，避免大型專案匯入時凍結 UI。

---

## 6. 效能監控

### 6.1 Status Bar 即時指標

Status Bar 位於應用程式底部，常駐顯示效能指標：

```
┌──────────────────────────────────────────────────┐
│  FPS: 60  │  DC: 7  │  Mem: 142 MB  │  Pool: 92%│
└──────────────────────────────────────────────────┘
```

| 指標 | 來源 | 更新頻率 |
|------|------|----------|
| FPS | PixiJS Ticker `deltaTime` 滑動平均 | 每秒 |
| DC（DrawCall） | `renderer.renderPipes` 統計 | 每幀 |
| Mem（記憶體） | `performance.measureUserAgentSpecificMemory()` | 每 5 秒 |
| Pool（物件池使用率） | `SymbolPool.utilization` | 每秒 |

### 6.2 開發者工具

| 工具 | 用途 |
|------|------|
| **PixiJS DevTools** | 瀏覽器擴充套件，檢視 Scene Graph、Texture、DrawCall 分佈 |
| **Chrome Performance** | 錄製 Timeline，定位長幀（Long Frame）與 GC 暫停 |
| **Chrome Memory** | 堆積快照（Heap Snapshot），追蹤記憶體洩漏 |
| **Zustand DevTools** | 檢視 Store 狀態變遷歷史，定位不必要的 re-render |

### 6.3 效能預算（Performance Budget）

為每項關鍵指標設定預算上限，開發模式下超出時自動發出警告：

```typescript
const PERF_BUDGET = {
  frameTimeMs: 16.6,
  drawCalls: 10,
  heapMB: 200,
  loadTimeSec: 3,
} as const;

function checkBudget(metrics: PerfMetrics): void {
  if (metrics.frameTimeMs > PERF_BUDGET.frameTimeMs) {
    console.warn(
      `⚠️ Frame time ${metrics.frameTimeMs.toFixed(1)}ms exceeds budget ${PERF_BUDGET.frameTimeMs}ms`
    );
  }
  if (metrics.drawCalls > PERF_BUDGET.drawCalls) {
    console.warn(
      `⚠️ DrawCalls ${metrics.drawCalls} exceeds budget ${PERF_BUDGET.drawCalls}`
    );
  }
}
```

---

## 7. 已知瓶頸與對策

| 瓶頸 | 原因 | 對策 |
|------|------|------|
| Spine 載入延遲 | `.skel` 二進位解析佔用主執行緒 | Web Worker 離線解析（§5.3） |
| 轉輪滾動卡頓 | Symbol 頻繁建立 / 銷毀引發 GC 壓力 | ObjectPool 複用（§2.4） |
| 大量圖層 DrawCall | 每個圖層獨立繪製 | CacheAsTexture + Spine 合批（§2.1, §2.2） |
| Atlas 過大 | 單張 Atlas 佔用過多 GPU 記憶體 | 按需載入 + 依使用頻率分組 Atlas（§2.3） |
| React re-render 風暴 | Store 狀態變更觸發大面積重繪 | 細粒度 Selector + useShallow（§3.1） |
| Spine 切換閃爍 | 切換時重新建立 Skeleton 耗時 | 快取 SkeletonData + 預建實例（§4.2） |
| 記憶體洩漏 | Texture 未正確釋放 | TextureRefCounter 追蹤 + 自動回收（§4.1） |

---

## 8. 效能最佳化檢查清單

開發過程中定期檢查以下項目：

- [ ] FPS 穩定 60，無明顯掉幀
- [ ] DrawCall < 10（使用 PixiJS DevTools 確認）
- [ ] JS Heap < 200 MB（含 Atlas + Skeleton Data）
- [ ] 轉輪滾動期間 GC 暫停 < 2 ms
- [ ] Spine 切換延遲 < 16 ms
- [ ] 初始載入 < 3 秒（Vite dev mode）
- [ ] 物件池 `missCount` 趨近 0
- [ ] 移除圖層後 Texture 確實釋放（Heap Snapshot 驗證）
- [ ] React DevTools 無不必要的 re-render 標記

---

*本文件為效能策略指南，實際數值可能因硬體與瀏覽器差異而調整。建議搭配 Chrome Performance 工具定期 profiling。*
