# Slot Previewer — UI 佈局與元件架構

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置閱讀**：`01-architecture-overview.md`、`02-module-design.md`
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 1. 整體佈局

### 1.1 佈局結構

應用程式採用經典的**三欄式佈局**，搭配頂部工具列與底部狀態列，構成完整的編輯器介面：

```
┌──────────┬──────────────────────────────────────┬──────────────────┐
│          │            頂部工具列 (48px)          │                  │
│          ├──────────────────────────────────────┤                  │
│          │                                      │                  │
│   左側   │                                      │       右側       │
│   圖層   │                                      │       屬性       │
│   面板   │          Canvas 預覽區                │       面板       │
│          │          (PixiJS)                     │                  │
│          │                                      │    (280px 預設,  │
│ (240px)  │                                      │     可拖拽至     │
│          │                                      │      600px)      │
│          │                                      │                  │
│          │                                      │                  │
│          ├──────────────────────────────────────┤                  │
│          │           底部狀態列 (32px)           │                  │
└──────────┴──────────────────────────────────────┴──────────────────┘
```

### 1.2 區域規格

| 區域 | 尺寸 | 行為 |
|------|------|------|
| **左側面板** | 寬度 240px，固定 | 可收合（收合後僅留 0px，以 toggle 按鈕展開） |
| **中央 Canvas** | 彈性寬度，佔滿剩餘空間 | 內嵌 PixiJS Canvas，支援縮放（Zoom）與平移（Pan） |
| **右側面板** | 預設 280px，可拖拽 280–600px | 左側邊緣拖拽手柄，可收合 |
| **頂部工具列** | 高度 48px，全寬 | 固定於頂部，橫跨左側面板與中央區域 |
| **底部狀態列** | 高度 32px | 僅位於中央 Canvas 下方，顯示效能指標 |

### 1.3 CSS Grid 佈局實作

整體佈局使用 CSS Grid 實作，確保面板收合時 Canvas 區域自動擴展：

```typescript
// Layout.tsx — 主佈局結構
<div className="h-screen w-screen overflow-hidden bg-[var(--color-bg-primary)]"
     style={{
       display: 'grid',
       gridTemplateColumns: `${leftCollapsed ? '0px' : '240px'} 1fr ${rightCollapsed ? '0px' : `${rightWidth}px`}`,
       gridTemplateRows: '48px 1fr 32px',
     }}>

  {/* 頂部工具列 — 橫跨左側面板與中央區域（col 1-2） */}
  <Toolbar className="col-span-2" />

  {/* 左側圖層面板 */}
  <LeftPanel />

  {/* 中央 Canvas（含頂部工具列下方到底部狀態列上方） */}
  <CanvasArea />

  {/* 右側屬性面板 — 橫跨所有列（row 1-3） */}
  <RightPanel style={{ gridRow: '1 / -1' }} />

  {/* 底部狀態列 — 僅中央區域 */}
  <StatusBar />
</div>
```

### 1.4 面板收合行為

| 面板 | 收合方式 | 展開觸發 |
|------|----------|----------|
| 左側面板 | 寬度動畫過渡至 0px，`overflow: hidden` | 頂部工具列上的 toggle 按鈕（Sidebar icon） |
| 右側面板 | 寬度動畫過渡至 0px，`overflow: hidden` | 頂部工具列上的 toggle 按鈕（PanelRight icon） |

收合動畫使用 `transition-all duration-300 ease-in-out`，搭配 `prefers-reduced-motion` 時降為 `duration-0`。

---

## 2. 左側面板 — 圖層面板

左側面板為固定寬度 240px 的圖層管理介面，使用 Glass Panel 風格：

```
┌────────────────────────────────┐
│  圖層面板               [−]   │ ← 標題列（含收合按鈕）
├────────────────────────────────┤
│  [+ 新增 ▾]  [🗑 刪除]  [📁]  │ ← 操作按鈕列
├────────────────────────────────┤
│                                │
│  ▼ 📁 主場景                   │ ← 群組圖層（可展開/收合）
│    👁 🔒  🦴 base_game_spine  │ ← Spine 圖層
│    👁 🔒  🎰 reel_container   │ ← 轉輪圖層
│    👁 🔒  🖼 background       │ ← Sprite 圖層
│  ▶ 📁 前景特效                 │ ← 收合的群組
│    👁 🔒  🦴 win_celebration  │
│                                │
│         （可拖拽排序）          │
│                                │
└────────────────────────────────┘
```

### 2.1 圖層樹（LayerTree）

圖層樹以遞迴樹狀結構呈現所有圖層節點，對應 `LayerNode` 資料模型。

#### 圖層項目結構

每個圖層項目水平排列以下元素：

```
[展開/收合] [可見性] [鎖定] [類型圖示] [名稱]        [更多]
    ▼          👁      🔒     🦴     "spine_anim"    ⋯
```

| 元素 | 互動行為 |
|------|----------|
| **展開/收合**（ChevronRight） | 僅群組圖層顯示，點擊切換子層展開/收合 |
| **可見性**（Eye / EyeOff） | 點擊切換圖層可見性，映射至 `LayerNode.visible` |
| **鎖定**（Lock / Unlock） | 點擊切換鎖定狀態，鎖定時無法在 Canvas 拖拽該圖層 |
| **類型圖示** | 依圖層類型顯示對應 icon：🦴 Spine、🖼 Sprite、🎰 Reel、📁 Group |
| **名稱** | 單擊選取，雙擊進入重新命名模式（inline input） |
| **更多**（MoreHorizontal） | 點擊開啟右鍵選單 |

#### 互動行為

| 操作 | 行為 |
|------|------|
| **單擊** | 選取圖層，右側屬性面板同步顯示該圖層屬性 |
| **雙擊名稱** | 進入 inline 重新命名模式，按 Enter 確認、Esc 取消 |
| **拖拽** | 拖拽圖層項目可重新排列 zIndex 順序（使用 `@dnd-kit/core`） |
| **右鍵** | 顯示 Context Menu：重新命名、刪除、複製、群組/解除群組 |
| **多選** | Ctrl+Click 追加選取，Shift+Click 範圍選取 |

#### 樣式規範

```typescript
// 圖層項目基本樣式
const layerItemClass = cn(
  'flex items-center gap-1 px-2 h-8 rounded-md cursor-pointer',
  'text-sm text-[var(--color-text-secondary)]',
  'hover:bg-[var(--color-bg-hover)]',
  'transition-colors duration-150',
);

// 選取狀態
const selectedClass = 'bg-[var(--color-accent)]/15 text-[var(--color-text-primary)] border-l-2 border-[var(--color-accent)]';

// 拖拽中狀態
const draggingClass = 'opacity-50 ring-1 ring-[var(--color-accent)]/30';
```

### 2.2 圖層操作按鈕

位於圖層樹上方，提供快速操作：

| 按鈕 | 變體 | 行為 |
|------|------|------|
| **新增圖層** | Ghost + Dropdown | 下拉選單：新增 Spine / Sprite / Group / Reel 圖層 |
| **刪除圖層** | Ghost | 刪除目前選取的圖層（需確認對話框） |
| **群組** | Ghost | 將選取的多個圖層包裝進新的 Group |

下拉選單使用 shadcn/ui 的 `DropdownMenu` 元件，搭配 Glass Panel 風格背景。

---

## 3. 右側面板 — 屬性面板

右側面板為**可調整寬度的 Tab 面板**，預設 280px，可拖拽至 600px。使用 `useResizablePanel` hook 實作拖拽調整。

```
┌──────────────────────────────────┐
│ [屬性] [Spine] [轉輪] [腳本]     │ ← Tab 列
├──────────────────────────────────┤
│                                  │
│        （依選中 Tab 顯示）        │
│        （可垂直捲動）             │
│                                  │
│                                  │
│                                  │
│                                  │
└──────────────────────────────────┘
  ↑
  拖拽手柄（左側邊緣 4px）
```

### 3.1 Tab 列設計

Tab 列使用 `role="tablist"` 語意，每個 Tab 使用 `role="tab"` + `aria-selected`：

```typescript
const tabs = [
  { id: 'properties', label: '屬性',   icon: Settings2 },
  { id: 'spine',      label: 'Spine',  icon: Bone },
  { id: 'reel',       label: '轉輪',   icon: Dices },
  { id: 'script',     label: '腳本',   icon: FileCode },
];
```

| 狀態 | 樣式 |
|------|------|
| **預設** | `text-[var(--color-text-muted)]`，無底線 |
| **Hover** | `text-[var(--color-text-secondary)]` |
| **選中** | `text-[var(--color-accent)]`，底部 2px accent 色底線，icon 使用 `icon-breathe` 呼吸光暈動畫 |

### 3.2 Tab: 屬性（Properties）

當選取任一圖層時，顯示該圖層的通用屬性。無圖層選取時顯示提示文字「請選取一個圖層」。

```
┌──────────────────────────────────┐
│ ◆ 變換                           │
├──────────────────────────────────┤
│  位置    X [  120  ]  Y [  340  ]│
│  縮放    X [ 1.00  ]  Y [ 1.00  ]│
│  旋轉      [  0.0  ] °           │
├──────────────────────────────────┤
│ ◆ 顯示                           │
├──────────────────────────────────┤
│  透明度  ━━━━━━━━━━━●━━ [0.85]   │
│  混合模式  [ NORMAL       ▾ ]    │
│  可見     [✓]                    │
├──────────────────────────────────┤
│ ◆ 裁切                           │
├──────────────────────────────────┤
│  裁切遮罩  [ 無 ▾ ]              │
│  裁切區域  X[0] Y[0] W[0] H[0]  │
└──────────────────────────────────┘
```

#### 變換（Transform）區段

| 欄位 | 元件 | 規格 |
|------|------|------|
| 位置 X / Y | `NumberInput` | step=1，無 min/max 限制 |
| 縮放 X / Y | `NumberInput` | step=0.01，min=0.01 |
| 旋轉 | `NumberInput` | step=1，單位：度（°），range: -360 ~ 360 |

#### 顯示（Display）區段

| 欄位 | 元件 | 規格 |
|------|------|------|
| 透明度 | `Slider` (default) | min=0, max=1, step=0.01，`--slider-color: var(--color-accent)` |
| 混合模式 | `Select` (shadcn/ui) | 選項：NORMAL, ADD, MULTIPLY, SCREEN |
| 可見 | `Checkbox` | 切換 `LayerNode.visible` |

#### 裁切（Clip）區段

| 欄位 | 元件 | 規格 |
|------|------|------|
| 裁切遮罩 | `Select` | 下拉選取其他圖層作為 mask source，或「無」 |
| 裁切區域 | 4 × `NumberInput` | X, Y, W, H 四欄，定義矩形裁切區域 |

### 3.3 Tab: Spine 控制

當選取的圖層類型為 Spine 時啟用，否則顯示「請選取一個 Spine 圖層」。

```
┌──────────────────────────────────┐
│ ◆ 動畫列表                       │
├──────────────────────────────────┤
│  ▸ idle               [循環]     │ ← 目前播放中（高亮）
│    walk                          │
│    attack                        │
│    win_celebration               │
├──────────────────────────────────┤
│ ◆ 軌道混合器                     │
├──────────────────────────────────┤
│  Track 0  idle     α ━━━●━ 1.0  │
│  Track 1  (none)   α ━●━━━ 0.5  │
│  Track 2  (none)   α ●━━━━ 0.0  │
├──────────────────────────────────┤
│ ◆ 控制                           │
├──────────────────────────────────┤
│  皮膚    [ default         ▾ ]   │
│  時間軸  ━━━━━━●━━━━━━━━━━ 0.45s │
│  速度    ━━━━━━━●━━━━━━━━━ 1.0x  │
│           0.1x              4.0x │
└──────────────────────────────────┘
```

#### 動畫列表

- 從 `SpineActor.getAnimationNames()` 取得
- 點擊切換播放動畫
- 目前播放中的動畫以 accent 色高亮
- 右側標記是否為循環動畫

#### 軌道混合器（Track Mixer）

- 顯示 Track 0–2（可擴充）
- 每軌道包含：軌道編號、動畫名稱下拉、Alpha slider（`Slider` compact 變體）
- Alpha slider 使用 `--slider-color: var(--color-accent-secondary)` 紫色系

#### 控制區段

| 欄位 | 元件 | 規格 |
|------|------|------|
| 皮膚 | `Select` | 從 `SpineActor.getSkinNames()` 取得選項 |
| 時間軸 | `Slider` (timeline) | min=0, max=動畫總長，step=0.01，可拖拽 scrub |
| 速度 | `Slider` (default) | min=0.1, max=4.0, step=0.1，顯示倍率值 |

### 3.4 Tab: 轉輪控制

當選取的圖層類型為 Reel 時啟用，否則顯示「請選取一個轉輪圖層」。

```
┌──────────────────────────────────┐
│ ◆ 轉輪操作                       │
├──────────────────────────────────┤
│  [▶ 開始] [⏹ 停止] [⏩ 快停]     │
├──────────────────────────────────┤
│ ◆ 結果輸入                       │
├──────────────────────────────────┤
│  手動結果  [A,B,C,A,B]           │
│            [套用結果]             │
├──────────────────────────────────┤
│ ◆ 盤面設定                       │
├──────────────────────────────────┤
│  [開啟盤面編輯器]                 │ ← 開啟 ReelGridEditor 彈窗
├──────────────────────────────────┤
│ ◆ 運動參數                       │
├──────────────────────────────────┤
│  速度    ━━━━━━━●━━━━━━━ 1.0     │
│  緩動    [ easeOutBack    ▾ ]    │
│  彈跳幅度 ━━━━━●━━━━━━━━ 0.3     │
│  彈跳次數 [ 2 ]                  │
└──────────────────────────────────┘
```

#### 轉輪操作按鈕

| 按鈕 | 變體 | 行為 |
|------|------|------|
| **開始** | Solid (accent) | 啟動轉輪旋轉，對應 `ReelEngine.spin()` |
| **停止** | Ghost | 正常停止（含緩動），對應 `ReelEngine.stop()` |
| **快停** | Ghost | 立即停止，無緩動，對應 `ReelEngine.quickStop()` |

#### 結果輸入

- 文字輸入框，接受逗號分隔的 Symbol ID
- 「套用結果」按鈕將手動結果傳入 `ReelEngine.setResult()`

#### 運動參數

| 欄位 | 元件 | 規格 |
|------|------|------|
| 速度 | `Slider` | min=0.5, max=3.0, step=0.1 |
| 緩動 | `Select` | easeOutBack, easeOutBounce, easeOutElastic 等 |
| 彈跳幅度 | `Slider` | min=0, max=1.0, step=0.05 |
| 彈跳次數 | `NumberInput` | min=0, max=5, step=1 |

### 3.5 Tab: 腳本

管理與執行演出腳本。

```
┌──────────────────────────────────┐
│ ◆ 腳本列表                       │
├──────────────────────────────────┤
│  ▸ base_game_flow.json    [✓]   │ ← 已載入（打勾）
│    free_game_flow.json           │
│    bonus_intro.json              │
├──────────────────────────────────┤
│ ◆ 控制                           │
├──────────────────────────────────┤
│  [📂 載入] [▶ 執行] [⏸ 暫停] [⏹]│
├──────────────────────────────────┤
│ ◆ 執行進度                       │
├──────────────────────────────────┤
│  步驟  3 / 12                    │
│  ━━━━━━━━━●━━━━━━━━━━━━━        │
│  目前動作: PlaySpineAction       │
│  狀態: 執行中                     │
└──────────────────────────────────┘
```

#### 腳本列表

- 顯示所有已載入的腳本檔案
- 點擊選取腳本作為執行目標
- 目前載入中的腳本以打勾標示

#### 控制按鈕

| 按鈕 | 變體 | 行為 |
|------|------|------|
| **載入** | Ghost | 開啟檔案選擇器，匯入 `.json` 腳本 |
| **執行** | Solid (accent) | 從頭或繼續執行選中的腳本 |
| **暫停** | Ghost | 暫停腳本執行（保留目前進度） |
| **停止** | Ghost | 停止並重設腳本進度 |

#### 執行進度

- `ProgressBar` 元件顯示目前步驟 / 總步驟
- 下方文字顯示目前正在執行的 Action 類型與狀態

---

## 4. 頂部工具列

頂部工具列高度 48px，水平排列所有全域操作按鈕。使用 Glass Panel 風格：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [≡] [◀][▶]  SlotPreviewer名稱  │ [新增][開啟][儲存] │ [解析度 ▾] [🎨] │ [−][+] 100% [⊞] │ [▶][⏸][⏹] │
└──────────────────────────────────────────────────────────────────────────────┘
  面板控制    專案名稱               專案操作            視圖控制            縮放控制          播放控制
```

### 4.1 專案控制（ProjectControls）

| 元素 | 元件 | 行為 |
|------|------|------|
| **面板切換** | Icon Button (☰) | 切換左側面板顯示/隱藏 |
| **面板切換** | Icon Button (◀/▶) | 切換右側面板顯示/隱藏 |
| **專案名稱** | Inline editable text | 點擊進入編輯模式，顯示目前專案名稱 |
| **新增專案** | Ghost Button | 重設所有狀態，建立空白專案 |
| **開啟專案** | Ghost Button | 開啟檔案選擇器，匯入 `.zip` 專案檔 |
| **儲存專案** | Ghost Button | 匯出目前專案為 `.zip` 下載 |

### 4.2 視圖控制（ViewControls）

| 元素 | 元件 | 行為 |
|------|------|------|
| **解析度選擇** | Dropdown | 預設尺寸：1920×1080、1280×720、自訂... |
| **背景色** | Color Picker | 設定 Canvas 背景色，預設 `#1a1a2e` |

### 4.3 縮放控制（ZoomControls）

| 元素 | 元件 | 行為 |
|------|------|------|
| **縮小** | Icon Button (Minus) | Canvas 縮放 -10% |
| **放大** | Icon Button (Plus) | Canvas 縮放 +10% |
| **百分比** | Text Display | 顯示目前縮放比例（如 `100%`） |
| **適應視窗** | Icon Button (Maximize2) | 自動縮放至 Canvas 區域最大 fit |

### 4.4 播放控制（PlayControls）

| 元素 | 元件 | 行為 |
|------|------|------|
| **播放/繼續** | Icon Button (Play) | 執行目前腳本 |
| **暫停** | Icon Button (Pause) | 暫停腳本 |
| **停止** | Icon Button (Square) | 停止並重設腳本 |

---

## 5. 底部狀態列

底部狀態列高度 32px，僅位於中央 Canvas 區域下方。顯示效能指標與狀態資訊，文字尺寸為 `text-xs`：

```
┌───────────────────────────────────────────────────────────────────┐
│  FPS: 60  │  DrawCalls: 12  │  Zoom: 100%  │  1920×1080  │  Heap: 48MB  │  ● Ready  │
└───────────────────────────────────────────────────────────────────┘
```

### 5.1 顯示項目

| 項目 | 資料來源 | 更新頻率 |
|------|----------|----------|
| **FPS** | `PixiApp.ticker.FPS` | 每秒更新（取整數） |
| **DrawCalls** | `renderer.renderPipes` 統計 | 每秒更新 |
| **Zoom** | `previewStore.zoom` | 即時更新 |
| **Canvas 解析度** | `previewStore.resolution` | 變更時更新 |
| **JS Heap** | `performance.memory.usedJSHeapSize`（Chrome 限定） | 每 2 秒更新 |
| **載入狀態** | `projectStore.loadingState` | 即時更新，顯示：● Ready（綠）/ ● Loading（黃）/ ● Error（紅） |

### 5.2 樣式規範

```typescript
const statusBarClass = cn(
  'flex items-center gap-4 px-4 h-8',
  'bg-[var(--color-bg-panel)]/80 backdrop-blur-sm',
  'border-t border-[var(--color-border-default)]',
  'text-xs text-[var(--color-text-muted)]',
  'select-none',
);
```

狀態指示燈顏色對應：

| 狀態 | 顏色 | CSS 變數 |
|------|------|----------|
| Ready | 綠色 | `var(--color-success)` |
| Loading | 黃色 | `var(--color-warning)` |
| Error | 紅色 | `var(--color-error)` |

---

## 6. React 元件架構

### 6.1 元件樹

```
App.tsx
├── ThemeProvider                        ← 主題管理（data-theme 屬性）
│   └── Layout                           ← CSS Grid 主佈局
│       ├── Toolbar                      ← 頂部工具列 (row 1, col 1-2)
│       │   ├── PanelToggles             ← 面板顯示/隱藏按鈕
│       │   ├── ProjectControls          ← 專案名稱 / 新增 / 開啟 / 儲存
│       │   ├── ViewControls             ← 解析度選擇 / 背景色
│       │   ├── ZoomControls             ← 縮放按鈕 / 百分比 / 適應視窗
│       │   └── PlayControls             ← 腳本播放 / 暫停 / 停止
│       │
│       ├── LeftPanel                    ← 左側面板 (row 2, col 1)
│       │   ├── PanelHeader              ← 「圖層」標題
│       │   ├── LayerActions             ← 新增 / 刪除 / 群組按鈕
│       │   └── LayerTree                ← 遞迴圖層樹
│       │       └── LayerItem (recursive)← 單一圖層項目
│       │
│       ├── CanvasArea                   ← 中央區域 (row 2, col 2)
│       │   ├── PixiCanvas               ← PixiJS Canvas 容器（ref → PixiApp）
│       │   └── CanvasOverlay            ← Canvas 上層覆蓋（縮放/平移指示器）
│       │
│       ├── RightPanel                   ← 右側面板 (row 1-3, col 3)
│       │   ├── ResizeHandle             ← 拖拽手柄（左側 4px）
│       │   ├── TabBar                   ← Tab 切換列
│       │   └── TabContent               ← 依選中 Tab 渲染
│       │       ├── PropertiesTab        ← 屬性 Tab 內容
│       │       │   ├── TransformSection ← 變換區段
│       │       │   ├── DisplaySection   ← 顯示區段
│       │       │   └── ClipSection      ← 裁切區段
│       │       ├── SpineTab             ← Spine 控制 Tab
│       │       │   ├── AnimationList    ← 動畫列表
│       │       │   ├── TrackMixer       ← 軌道混合器
│       │       │   └── SpineControls    ← 皮膚 / 時間軸 / 速度
│       │       ├── ReelTab              ← 轉輪控制 Tab
│       │       │   ├── ReelActions      ← 開始 / 停止 / 快停
│       │       │   ├── ResultInput      ← 手動結果輸入
│       │       │   └── ReelParams       ← 運動參數
│       │       └── ScriptTab            ← 腳本 Tab
│       │           ├── ScriptList       ← 腳本列表
│       │           ├── ScriptControls   ← 載入 / 執行 / 暫停 / 停止
│       │           └── ScriptProgress   ← 執行進度指示
│       │
│       └── StatusBar                    ← 底部狀態列 (row 3, col 2)
│
└── ToastContainer                       ← Toast 通知（fixed 定位，z-[9999]）
```

### 6.2 元件檔案對應

| 元件 | 檔案路徑 |
|------|----------|
| `Layout` | `src/ui/layout/Layout.tsx` |
| `Toolbar` | `src/ui/toolbar/Toolbar.tsx` |
| `ProjectControls` | `src/ui/toolbar/ProjectControls.tsx` |
| `ViewControls` | `src/ui/toolbar/ViewControls.tsx` |
| `ZoomControls` | `src/ui/toolbar/ZoomControls.tsx` |
| `PlayControls` | `src/ui/toolbar/PlayControls.tsx` |
| `LeftPanel` | `src/ui/panels/LeftPanel.tsx` |
| `LayerTree` | `src/ui/panels/LayerTree.tsx` |
| `LayerItem` | `src/ui/panels/LayerItem.tsx` |
| `LayerActions` | `src/ui/panels/LayerActions.tsx` |
| `RightPanel` | `src/ui/panels/RightPanel.tsx` |
| `PropertiesTab` | `src/ui/panels/PropertiesTab.tsx` |
| `SpineTab` | `src/ui/panels/SpineTab.tsx` |
| `ReelTab` | `src/ui/panels/ReelTab.tsx` |
| `ScriptTab` | `src/ui/panels/ScriptTab.tsx` |
| `CanvasArea` | `src/ui/layout/CanvasArea.tsx` |
| `StatusBar` | `src/ui/layout/StatusBar.tsx` |
| `ToastContainer` | `src/ui/components/ToastContainer.tsx` |

### 6.3 狀態管理映射

元件與 Zustand Store 的對應關係：

```
┌─────────────────────┐     ┌─────────────────────────┐
│   UI 元件            │     │   Zustand Store          │
├─────────────────────┤     ├─────────────────────────┤
│ LayerTree           │ ──→ │ layerStore.layers        │
│ LayerItem           │ ──→ │ layerStore.selectedId    │
│ PropertiesTab       │ ──→ │ layerStore.selectedLayer │
│ SpineTab            │ ──→ │ spineStore               │
│ ReelTab             │ ──→ │ reelStore                │
│ ScriptTab           │ ──→ │ scriptStore              │
│ ZoomControls        │ ──→ │ previewStore.zoom        │
│ StatusBar           │ ──→ │ previewStore (多欄位)     │
│ Toolbar (面板切換)   │ ──→ │ uiStore.panelStates     │
│ ProjectControls     │ ──→ │ projectStore             │
└─────────────────────┘     └─────────────────────────┘
```

---

## 7. 響應式行為

### 7.1 最小視窗尺寸

| 約束 | 值 |
|------|------|
| 最小寬度 | 1024px |
| 最小高度 | 768px |

當視窗尺寸小於最小值時，不進行特殊處理（本工具為桌面端應用，不考慮行動裝置適配）。

### 7.2 Canvas 自動縮放

Canvas 區域使用 `ResizeObserver` 監聽容器尺寸變化，自動調整 PixiJS renderer 的大小：

```typescript
useEffect(() => {
  const container = canvasRef.current;
  if (!container) return;

  const observer = new ResizeObserver((entries) => {
    const { width, height } = entries[0].contentRect;
    pixiApp.resize(width, height);
  });

  observer.observe(container);
  return () => observer.disconnect();
}, []);
```

### 7.3 面板收合策略

面板收合可為 Canvas 騰出更多空間。收合狀態儲存於 `uiStore`：

```typescript
interface UiState {
  leftPanelCollapsed: boolean;
  rightPanelCollapsed: boolean;
  rightPanelWidth: number;
}
```

| 場景 | 可用 Canvas 寬度（以 1920px 為例） |
|------|------|
| 全部展開（左 240 + 右 280） | 1400px |
| 左側收合 | 1640px |
| 右側收合 | 1680px |
| 全部收合 | 1920px |

### 7.4 右側面板拖拽

右側面板使用 `useResizablePanel` hook（定義於 JR UI Design System）：

```typescript
const { size: rightWidth, handleMouseDown } = useResizablePanel({
  direction: 'horizontal',
  initialSize: 280,
  minSize: 280,
  maxSize: 600,
});
```

拖拽手柄位於面板左側邊緣，寬度 4px，hover 時顯示 `col-resize` 游標並以 accent 色高亮：

```typescript
<div
  onMouseDown={handleMouseDown}
  className={cn(
    'absolute left-0 top-0 bottom-0 w-1 cursor-col-resize z-10',
    'hover:bg-[var(--color-accent)]/30',
    'transition-colors duration-150',
  )}
/>
```

---

## 8. 鍵盤快捷鍵

所有快捷鍵透過全域 `useEffect` 監聽 `keydown` 事件。當焦點在 input/textarea 元素上時，不攔截快捷鍵（避免干擾文字輸入）。

| 快捷鍵 | 功能 | 對應 Store Action |
|--------|------|-------------------|
| `Delete` | 刪除選取的圖層 | `layerStore.deleteLayer()` |
| `Ctrl + Z` | 復原（未來功能） | `historyStore.undo()` |
| `Ctrl + Y` | 重做（未來功能） | `historyStore.redo()` |
| `Space + 拖拽` | 平移 Canvas | `previewStore.setPan()` |
| `Ctrl + 滾輪` | 縮放 Canvas | `previewStore.setZoom()` |
| `Ctrl + S` | 匯出專案 | `projectStore.exportProject()` |
| `Ctrl + O` | 匯入專案 | `projectStore.importProject()` |
| `F11` | 全螢幕 Canvas | `document.fullscreenElement` toggle |
| `Ctrl + Shift + L` | 切換左側面板 | `uiStore.toggleLeftPanel()` |
| `Ctrl + Shift + R` | 切換右側面板 | `uiStore.toggleRightPanel()` |

### 8.1 快捷鍵實作範例

```typescript
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    const target = e.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') return;

    if (e.key === 'Delete') {
      e.preventDefault();
      layerStore.getState().deleteSelectedLayer();
    }

    if (e.ctrlKey && e.key === 's') {
      e.preventDefault();
      projectStore.getState().exportProject();
    }
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, []);
```

---

## 9. 無障礙設計（Accessibility）

雖然目標使用者為美術人員，仍遵循基本無障礙設計原則，確保介面具備良好的可操作性。

### 9.1 焦點管理

| 規則 | 實作方式 |
|------|----------|
| 所有互動元素皆有 `:focus-visible` 外框 | `focus-visible:ring-2 focus-visible:ring-[var(--color-accent)]/50` |
| Tab 鍵可循序切換焦點 | 確保所有按鈕、輸入框、Tab 列皆可 Tab 聚焦 |
| Escape 鍵關閉彈窗/下拉 | 所有 Dropdown / Dialog 監聽 `Escape` 事件 |

### 9.2 ARIA 標記

| 元素 | ARIA 屬性 |
|------|-----------|
| Icon-only 按鈕 | `aria-label`（如 `aria-label="放大"`） |
| Tab 列 | `role="tablist"` + 每個 Tab 設定 `role="tab"` + `aria-selected` |
| Tab 面板 | `role="tabpanel"` + `aria-labelledby` 指向對應 Tab |
| Slider | `role="slider"` + `aria-valuemin` / `aria-valuemax` / `aria-valuenow` / `aria-label` |
| Toast 通知 | `role="alert"` + `aria-live="polite"` |
| 圖層樹 | `role="tree"` + 每個項目 `role="treeitem"` + `aria-expanded`（群組） |

### 9.3 減弱動畫

尊重使用者的系統偏好設定。當 `prefers-reduced-motion: reduce` 時：

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

所有 CSS transition 與 animation 將被降至接近零，包括：
- 按鈕 `active:scale-90` 的過渡
- 面板收合/展開的動畫
- Tab icon 的呼吸光暈
- Toast 的滑入動畫

### 9.4 對比度

所有文字與背景的對比度符合 WCAG AA 標準：

| 元素 | 前景色 | 背景色 | 對比度 |
|------|--------|--------|--------|
| 主要文字 | `#f8fafc` | `#0f172a` | 15.4:1 ✅ |
| 次要文字 | `#94a3b8` | `#0f172a` | 5.6:1 ✅ |
| 靜默文字 | `#475569` | `#0f172a` | 2.7:1 ⚠️ 僅用於裝飾/標籤 |
| Accent | `#3b82f6` | `#1e293b` | 4.1:1 ✅（UI 元素） |

---

## 10. 元件設計規範

所有 UI 元件皆遵循 **JR UI Design System** 規範。以下為各元件類型的設計準則。

### 10.1 按鈕（Button）

| 使用場景 | 變體 | 說明 |
|----------|------|------|
| 一般操作 | **Ghost**（預設） | 大部分工具列與面板按鈕皆使用 Ghost |
| 主要行動 | **Solid** | 僅限「執行」、「開始」等主要 CTA 按鈕 |
| 工具列圖示 | **Icon** | 正方形，無文字，必須設定 `aria-label` |
| 切換狀態 | Ghost + `active` | 啟用時顯示漸層背景與 glow shadow |

按鈕尺寸規範：

| 位置 | 尺寸 | 高度 |
|------|------|------|
| 頂部工具列 | `sm` | 28px |
| 面板內按鈕 | `sm` | 28px |
| 彈窗主按鈕 | `md` | 36px |

### 10.2 Glass Panel

所有面板（左側、右側、頂部工具列）皆使用 Glass Panel 風格：

```typescript
const glassPanelClass = cn(
  'bg-[var(--color-bg-panel)] backdrop-blur-lg',
  'border border-[var(--color-border-default)]',
  'shadow-[var(--shadow-glass)]',
);
```

### 10.3 Slider

| 使用場景 | 變體 | `--slider-color` |
|----------|------|-------------------|
| 透明度滑桿 | `default` | `var(--color-accent)` 藍色 |
| 軌道 Alpha | `compact` | `var(--color-accent-secondary)` 紫色 |
| 時間軸 scrubber | `timeline` | `var(--color-accent)` 藍色 |
| 速度控制 | `default` | `var(--color-warning)` 琥珀色 |

### 10.4 NumberInput

- 使用 `NumberInput` 元件（定義於 JR UI Design System）
- 對齊方式：`text-right`，使用 `tabular-nums` 等寬數字
- Hover 時顯示上下箭頭微調按鈕
- Focus 時使用 accent 色邊框與 ring

### 10.5 區段標題（Section Label）

面板內各區段的標題使用統一樣式：

```typescript
const sectionLabelClass = cn(
  'text-xs font-medium uppercase tracking-wider',
  'text-[var(--color-text-muted)]',
  'px-2 py-1.5',
  'border-b border-[var(--color-border-default)]',
);
```

### 10.6 捲軸（Scrollbar）

所有面板的可捲動區域使用 `custom-scrollbar` class：

- 寬度：6px
- 軌道：透明
- 滑塊：`rgba(255,255,255,0.1)`，hover 時 `rgba(255,255,255,0.2)`
- 圓角：3px

```typescript
<div className="flex-1 overflow-y-auto custom-scrollbar">
  {/* 可捲動內容 */}
</div>
```

### 10.7 分隔線（Divider）

區段之間使用細線分隔：

```typescript
const dividerClass = 'border-t border-[var(--color-border-default)]';
```

---

> **下一份文件**：`10-scripting-engine.md` — 腳本引擎設計規格
