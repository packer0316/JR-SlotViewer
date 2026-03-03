# Slot Previewer — 轉輪系統設計規格

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 1. 轉輪系統概覽

轉輪系統（Reel System）是 Slot Previewer 的核心模擬引擎，負責重現 Slot 遊戲中轉輪旋轉、停輪、回彈等完整行為。系統設計遵循以下原則：

| 原則 | 說明 |
|------|------|
| **PixiJS Ticker 驅動** | 所有運動皆由 PixiJS Ticker 驅動（非 `requestAnimationFrame`），確保與渲染引擎同步，穩定 60 fps。 |
| **每軸獨立** | 每條轉輪（Reel）是獨立的垂直欄位，可各自啟動、旋轉、停輪，互不干涉。 |
| **可變列數** | 支援每軸不同列數（如 Megaways 風格 `[3,4,5,4,3]`），突破傳統固定 NxM 盤面限制。 |
| **物件池** | Symbol 節點透過物件池（Object Pool）管理，避免頻繁建立 / 銷毀造成的效能損耗。 |
| **Store 驅動** | 所有配置與狀態皆存放於 Zustand `reelStore`，UI 層與 Engine 層透過 Store 同步。 |

### 系統定位

```
┌─────────────────────────────────────────────────────────────┐
│  React UI Layer                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ ReelPanel    │  │ReelGridEditor│  │ SpinControlPanel │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
│         │                 │                    │             │
│  ┌──────▼─────────────────▼────────────────────▼──────────┐  │
│  │              Zustand reelStore                          │  │
│  │  reelConfig │ spinState │ resultMatrix │ poolStats      │  │
│  └──────┬─────────────────────────────────────────────────┘  │
│         │  訂閱變化                                          │
│  ┌──────▼─────────────────────────────────────────────────┐  │
│  │              Reel Engine Layer                          │  │
│  │  ReelEngine │ SymbolPool │ ReelContainer │ ClipManager  │  │
│  └──────┬─────────────────────────────────────────────────┘  │
│         │  操作 Scene Graph                                  │
│  ┌──────▼─────────────────────────────────────────────────┐  │
│  │              PixiJS Application (Canvas)                │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 相關原始碼

| 檔案 | 職責 |
|------|------|
| `src/reel/ReelConfig.ts` | 盤面配置型別定義與預設值 |
| `src/reel/ReelEngine.ts` | 轉輪運動邏輯（加速、等速、減速、停輪、回彈） |
| `src/reel/ReelGridEditor.ts` | 盤面網格視覺化編輯器（React 元件） |
| `src/reel/SymbolPool.ts` | Symbol 物件池管理 |
| `src/pixi/ReelContainer.ts` | PixiJS 轉輪容器與 Clipping |
| `src/store/reelStore.ts` | Zustand 轉輪狀態 |

---

## 2. 盤面配置（ReelConfig）

`ReelConfig` 是轉輪系統的核心資料結構，定義了盤面的所有可調參數。

### 2.1 型別定義

```typescript
interface SymbolDefinition {
  id: string;
  name: string;
  type: 'sprite' | 'spine';
  /** Sprite 模式：圖片路徑 */
  spritePath?: string;
  /** Spine 模式：skeleton 資料路徑 */
  spinePath?: string;
  /** Spine 預設動畫名稱 */
  defaultAnimation?: string;
}

interface SpinConfig {
  /** 加速階段持續時間（ms） */
  easeInDuration: number;
  /** 等速階段持續時間（ms） */
  constantDuration: number;
  /** 減速階段持續時間（ms） */
  easeOutDuration: number;
  /** 最大速度（px/s） */
  maxSpeed: number;
  /** 加速曲線（如 'power2.in'） */
  easeInCurve: string;
  /** 減速曲線（如 'power2.out'） */
  easeOutCurve: string;
  /** 每軸停輪延遲（ms）：第 i 軸延遲 = baseDelay + i * staggerDelay */
  baseStopDelay: number;
  staggerDelay: number;
}

interface BounceConfig {
  /** 是否啟用回彈 */
  enabled: boolean;
  /** 回彈振幅（px） */
  amplitude: number;
  /** 回彈次數 */
  count: number;
  /** 衰減係數（0–1，值越小衰減越快） */
  damping: number;
  /** 回彈動畫持續時間（ms） */
  duration: number;
}

interface ReelConfig {
  /** 配置識別碼 */
  id: string;
  /** 輪軸數量（欄數） */
  reels: number;
  /** 每軸列數陣列，長度必須等於 reels */
  rowsPerReel: number[];
  /** 符號尺寸（px） */
  symbolSize: { w: number; h: number };
  /** 符號間距（px） */
  symbolGap: { x: number; y: number };
  /** 垂直對齊方式 */
  alignment: 'top' | 'center' | 'bottom';
  /** 各軸符號帶（Symbol Strip）：reelStrips[i] 為第 i 軸的完整符號序列 */
  reelStrips: number[][];
  /** 符號定義表 */
  symbols: Record<string, SymbolDefinition>;
  /** 旋轉配置 */
  spinConfig: SpinConfig;
  /** 回彈配置 */
  bounceConfig: BounceConfig;
}
```

### 2.2 預設配置範例

**標準 5×3 盤面**

```typescript
const standard5x3: ReelConfig = {
  id: 'standard-5x3',
  reels: 5,
  rowsPerReel: [3, 3, 3, 3, 3],
  symbolSize: { w: 160, h: 160 },
  symbolGap: { x: 4, y: 4 },
  alignment: 'center',
  reelStrips: [
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9],  // 軸 0
    [1, 3, 5, 7, 9, 0, 2, 4, 6, 8],  // 軸 1
    [2, 4, 6, 8, 0, 1, 3, 5, 7, 9],  // 軸 2
    [3, 5, 7, 9, 1, 0, 4, 6, 8, 2],  // 軸 3
    [4, 6, 8, 0, 2, 1, 5, 7, 9, 3],  // 軸 4
  ],
  symbols: { /* 符號定義 */ },
  spinConfig: {
    easeInDuration: 300,
    constantDuration: 800,
    easeOutDuration: 500,
    maxSpeed: 2400,
    easeInCurve: 'power2.in',
    easeOutCurve: 'power2.out',
    baseStopDelay: 0,
    staggerDelay: 200,
  },
  bounceConfig: {
    enabled: true,
    amplitude: 20,
    count: 2,
    damping: 0.5,
    duration: 300,
  },
};
```

**Megaways 風格盤面**

```typescript
const megaways: ReelConfig = {
  id: 'megaways-style',
  reels: 6,
  rowsPerReel: [3, 4, 5, 5, 4, 3],
  symbolSize: { w: 120, h: 120 },
  symbolGap: { x: 4, y: 4 },
  alignment: 'center',
  // ...其餘欄位同上
};
```

### 2.3 盤面尺寸計算

根據 `ReelConfig`，可推導出各軸與整體盤面的像素尺寸：

```typescript
function calculateReelDimensions(config: ReelConfig) {
  const { symbolSize, symbolGap, rowsPerReel, reels } = config;

  const reelWidths = Array(reels).fill(symbolSize.w);
  const reelHeights = rowsPerReel.map(
    (rows) => rows * symbolSize.h + (rows - 1) * symbolGap.y
  );

  const totalWidth =
    reels * symbolSize.w + (reels - 1) * symbolGap.x;
  const maxHeight = Math.max(...reelHeights);

  return { reelWidths, reelHeights, totalWidth, maxHeight };
}
```

### 2.4 對齊模式圖示

當各軸列數不同時，`alignment` 決定符號的垂直起始位置：

```
alignment: 'top'           alignment: 'center'        alignment: 'bottom'

 ┌───┐ ┌───┐ ┌───┐         ┌───┐ ┌───┐ ┌───┐               ┌───┐
 │ A │ │ A │ │ A │         │ A │ │   │ │   │               │   │
 ├───┤ ├───┤ ├───┤         ├───┤ │ A │ │   │               │   │
 │ B │ │ B │ │ B │         │ B │ ├───┤ │ A │         ┌───┐ │ A │
 ├───┤ ├───┤ ├───┤         ├───┤ │ B │ ├───┤         │ A │ ├───┤
 │ C │ │ C │ │ C │         │ C │ ├───┤ │ B │         ├───┤ │ B │
 └───┘ ├───┤ ├───┤         └───┘ │ C │ ├───┤         │ B │ ├───┤
       │ D │ │ D │               ├───┤ │ C │         ├───┤ │ C │
       └───┘ ├───┤               │ D │ ├───┤         │ C │ ├───┤
             │ E │               └───┘ │ D │         └───┘ │ D │
             └───┘                     ├───┤               ├───┤
                                       │ E │               │ E │
                                       └───┘               └───┘
 rows: [3]  [4]  [5]       rows: [3]  [4]  [5]      rows: [3]  [4]  [5]
```

---

## 3. 盤面編輯器（ReelGridEditor）

盤面編輯器是面向**美術人員**的視覺化工具，操作邏輯類似 Excel 試算表，讓使用者以直覺方式設定轉輪盤面。

### 3.1 UI 佈局

```
┌──────────────────────────────────────────────────────────────┐
│  盤面配置                                              [×]   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  基本設定                                                    │
│  ┌────────────────────────┐  ┌────────────────────────────┐  │
│  │ 輪軸數  [5]  [−] [+]  │  │ 對齊方式  ○ 頂部  ● 置中  ○ 底部 │  │
│  └────────────────────────┘  └────────────────────────────┘  │
│                                                              │
│  符號尺寸                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 寬度  ◄━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━►  [160] px  │    │
│  │ 高度  ◄━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━►  [160] px  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  符號間距                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 水平  ◄━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━►  [4] px    │    │
│  │ 垂直  ◄━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━►  [4] px    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  盤面預覽                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                                                      │    │
│  │    ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    ├────┤  ├────┤  ├────┤  ├────┤  ├────┤          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    ├────┤  ├────┤  ├────┤  ├────┤  ├────┤          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    │    │  │    │  │    │  │    │  │    │          │    │
│  │    └────┘  ├────┤  ├────┤  ├────┤  └────┘          │    │
│  │            │    │  │    │  │    │                   │    │
│  │            │    │  │    │  │    │                   │    │
│  │            └────┘  ├────┤  └────┘                   │    │
│  │                    │    │                            │    │
│  │                    │    │                            │    │
│  │                    └────┘                            │    │
│  │                                                      │    │
│  │  每軸列數                                             │    │
│  │  [3]▼   [4]▼   [5]▼   [4]▼   [3]▼                   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────┐  ┌──────────┐                                  │
│  │  套用變更  │  │  重設預設  │                                  │
│  └──────────┘  └──────────┘                                  │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 互動行為

| 操作 | 行為 |
|------|------|
| **調整輪軸數** | 點擊 `[+]` / `[−]` 增減欄數，`rowsPerReel` 陣列同步增減（新增軸預設列數取最後一軸值） |
| **調整每軸列數** | 每軸下方的下拉選單，可選 1–10 列 |
| **拖曳調整符號尺寸** | 預覽區中可直接拖曳符號角落的手柄，即時調整 `symbolSize` |
| **Slider 調整** | 符號尺寸與間距皆提供 Slider + 數值輸入框，支援精確輸入與快速拖曳 |
| **對齊方式切換** | Radio 按鈕組，切換後預覽即時更新 |
| **即時預覽** | 任何參數變更皆立即反映在盤面預覽區域，無須手動套用 |
| **套用 / 重設** | 「套用變更」將配置寫入 `reelStore`，「重設預設」回復到標準 5×3 |

### 3.3 React 元件結構

```typescript
// ReelGridEditor 元件樹
<ReelGridEditor>
  <GridBasicSettings />        {/* 輪軸數、對齊方式 */}
  <SymbolSizeControls />       {/* 符號尺寸 Slider */}
  <SymbolGapControls />        {/* 符號間距 Slider */}
  <GridPreview>                {/* 盤面預覽畫布 */}
    <ReelColumn />             {/* 各軸欄位（×N） */}
    <DragHandle />             {/* 拖曳調整手柄 */}
  </GridPreview>
  <RowCountControls />         {/* 每軸列數下拉選單 */}
  <ActionButtons />            {/* 套用 / 重設 */}
</ReelGridEditor>
```

### 3.4 Store 整合

```typescript
// reelStore.ts — 盤面編輯相關 actions
interface ReelStoreActions {
  setReelCount: (count: number) => void;
  setRowsForReel: (reelIndex: number, rows: number) => void;
  setSymbolSize: (size: { w: number; h: number }) => void;
  setSymbolGap: (gap: { x: number; y: number }) => void;
  setAlignment: (align: 'top' | 'center' | 'bottom') => void;
  applyConfig: (config: ReelConfig) => void;
  resetToDefault: () => void;
}
```

---

## 4. ReelEngine 運動模型

ReelEngine 是轉輪系統的物理引擎，負責計算每一幀（tick）中各軸的位移量。整個旋轉生命週期分為四個階段：

### 4.1 旋轉生命週期

```
                    spin()                    stop(reelIndex)
                      │                           │
  ┌───────┐     ┌─────▼─────┐     ┌──────────┐   │  ┌──────────┐     ┌────────┐
  │ IDLE  │────▶│  EASE_IN   │────▶│ CONSTANT │───▼─▶│ EASE_OUT │────▶│ BOUNCE │──▶ IDLE
  └───────┘     └───────────┘     └──────────┘      └──────────┘     └────────┘
                  加速階段            等速階段          減速階段         回彈階段
```

### 4.2 各階段詳解

#### EASE_IN — 加速階段

```typescript
/**
 * 速度由 0 加速至 maxSpeed。
 * progress = elapsed / easeInDuration （0 → 1）
 * speed = maxSpeed × easingFn(progress)
 */
function updateEaseIn(reel: ReelState, dt: number): void {
  reel.elapsed += dt;
  const progress = Math.min(reel.elapsed / reel.config.spinConfig.easeInDuration, 1);
  const eased = applyEasing(progress, reel.config.spinConfig.easeInCurve);
  reel.speed = reel.config.spinConfig.maxSpeed * eased;

  if (progress >= 1) {
    reel.phase = 'CONSTANT';
    reel.elapsed = 0;
  }
}
```

#### CONSTANT — 等速階段

```typescript
/**
 * 以 maxSpeed 等速捲動。
 * 持續到收到 stop 指令或 constantDuration 到期。
 */
function updateConstant(reel: ReelState, dt: number): void {
  reel.speed = reel.config.spinConfig.maxSpeed;
  reel.elapsed += dt;

  if (reel.stopRequested) {
    reel.phase = 'EASE_OUT';
    reel.elapsed = 0;
    reel.targetOffset = calculateTargetOffset(reel);
  }
}
```

#### EASE_OUT — 減速階段

```typescript
/**
 * 速度由 maxSpeed 減速至 0，同時精確定位到目標符號位置。
 * 使用減速曲線確保平滑停止。
 */
function updateEaseOut(reel: ReelState, dt: number): void {
  reel.elapsed += dt;
  const progress = Math.min(reel.elapsed / reel.config.spinConfig.easeOutDuration, 1);
  const eased = 1 - applyEasing(progress, reel.config.spinConfig.easeOutCurve);
  reel.speed = reel.config.spinConfig.maxSpeed * eased;

  if (progress >= 1) {
    reel.offset = reel.targetOffset;
    reel.speed = 0;
    reel.phase = reel.config.bounceConfig.enabled ? 'BOUNCE' : 'IDLE';
    reel.elapsed = 0;
  }
}
```

### 4.3 Stop 機制

每條轉輪的停輪時機由兩部分決定：

```
停輪延遲 = baseStopDelay + reelIndex × staggerDelay
```

| 參數 | 說明 | 範例 |
|------|------|------|
| `baseStopDelay` | 全局基礎延遲 | 0 ms |
| `staggerDelay` | 每軸遞增延遲 | 200 ms |
| 第 0 軸延遲 | 0 + 0 × 200 | 0 ms |
| 第 1 軸延遲 | 0 + 1 × 200 | 200 ms |
| 第 2 軸延遲 | 0 + 2 × 200 | 400 ms |

**Quick Stop（急停）**：呼叫 `quickStop()` 時，所有軸立即進入 EASE_OUT，忽略 `staggerDelay`，使用更短的 `easeOutDuration`（如 150ms）快速停輪。

```typescript
function quickStop(engine: ReelEngine): void {
  for (const reel of engine.reels) {
    if (reel.phase === 'IDLE') continue;
    reel.stopRequested = true;
    reel.phase = 'EASE_OUT';
    reel.elapsed = 0;
    reel.config.spinConfig.easeOutDuration = 150; // 覆寫為急停時長
    reel.targetOffset = calculateTargetOffset(reel);
  }
}
```

**目標位置計算**：停輪時需將轉輪精確對齊到符號網格上：

```typescript
function calculateTargetOffset(reel: ReelState): number {
  const { symbolSize, symbolGap } = reel.config;
  const cellHeight = symbolSize.h + symbolGap.y;
  const resultIndex = reel.resultSymbolIndex;
  return resultIndex * cellHeight;
}
```

### 4.4 Bounce 效果

停輪後的回彈動畫模擬物理阻尼振盪，讓停輪感覺更自然：

```
位移（Offset）
  ▲
  │   ╱╲
  │  ╱  ╲        ╱╲
  │ ╱    ╲      ╱  ╲      ╱╲
──┼╱──────╲────╱────╲────╱──╲───────▶ 時間
  │        ╲  ╱      ╲  ╱    ╲
  │         ╲╱        ╲╱      ▔▔▔ (收斂至 0)
  │
  amplitude × damping^n
```

**阻尼振盪公式**：

```typescript
function calculateBounceOffset(
  elapsed: number,
  config: BounceConfig
): number {
  const { amplitude, count, damping, duration } = config;
  const progress = Math.min(elapsed / duration, 1);
  const frequency = count * Math.PI * 2;
  const decay = Math.pow(damping, progress * count);
  const oscillation = Math.sin(progress * frequency);
  return amplitude * decay * oscillation * (1 - progress);
}
```

| 參數 | 預設值 | 效果說明 |
|------|--------|---------|
| `amplitude` | 20 px | 首次回彈的最大偏移量 |
| `count` | 2 | 振盪次數 |
| `damping` | 0.5 | 每次振盪振幅衰減為前次的 50% |
| `duration` | 300 ms | 回彈動畫總時長 |

### 4.5 Speed / Easing 曲線

系統內建多種 Easing 函式，亦支援自訂擴充：

```
power2.in                 power2.out                linear

速度 ▲                    速度 ▲                    速度 ▲
     │          ╱              │  ╲                       │  ╱
     │        ╱                │    ╲                     │╱
     │      ╱                  │      ╲                   ╱
     │    ╱                    │        ╲               ╱│
     │  ╱                      │          ╲           ╱  │
     │╱                        │            ╲       ╱    │
     └──────────▶ 時間         └──────────▶ 時間    └──────▶ 時間
```

```typescript
type EasingFunction = (t: number) => number;

const EASING_FUNCTIONS: Record<string, EasingFunction> = {
  'linear':      (t) => t,
  'power2.in':   (t) => t * t,
  'power2.out':  (t) => 1 - (1 - t) * (1 - t),
  'power2.inOut':(t) => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2,
  'power3.in':   (t) => t * t * t,
  'power3.out':  (t) => 1 - Math.pow(1 - t, 3),
  'back.out':    (t) => { const c = 1.70158; return 1 + (c + 1) * Math.pow(t - 1, 3) + c * Math.pow(t - 1, 2); },
  'elastic.out': (t) => t === 0 ? 0 : t === 1 ? 1 : Math.pow(2, -10 * t) * Math.sin((t * 10 - 0.75) * (2 * Math.PI) / 3) + 1,
};

function applyEasing(progress: number, curveName: string): number {
  const fn = EASING_FUNCTIONS[curveName];
  if (!fn) throw new Error(`Unknown easing: ${curveName}`);
  return fn(progress);
}
```

### 4.6 Ticker 整合

ReelEngine 在 PixiJS Ticker 的 `update` 回呼中執行所有運動計算：

```typescript
class ReelEngine {
  private reels: ReelState[];
  private ticker: Ticker;

  constructor(app: Application, config: ReelConfig) {
    this.ticker = app.ticker;
    this.reels = this.initReels(config);
    this.ticker.add(this.update, this);
  }

  private update(ticker: Ticker): void {
    const dt = ticker.deltaMS;
    for (const reel of this.reels) {
      switch (reel.phase) {
        case 'EASE_IN':  this.updateEaseIn(reel, dt);  break;
        case 'CONSTANT': this.updateConstant(reel, dt); break;
        case 'EASE_OUT': this.updateEaseOut(reel, dt);  break;
        case 'BOUNCE':   this.updateBounce(reel, dt);   break;
        case 'IDLE':     break;
      }
      if (reel.phase !== 'IDLE') {
        this.moveSymbols(reel, dt);
      }
    }
  }

  private moveSymbols(reel: ReelState, dt: number): void {
    const displacement = reel.speed * (dt / 1000);
    reel.offset += displacement;

    const { symbolSize, symbolGap } = reel.config;
    const cellHeight = symbolSize.h + symbolGap.y;

    // 回收超出視窗的符號，從物件池取得新符號補上
    while (reel.offset >= cellHeight) {
      reel.offset -= cellHeight;
      const topSymbol = reel.visibleSymbols.shift()!;
      this.symbolPool.release(topSymbol);
      const nextId = this.getNextSymbolId(reel);
      const newSymbol = this.symbolPool.acquire(nextId);
      reel.visibleSymbols.push(newSymbol);
    }

    this.updateSymbolPositions(reel);
  }
}
```

---

## 5. Symbol 物件池（SymbolPool）

物件池模式避免在高速旋轉時頻繁建立 / 銷毀 PixiJS 節點，顯著降低 GC 壓力。

### 5.1 架構設計

```
SymbolPool
├── pools: Map<string, PixiNode[]>    ← 每種 symbolId 各有一個池
├── active: Set<PixiNode>             ← 目前場景中使用中的節點
└── stats: PoolStats                  ← 監控數據

acquire(symbolId)  →  從 pools[symbolId] 取出一個節點
                      若池空則自動擴充
release(node)      →  將節點從 active 移回 pools[symbolId]
                      重設位置 / 透明度 / 動畫狀態
```

### 5.2 API 定義

```typescript
interface PoolStats {
  /** 各 symbolId 的池大小（含可用 + 使用中） */
  sizes: Record<string, number>;
  /** 各 symbolId 的使用中數量 */
  activeCount: Record<string, number>;
  /** 歷史峰值使用量 */
  peakUsage: Record<string, number>;
  /** 總節點數 */
  totalNodes: number;
}

class SymbolPool {
  private pools: Map<string, Container[]> = new Map();
  private active: Set<Container> = new Set();
  private stats: PoolStats;

  /**
   * 從物件池取得一個可用的符號節點。
   * 若池中無可用節點，自動擴充 batchSize 個。
   */
  acquire(symbolId: string): Container {
    const pool = this.pools.get(symbolId);
    if (!pool || pool.length === 0) {
      this.expand(symbolId, this.batchSize);
    }
    const node = this.pools.get(symbolId)!.pop()!;
    this.resetNode(node);
    node.visible = true;
    this.active.add(node);
    this.updateStats(symbolId);
    return node;
  }

  /**
   * 將符號節點歸還至物件池。
   */
  release(node: Container): void {
    node.visible = false;
    node.removeFromParent();
    this.active.delete(node);
    const symbolId = (node as any).__symbolId;
    this.pools.get(symbolId)!.push(node);
    this.updateStats(symbolId);
  }

  /**
   * 批量擴充指定符號的物件池。
   */
  private expand(symbolId: string, count: number): void {
    if (!this.pools.has(symbolId)) {
      this.pools.set(symbolId, []);
    }
    const pool = this.pools.get(symbolId)!;
    for (let i = 0; i < count; i++) {
      const node = this.createSymbolNode(symbolId);
      (node as any).__symbolId = symbolId;
      node.visible = false;
      pool.push(node);
    }
  }

  /**
   * 根據符號定義建立 Sprite 或 Spine 節點。
   */
  private createSymbolNode(symbolId: string): Container {
    const def = this.symbolDefs[symbolId];
    if (def.type === 'spine') {
      return this.createSpineNode(def);
    }
    return this.createSpriteNode(def);
  }

  /**
   * 重設節點狀態（位置、縮放、透明度、動畫）。
   */
  private resetNode(node: Container): void {
    node.position.set(0, 0);
    node.scale.set(1, 1);
    node.alpha = 1;
    node.rotation = 0;
  }

  /** 取得物件池監控數據。 */
  getStats(): PoolStats {
    return { ...this.stats };
  }

  /** 釋放所有節點，清空物件池。 */
  dispose(): void {
    for (const [, pool] of this.pools) {
      for (const node of pool) {
        node.destroy({ children: true });
      }
    }
    for (const node of this.active) {
      node.destroy({ children: true });
    }
    this.pools.clear();
    this.active.clear();
  }
}
```

### 5.3 預分配策略

```typescript
/**
 * 根據盤面配置計算初始池大小。
 * 每軸需要 (rows + bufferRows) 個符號，bufferRows 用於旋轉時上下方的額外符號。
 */
function calculateInitialPoolSize(config: ReelConfig): Record<string, number> {
  const bufferRows = 2; // 上下各多 1 行緩衝
  const totalSlots = config.rowsPerReel.reduce(
    (sum, rows) => sum + rows + bufferRows, 0
  );
  const symbolIds = Object.keys(config.symbols);
  const perSymbol = Math.ceil(totalSlots / symbolIds.length) + 2;

  const sizes: Record<string, number> = {};
  for (const id of symbolIds) {
    sizes[id] = perSymbol;
  }
  return sizes;
}
```

### 5.4 記憶體管理

| 策略 | 說明 |
|------|------|
| **惰性擴充** | 池空時自動擴充 `batchSize`（預設 4）個節點，避免一次分配過多 |
| **峰值追蹤** | 記錄每種符號的歷史最大使用量，作為未來預分配的參考 |
| **手動縮減** | 提供 `shrink()` API，將池大小縮減至峰值 + 緩衝，釋放多餘記憶體 |
| **完整釋放** | `dispose()` 銷毀所有節點，在配置變更或頁面卸載時呼叫 |

---

## 6. Reel Clipping（轉輪裁切）

每條轉輪需要一個裁切遮罩（Mask），確保符號只在可見視窗內顯示，超出範圍的部分被隱藏。

### 6.1 裁切原理

```
                ┌─ Mask 邊界（僅此區域內可見）
                │
     ···········▼···········
     :  ┌──────────────┐  :
 隱藏 :  │  Symbol A    │  :  ← 從上方進入
     :  ├──────────────┤  :
─────╋══╪══════════════╪══╋───── Mask 頂部
     ║  │  Symbol B    │  ║
     ║  ├──────────────┤  ║
     ║  │  Symbol C    │  ║  ← 可見區域
     ║  ├──────────────┤  ║
     ║  │  Symbol D    │  ║
─────╋══╪══════════════╪══╋───── Mask 底部
     :  ├──────────────┤  :
 隱藏 :  │  Symbol E    │  :  ← 從下方離開
     :  └──────────────┘  :
     ························
```

### 6.2 Mask 建立

```typescript
function createReelMask(
  reelIndex: number,
  config: ReelConfig
): Graphics {
  const { symbolSize, symbolGap, rowsPerReel, alignment } = config;
  const rows = rowsPerReel[reelIndex];

  const maskWidth = symbolSize.w;
  const maskHeight = rows * symbolSize.h + (rows - 1) * symbolGap.y;

  const x = reelIndex * (symbolSize.w + symbolGap.x);
  const y = calculateAlignmentOffset(reelIndex, config);

  const mask = new Graphics();
  mask.rect(x, y, maskWidth, maskHeight);
  mask.fill({ color: 0xffffff });

  return mask;
}

/**
 * 根據對齊方式計算每軸的 Y 偏移量。
 */
function calculateAlignmentOffset(
  reelIndex: number,
  config: ReelConfig
): number {
  const { symbolSize, symbolGap, rowsPerReel, alignment } = config;
  const maxRows = Math.max(...rowsPerReel);
  const currentRows = rowsPerReel[reelIndex];

  const maxHeight = maxRows * symbolSize.h + (maxRows - 1) * symbolGap.y;
  const currentHeight = currentRows * symbolSize.h + (currentRows - 1) * symbolGap.y;

  switch (alignment) {
    case 'top':    return 0;
    case 'center': return (maxHeight - currentHeight) / 2;
    case 'bottom': return maxHeight - currentHeight;
  }
}
```

### 6.3 套用 Mask 至 Container

```typescript
function setupReelContainer(
  reelIndex: number,
  config: ReelConfig,
  parentContainer: Container
): Container {
  const reelContainer = new Container();
  const mask = createReelMask(reelIndex, config);

  parentContainer.addChild(mask);
  parentContainer.addChild(reelContainer);
  reelContainer.mask = mask;

  return reelContainer;
}
```

### 6.4 自訂裁切形狀

除了矩形，系統支援自訂 Graphics 路徑作為裁切區域，適用於特殊盤面設計：

```typescript
function createCustomReelMask(path: { x: number; y: number }[]): Graphics {
  const mask = new Graphics();
  mask.moveTo(path[0].x, path[0].y);
  for (let i = 1; i < path.length; i++) {
    mask.lineTo(path[i].x, path[i].y);
  }
  mask.closePath();
  mask.fill({ color: 0xffffff });
  return mask;
}
```

### 6.5 動態更新

當盤面配置變更（列數、符號尺寸、間距）時，所有 Mask 需同步更新：

```typescript
function updateAllMasks(config: ReelConfig, masks: Graphics[]): void {
  for (let i = 0; i < config.reels; i++) {
    const mask = masks[i];
    mask.clear();
    const rows = config.rowsPerReel[i];
    const maskWidth = config.symbolSize.w;
    const maskHeight = rows * config.symbolSize.h + (rows - 1) * config.symbolGap.y;
    const x = i * (config.symbolSize.w + config.symbolGap.x);
    const y = calculateAlignmentOffset(i, config);
    mask.rect(x, y, maskWidth, maskHeight);
    mask.fill({ color: 0xffffff });
  }
}
```

---

## 7. 轉輪控制面板 UI

轉輪控制面板是美術人員操作轉輪的主要介面，提供直覺的控制項與即時回饋。

### 7.1 面板佈局

```
┌──────────────────────────────────────────────────────────────┐
│  轉輪控制                                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  操作                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  ▶ 旋轉  │  │  ■ 停輪  │  │  ⏩ 急停  │  │ ⚙ 盤面  │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│                                                              │
│  ─────────────────────────────────────────────────           │
│                                                              │
│  結果設定                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  軸0   軸1   軸2   軸3   軸4                         │    │
│  │ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                      │    │
│  │ │ 2 │ │ 5 │ │ 1 │ │ 3 │ │ 7 │  ← 第 0 列           │    │
│  │ ├───┤ ├───┤ ├───┤ ├───┤ ├───┤                      │    │
│  │ │ 8 │ │ 0 │ │ 4 │ │ 6 │ │ 9 │  ← 第 1 列           │    │
│  │ ├───┤ ├───┤ ├───┤ ├───┤ ├───┤                      │    │
│  │ │ 3 │ │ 7 │ │ 2 │ │ 1 │ │ 5 │  ← 第 2 列           │    │
│  │ └───┘ ├───┤ ├───┤ ├───┤ └───┘                      │    │
│  │       │ 4 │ │ 8 │ │ 0 │         ← 第 3 列           │    │
│  │       └───┘ ├───┤ └───┘                             │    │
│  │             │ 6 │                ← 第 4 列           │    │
│  │             └───┘                                    │    │
│  │  ┌─────────┐                                        │    │
│  │  │ 隨機填充 │                                        │    │
│  │  └─────────┘                                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ─────────────────────────────────────────────────           │
│                                                              │
│  旋轉參數                                                    │
│  最大速度    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [2400] px/s      │
│  加速時間    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [300]  ms        │
│  等速時間    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [800]  ms        │
│  減速時間    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [500]  ms        │
│  停輪間隔    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [200]  ms        │
│                                                              │
│  加速曲線    [power2.in  ▼]                                  │
│  減速曲線    [power2.out ▼]                                  │
│                                                              │
│  ─────────────────────────────────────────────────           │
│                                                              │
│  回彈效果                                                    │
│  啟用回彈    [✓]                                             │
│  振幅        ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [20]  px        │
│  次數        ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [2]             │
│  衰減係數    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [0.5]            │
│  持續時間    ◄━━━━━━━━━━━━━━━━━━━━━━━━━━►  [300]  ms        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 元件結構

```typescript
<SpinControlPanel>
  <ActionBar>
    <SpinButton />           {/* 旋轉按鈕 */}
    <StopButton />           {/* 停輪按鈕 */}
    <QuickStopButton />      {/* 急停按鈕 */}
    <GridEditorButton />     {/* 開啟盤面編輯器 */}
  </ActionBar>

  <ResultGrid>               {/* 結果矩陣編輯器 */}
    <SymbolCell />           {/* 可拖曳 / 輸入的符號格 ×N */}
    <RandomFillButton />     {/* 隨機填充按鈕 */}
  </ResultGrid>

  <SpinSettings>             {/* 旋轉參數 Slider 組 */}
    <SliderRow label="最大速度"  ... />
    <SliderRow label="加速時間"  ... />
    <SliderRow label="等速時間"  ... />
    <SliderRow label="減速時間"  ... />
    <SliderRow label="停輪間隔"  ... />
    <EasingSelect label="加速曲線" ... />
    <EasingSelect label="減速曲線" ... />
  </SpinSettings>

  <BounceSettings>           {/* 回彈參數 Slider 組 */}
    <Toggle label="啟用回彈" ... />
    <SliderRow label="振幅"    ... />
    <SliderRow label="次數"    ... />
    <SliderRow label="衰減係數" ... />
    <SliderRow label="持續時間" ... />
  </BounceSettings>
</SpinControlPanel>
```

### 7.3 按鈕狀態機

```
                spin()
  ┌──────────┐ ─────────▶ ┌──────────────┐
  │   IDLE   │            │  SPINNING    │
  │          │ ◀───────── │              │
  └──────────┘  all idle  └──────┬───────┘
                                 │ stop() / quickStop()
                                 ▼
                          ┌──────────────┐
                          │  STOPPING    │
                          │              │──▶ IDLE (all reels stopped)
                          └──────────────┘

按鈕啟用狀態：
┌──────────┬──────────┬────────────┬──────────┐
│  狀態    │  ▶ 旋轉  │  ■ 停輪    │  ⏩ 急停  │
├──────────┼──────────┼────────────┼──────────┤
│  IDLE    │  啟用    │  停用      │  停用    │
│ SPINNING │  停用    │  啟用      │  啟用    │
│ STOPPING │  停用    │  停用      │  啟用    │
└──────────┴──────────┴────────────┴──────────┘
```

### 7.4 結果矩陣編輯

結果矩陣讓美術人員指定停輪後各位置顯示的符號，操作方式：

| 操作 | 說明 |
|------|------|
| **直接輸入** | 點擊格子，輸入符號 ID（數字） |
| **下拉選擇** | 點擊格子展開符號清單，含縮圖預覽 |
| **拖曳填入** | 從符號清單拖曳符號到格子 |
| **隨機填充** | 一鍵隨機產生整個結果矩陣 |
| **長按複製** | 長按格子複製符號 ID，再點擊其他格子貼上 |

```typescript
interface ResultMatrix {
  /** result[reelIndex][rowIndex] = symbolId */
  [reelIndex: number]: number[];
}

// 範例：5 軸盤面 rowsPerReel = [3, 4, 5, 4, 3]
const exampleResult: ResultMatrix = {
  0: [2, 8, 3],
  1: [5, 0, 7, 4],
  2: [1, 4, 2, 8, 6],
  3: [3, 6, 1, 0],
  4: [7, 9, 5],
};
```

---

## 8. 擴充性

轉輪系統的架構在設計時已預留多個擴充點，確保未來能支援更複雜的需求。

### 8.1 新增轉輪類型

透過擴展 `ReelConfig` 介面新增參數，而不修改現有欄位：

```typescript
interface CascadeReelConfig extends ReelConfig {
  /** 消除後的掉落動畫設定 */
  cascadeConfig: {
    fallSpeed: number;
    fallEasing: string;
    delayPerRow: number;
  };
}

interface HoldReelConfig extends ReelConfig {
  /** Hold & Spin 模式設定 */
  holdConfig: {
    holdPositions: { reel: number; row: number }[];
    respinCount: number;
  };
}
```

### 8.2 自訂 Easing 函式

透過註冊機制新增 Easing 函式，無須修改核心邏輯：

```typescript
// 註冊自訂 Easing
EasingRegistry.register('customBounce', (t: number) => {
  if (t < 0.5) return 4 * t * t * t;
  return 1 - Math.pow(-2 * t + 2, 3) / 2;
});

// 在 ReelConfig 中使用
const config: ReelConfig = {
  // ...
  spinConfig: {
    easeInCurve: 'customBounce',
    easeOutCurve: 'power2.out',
    // ...
  },
};
```

### 8.3 自訂停輪行為

每條轉輪可覆寫預設的停輪邏輯：

```typescript
interface CustomStopBehavior {
  /** 自訂減速階段的位移計算 */
  calculateEaseOut(reel: ReelState, dt: number): number;
  /** 自訂目標位置計算 */
  calculateTargetOffset(reel: ReelState): number;
  /** 自訂停輪後效果（如爆炸、閃光） */
  onStopped?(reel: ReelState): void;
}

// 使用範例：Anticipation 停輪效果（先向上拉再落下）
const anticipationStop: CustomStopBehavior = {
  calculateEaseOut(reel, dt) {
    const progress = reel.elapsed / reel.config.spinConfig.easeOutDuration;
    if (progress < 0.3) {
      return -reel.config.spinConfig.maxSpeed * 0.2; // 短暫反向
    }
    return reel.config.spinConfig.maxSpeed * (1 - progress);
  },
  calculateTargetOffset(reel) {
    return calculateTargetOffset(reel);
  },
  onStopped(reel) {
    // 觸發停輪特效
  },
};
```

### 8.4 非矩形盤面

架構已預留非矩形盤面的擴充空間（如六角形、菱形）：

```typescript
interface NonRectangularGridConfig {
  type: 'hexagonal' | 'diamond' | 'custom';
  /** 自訂格位座標（像素座標） */
  cellPositions: { reel: number; row: number; x: number; y: number }[];
  /** 自訂裁切路徑 */
  clipPath: { x: number; y: number }[];
}
```

> ⚠️ 此功能為未來擴充點，目前不實作。核心架構（ReelEngine、SymbolPool、ClipManager）已確保新增此功能時不需重構現有程式碼。

### 8.5 擴充路線圖

| 階段 | 功能 | 優先級 |
|------|------|--------|
| v1.0 | 標準 NxM 盤面 + 可變列數 | ★★★ |
| v1.1 | Cascade（消除掉落） | ★★☆ |
| v1.2 | Hold & Spin | ★★☆ |
| v2.0 | 非矩形盤面 | ★☆☆ |
| v2.0 | 自訂物理引擎（如 3D 轉輪） | ★☆☆ |

---

## 附錄 A：狀態流程圖

完整的旋轉流程——從使用者按下「旋轉」到所有軸停輪完畢：

```
使用者按下 [▶ 旋轉]
       │
       ▼
 UI dispatch spin()
       │
       ▼
 reelStore.setState({ spinState: 'SPINNING' })
       │
       ▼
 ReelEngine.spin()
       │
       ├──▶ Reel 0: IDLE → EASE_IN → CONSTANT ─┐
       ├──▶ Reel 1: IDLE → EASE_IN → CONSTANT ─┤
       ├──▶ Reel 2: IDLE → EASE_IN → CONSTANT ─┤  (各軸同時啟動)
       ├──▶ Reel 3: IDLE → EASE_IN → CONSTANT ─┤
       └──▶ Reel 4: IDLE → EASE_IN → CONSTANT ─┘
                                                 │
 使用者按下 [■ 停輪] 或 auto-stop ──────────────┘
       │
       ▼
 reelStore.setState({ resultMatrix: [...] })
       │
       ├──▶ Reel 0 (+  0ms): CONSTANT → EASE_OUT → BOUNCE → IDLE
       ├──▶ Reel 1 (+200ms): CONSTANT → EASE_OUT → BOUNCE → IDLE
       ├──▶ Reel 2 (+400ms): CONSTANT → EASE_OUT → BOUNCE → IDLE
       ├──▶ Reel 3 (+600ms): CONSTANT → EASE_OUT → BOUNCE → IDLE
       └──▶ Reel 4 (+800ms): CONSTANT → EASE_OUT → BOUNCE → IDLE
                                                              │
                                                              ▼
                                              reelStore.setState({
                                                spinState: 'IDLE'
                                              })
                                                              │
                                                              ▼
                                                    UI 更新按鈕狀態
```

---

## 附錄 B：效能指標

| 指標 | 目標 |
|------|------|
| 旋轉時 FPS | ≥ 60 fps |
| 符號切換延遲 | < 1 ms（物件池 acquire + release） |
| 記憶體佔用（5×3 盤面） | ≤ 30 MB（含所有符號貼圖） |
| 物件池預分配時間 | < 100 ms |
| 配置變更響應 | < 16 ms（一幀內完成 Mask 重建） |

---

> **上一份文件**：`04-spine-system.md` — Spine 動畫系統設計規格
> **下一份文件**：`06-script-engine.md` — 腳本引擎設計規格
