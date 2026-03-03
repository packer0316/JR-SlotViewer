# Slot Previewer — 渲染管線

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置閱讀**：`01-architecture-overview.md`

---

## 1. PixiJS Application 生命週期

### 1.1 初始化 (init)

Application 啟動時優先嘗試 WebGPU，若不支援則自動降級為 WebGL2：

```typescript
import { Application } from 'pixi.js';

const app = new Application();

await app.init({
  preference: 'webgpu',       // 優先 WebGPU，降級為 webgl2
  antialias: true,
  resolution: window.devicePixelRatio || 1,
  autoDensity: true,
  backgroundColor: 0x1a1a2e,  // 深色背景，搭配 Glassmorphism 主題
  resizeTo: undefined,         // 手動管理 resize（見 1.3）
});

canvasContainer.appendChild(app.canvas);
```

> **注意**：`resizeTo` 設為 `undefined`，改由 `ResizeObserver` 精確控制尺寸，避免多次不必要的 resize 呼叫。

### 1.2 Ticker（主迴圈）

PixiJS Ticker 是整個渲染管線的心跳。我們**不使用** `requestAnimationFrame` 直接驅動，而是完全交由 PixiJS 管理：

```typescript
app.ticker.maxFPS = 60;

app.ticker.add((ticker) => {
  const dt = ticker.deltaTime; // 以 60fps 為基準的 delta（1.0 = 正常速度）

  reelEngine.update(dt);
  spineActors.forEach((actor) => actor.update(dt));
  layerManager.applyPendingChanges();
});
```

| 屬性 | 值 | 說明 |
|------|----|------|
| `maxFPS` | 60 | 限制最大更新頻率，避免高刷螢幕上過度耗電 |
| `deltaTime` | ~1.0 | 基於 60fps 正規化的 delta，用於插值計算 |
| `deltaMS` | ~16.67 | 實際毫秒差值，用於需要真實時間的邏輯 |

### 1.3 Resize 處理

使用 `ResizeObserver` 監聽 Canvas 容器尺寸變化，確保 Renderer 與畫面同步：

```typescript
const resizeObserver = new ResizeObserver((entries) => {
  const { width, height } = entries[0].contentRect;
  app.renderer.resize(width, height);
  viewportManager.onResize(width, height);
});

resizeObserver.observe(canvasContainer);
```

### 1.4 銷毀 (destroy)

元件卸載時需徹底清理資源，避免記憶體洩漏：

```typescript
function destroy(): void {
  resizeObserver.disconnect();

  spineActors.forEach((actor) => actor.destroy());
  reelEngine.destroy();
  layerManager.destroy();

  app.destroy(true, { children: true, texture: true, textureSource: true });
}
```

`app.destroy(true, ...)` 會同時移除 Canvas DOM 元素並釋放所有 GPU 資源。

---

## 2. 場景圖結構 (Scene Graph)

整個畫面由一棵樹狀的 Container 階層構成，每一層各司其職：

```
Stage (root Container)
├── BackgroundLayer          ← 背景圖、底色
├── ReelContainer            ← 轉輪區域（所有軸的父容器）
│   ├── Reel[0] (Container + Graphics Mask)
│   │   ├── Symbol[0]       ← 由 SymbolPool 取得
│   │   ├── Symbol[1]
│   │   └── Symbol[2]
│   ├── Reel[1]
│   │   ├── Symbol[0]
│   │   ├── Symbol[1]
│   │   └── Symbol[2]
│   └── Reel[...]
├── SpineLayer               ← Spine 動畫演出層
│   ├── SpineActor: "wild"
│   └── SpineActor: "scatter"
├── OverlayLayer             ← 疊加特效（粒子、光暈等）
└── UILayer                  ← Canvas 內嵌 UI（如計分動畫）
```

### 各層職責

| 圖層 | 職責 | 典型內容 |
|------|------|---------|
| **BackgroundLayer** | 靜態背景，通常啟用 CacheAsTexture | 底圖 Sprite |
| **ReelContainer** | 轉輪模擬的根容器，含多軸 | Reel + Symbol |
| **SpineLayer** | Spine 動畫播放 | Wild、Scatter、Bonus 演出 |
| **OverlayLayer** | 疊加效果 | 粒子系統、全螢幕閃光 |
| **UILayer** | Canvas 內 UI 元素 | 計分動畫、贏分文字 |

> **原則**：圖層的繪製順序由 Scene Graph 中的**子節點順序**決定（後加入的畫在上方）。使用者可在圖層面板中拖曳調整順序，LayerManager 會同步更新 Container 的 `zIndex` 或重新排序子節點。

---

## 3. 圖層系統 (LayerManager)

### 3.1 LayerNode ↔ Container 映射

每個 `LayerNode`（Zustand Store 中的資料物件）對應一個 PixiJS `Container`。LayerManager 維護這份**雙向映射**：

```typescript
class LayerManager {
  private nodeToContainer = new Map<string, Container>();
  private containerToNode = new Map<Container, string>();

  getContainer(nodeId: string): Container | undefined {
    return this.nodeToContainer.get(nodeId);
  }

  getNodeId(container: Container): string | undefined {
    return this.containerToNode.get(container);
  }
}
```

### 3.2 同步流程

```
Zustand layerStore 變化
       │
       ▼
LayerManager.syncFromStore()
       │
       ├── diff 新舊 LayerNode 陣列
       │
       ├── 新增節點 → 建立 Container 並加入 Scene Graph
       ├── 刪除節點 → 從 Scene Graph 移除並銷毀
       ├── 順序變動 → 重新排列子節點 (setChildIndex / sortChildren)
       └── 屬性變動 → 更新對應 Container 屬性
```

### 3.3 支援的屬性操作

| 屬性 | 對應 PixiJS API | 說明 |
|------|-----------------|------|
| `visible` | `container.visible` | 顯示 / 隱藏 |
| `alpha` | `container.alpha` | 透明度 0–1 |
| `blendMode` | `container.blendMode` | 混合模式（見第 6 節） |
| `position` | `container.position.set(x, y)` | 位置偏移 |
| `scale` | `container.scale.set(sx, sy)` | 縮放 |
| `rotation` | `container.rotation` | 旋轉（弧度） |
| `zIndex` | 子節點順序 | 透過 `sortableChildren` + `zIndex` 管理 |

### 3.4 群組圖層 (Group Layers)

群組圖層即 Container 巢狀。將多個圖層放入同一個 Group 後，對 Group 操作（移動、縮放、隱藏）會同時作用於所有子圖層：

```typescript
function createGroupLayer(nodeId: string, children: string[]): Container {
  const group = new Container();
  group.sortableChildren = true;

  children.forEach((childId, index) => {
    const childContainer = this.nodeToContainer.get(childId);
    if (childContainer) {
      group.addChild(childContainer);
      childContainer.zIndex = index;
    }
  });

  return group;
}
```

### 3.5 圖層選取效果

當使用者在圖層面板中選取某圖層時，LayerManager 會在該 Container 周圍繪製**選取框**：

```typescript
function showSelectionOutline(container: Container): void {
  const bounds = container.getBounds();
  const outline = new Graphics()
    .rect(bounds.x, bounds.y, bounds.width, bounds.height)
    .stroke({ width: 2, color: 0x00bfff, alignment: 0 });

  outline.label = '__selection_outline__';
  app.stage.addChild(outline);
}
```

> 選取框繪製在 Stage 最頂層，不影響圖層本身的 Scene Graph 結構。

---

## 4. Clipping 系統 (ClipManager)

Clipping 用於限制特定區域的可見範圍，最典型的應用場景是**轉輪視窗**——Symbol 滾動時超出視窗的部分必須被裁切。

### 4.1 三種 Mask 模式

```
┌─────────────────────────────────────────────────────────┐
│                    Clipping 模式對照                      │
├──────────────┬────────────┬────────────┬────────────────┤
│              │ Graphics   │ Sprite     │ Stencil        │
│              │ Mask       │ Mask       │ Mask           │
├──────────────┼────────────┼────────────┼────────────────┤
│ 邊緣效果      │ 硬邊       │ 柔邊       │ 複雜形狀        │
│ 效能          │ ★★★★★    │ ★★★☆☆    │ ★★☆☆☆         │
│ 典型用途      │ 轉輪視窗   │ 漸層漸隱   │ 多層複合遮罩     │
│ GPU 開銷      │ 極低       │ 中等       │ 較高           │
└──────────────┴────────────┴────────────┴────────────────┘
```

#### 模式 1：Graphics Mask（硬邊）

**最佳效能**，適用於矩形或多邊形裁切。轉輪視窗預設使用此模式：

```typescript
function createReelMask(x: number, y: number, w: number, h: number): Graphics {
  const mask = new Graphics()
    .rect(x, y, w, h)
    .fill(0xffffff);

  return mask;
}

// 套用 mask 到轉輪容器
reelContainer.addChild(mask);
reelContainer.mask = mask;
```

#### 模式 2：Sprite Mask（柔邊）

使用圖片的 **Alpha 通道**作為遮罩，適合漸層消失、暈影等柔和效果：

```typescript
import { Sprite } from 'pixi.js';

const vignetteMask = Sprite.from('masks/vignette.png');
vignetteMask.anchor.set(0.5);
vignetteMask.position.set(centerX, centerY);

targetContainer.mask = vignetteMask;
```

> Sprite Mask 的圖片白色區域為可見，黑色區域為不可見，灰階區域為半透明。

#### 模式 3：Stencil Mask

當需要多個形狀組合成複雜遮罩時使用，需要 WebGL Stencil Buffer 支援：

```typescript
const complexMask = new Container();

const circle = new Graphics().circle(100, 100, 80).fill(0xffffff);
const rect = new Graphics().rect(50, 50, 200, 100).fill(0xffffff);

complexMask.addChild(circle, rect);
targetContainer.mask = complexMask;
```

### 4.2 轉輪視窗裁切

每個 Reel 配置一個 Graphics Mask 矩形。Symbol 即使超出視窗外也**仍然存在於 Scene Graph 中**，僅在視覺上被裁切。這確保了滾動動畫的流暢性——Symbol 可以從上方平滑進入視窗、從下方滑出：

```
          ┌────────────── Reel Container ──────────────┐
          │  Symbol[-1] (clipped, above window)         │
          │ ┌──────────── Mask Rect ────────────┐       │
          │ │  Symbol[0]  ← 可見                 │       │
          │ │  Symbol[1]  ← 可見                 │       │
          │ │  Symbol[2]  ← 可見                 │       │
          │ └──────────────────────────────────┘       │
          │  Symbol[3] (clipped, below window)         │
          └────────────────────────────────────────────┘
```

```typescript
class ReelContainer {
  private mask: Graphics;
  private symbols: Container[] = [];

  constructor(config: ReelConfig, reelIndex: number) {
    const { symbolSize, rowsPerReel } = config;
    const rows = rowsPerReel[reelIndex];
    const maskW = symbolSize.w;
    const maskH = symbolSize.h * rows;

    this.mask = new Graphics().rect(0, 0, maskW, maskH).fill(0xffffff);
    this.addChild(this.mask);
    this.container.mask = this.mask;
  }
}
```

---

## 5. 渲染管線流程

每一幀（目標 60fps）的完整處理流程如下：

```
  ┌─────────────────────────────────────────────────────────────┐
  │                  Ticker.update (每幀觸發)                     │
  │                                                              │
  │  ① ReelEngine.update(dt)                                     │
  │     └── 更新每個 Symbol 的 Y 座標（若正在轉動）                  │
  │         └── SymbolPool 回收離開視窗的 Symbol                    │
  │             SymbolPool 取出即將進入視窗的 Symbol                 │
  │                                                              │
  │  ② SpineActors.update(dt)                                    │
  │     └── 更新骨骼變換、計算當前動畫幀                              │
  │                                                              │
  │  ③ LayerManager.applyPendingChanges()                        │
  │     └── 批次套用所有待處理的屬性變更                              │
  │         （visible, alpha, blendMode, position 等）             │
  │                                                              │
  │  ④ PixiJS Renderer 遍歷 Scene Graph                          │
  │     └── 深度優先遍歷所有 Container                               │
  │         └── 對每個可見節點：                                     │
  │             ├── 計算世界矩陣 (updateTransform)                  │
  │             ├── 套用 Mask (如有)                                │
  │             ├── 套用 BlendMode                                 │
  │             ├── CacheAsTexture 檢查                            │
  │             └── 提交 Draw Call                                 │
  │                                                              │
  │  ⑤ GPU 執行繪製指令 → 畫面輸出                                  │
  └─────────────────────────────────────────────────────────────┘
```

### 效能要點

- **步驟 ①②③** 為邏輯層更新，由我們的程式碼控制
- **步驟 ④⑤** 為 PixiJS 內部渲染，開發者無需干預
- React **完全不參與**此迴圈，確保 UI 狀態更新不會拖累渲染幀率

---

## 6. Blend Modes

PixiJS 8 支援多種混合模式，可在圖層面板中逐層設定：

| 模式 | PixiJS 常數 | 效果 | 典型用途 |
|------|-------------|------|---------|
| **Normal** | `'normal'` | 預設混合，覆蓋下方 | 一般圖層 |
| **Add** | `'add'` | 顏色加法，產生發光效果 | 光暈、火焰 |
| **Multiply** | `'multiply'` | 顏色相乘，產生加深效果 | 陰影、暗角 |
| **Screen** | `'screen'` | 反轉相乘，產生提亮效果 | 閃光、光線 |

### 使用方式

```typescript
container.blendMode = 'add';
```

> **注意**：混合模式會打斷 PixiJS 的自動 batching，每切換一次 blendMode 就會產生一次新的 Draw Call。若非必要，盡量讓相鄰圖層使用相同的混合模式以維持 batching 效率。

---

## 7. CacheAsTexture 策略

### 7.1 概念

`CacheAsTexture` 將一個 Container 及其所有子節點渲染為一張 **RenderTexture**，後續幀直接繪製該材質而非逐一遍歷子節點。對於靜態或不常變動的圖層，這能顯著減少 Draw Call 數量。

### 7.2 何時快取

```
┌─────────────────────────────────────────────────────┐
│          CacheAsTexture 決策流程                      │
│                                                      │
│  圖層是否正在播放動畫？                                 │
│      ├── 是 → ❌ 不快取（每幀內容都在變）                │
│      └── 否 → 圖層子節點數量是否 > 1？                  │
│               ├── 是 → ✅ 快取（減少多次 Draw Call）     │
│               └── 否 → 🔸 視情況而定                    │
│                        （單個 Sprite 快取收益低）        │
└─────────────────────────────────────────────────────┘
```

### 7.3 API 使用

```typescript
// 啟用快取
container.cacheAsTexture(true);

// 手動使快取失效（當內容有變更時）
container.updateCacheTexture();

// 關閉快取
container.cacheAsTexture(false);
```

### 7.4 自動管理

LayerManager 根據圖層狀態自動管理快取：

```typescript
class LayerManager {
  private updateCacheStrategy(nodeId: string, container: Container): void {
    const node = this.getNode(nodeId);

    const shouldCache =
      node.type !== 'spine' &&        // Spine 動畫不快取
      !node.isAnimating &&            // 正在播放動畫的不快取
      container.children.length > 1;  // 子節點多於一個才有收益

    if (shouldCache && !container.cacheAsBitmap) {
      container.cacheAsTexture(true);
    } else if (!shouldCache && container.cacheAsBitmap) {
      container.cacheAsTexture(false);
    }
  }
}
```

### 7.5 失效時機

| 觸發條件 | 處理方式 |
|----------|---------|
| 子節點新增 / 移除 | `container.cacheAsTexture(false)` → 重新評估 |
| 子節點屬性變更（position, alpha 等） | `container.updateCacheTexture()` |
| 動畫開始播放 | `container.cacheAsTexture(false)` |
| 動畫結束 | 重新評估，可能再次啟用快取 |

---

## 8. 座標系統與視角控制

### 8.1 座標系統

PixiJS 使用**左上角為原點**的座標系統：

```
(0, 0) ─────────→ +X
  │
  │
  │
  ▼
 +Y
```

所有圖層的 `position` 值皆為**世界座標**（相對於 Stage 根節點）。

### 8.2 視角控制 (ViewportManager)

透過縮放與平移 Stage 容器，實現整體畫面的縮放與拖曳：

```typescript
class ViewportManager {
  private stage: Container;
  private scale = 1;
  private offset = { x: 0, y: 0 };

  zoom(delta: number, pivotX: number, pivotY: number): void {
    const oldScale = this.scale;
    this.scale = Math.max(0.1, Math.min(5, this.scale + delta));

    // 以滑鼠位置為中心縮放
    const ratio = this.scale / oldScale;
    this.offset.x = pivotX - (pivotX - this.offset.x) * ratio;
    this.offset.y = pivotY - (pivotY - this.offset.y) * ratio;

    this.applyTransform();
  }

  pan(dx: number, dy: number): void {
    this.offset.x += dx;
    this.offset.y += dy;
    this.applyTransform();
  }

  resetView(canvasWidth: number, canvasHeight: number): void {
    const bounds = this.stage.getLocalBounds();
    const scaleX = canvasWidth / bounds.width;
    const scaleY = canvasHeight / bounds.height;
    this.scale = Math.min(scaleX, scaleY) * 0.9; // 留 10% 邊距

    this.offset.x = (canvasWidth - bounds.width * this.scale) / 2;
    this.offset.y = (canvasHeight - bounds.height * this.scale) / 2;

    this.applyTransform();
  }

  private applyTransform(): void {
    this.stage.scale.set(this.scale);
    this.stage.position.set(this.offset.x, this.offset.y);
  }
}
```

### 8.3 座標轉換

在處理使用者點擊、拖曳等互動時，需要在**螢幕座標**與**世界座標**之間轉換：

```typescript
class ViewportManager {
  /** 螢幕座標 → 世界座標 */
  screenToWorld(screenX: number, screenY: number): { x: number; y: number } {
    return {
      x: (screenX - this.offset.x) / this.scale,
      y: (screenY - this.offset.y) / this.scale,
    };
  }

  /** 世界座標 → 螢幕座標 */
  worldToScreen(worldX: number, worldY: number): { x: number; y: number } {
    return {
      x: worldX * this.scale + this.offset.x,
      y: worldY * this.scale + this.offset.y,
    };
  }
}
```

### 8.4 操作方式

| 操作 | 觸發方式 | 行為 |
|------|---------|------|
| 縮放 | 滑鼠滾輪 / 觸控板捏合 | 以游標位置為中心縮放 |
| 平移 | 中鍵拖曳 / 空白鍵 + 左鍵拖曳 | 平移視角 |
| 重置 | 雙擊中鍵 / 工具列按鈕 | 置中並自適應縮放 |

---

> **下一份文件**：`05-reel-engine.md` — 轉輪引擎設計規格
