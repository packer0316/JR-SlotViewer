# Slot Previewer — 開發路線圖

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管、專案管理

---

## 1. 開發路線圖

```
Week  1    2    3    4    5    6    7    8    9   10   11   12
      ├────┤    │    │    │    │    │    │    │    │    │    │
      │ P0 │    │    │    │    │    │    │    │    │    │    │
      │基礎│    │    │    │    │    │    │    │    │    │    │
      │建設│    │    │    │    │    │    │    │    │    │    │
      ├────┼────┼────┤    │    │    │    │    │    │    │    │
           │  P1 渲染核心  │    │    │    │    │    │    │    │
           ├────┼────┼────┼────┤    │    │    │    │    │    │
                          │ P2 Spine│    │    │    │    │    │
                          │  整合   │    │    │    │    │    │
                          ├────┼────┼────┼────┤    │    │    │
                                    │ P3 轉輪系統  │    │    │
                                    ├────┼────┼────┼────┤    │
                                              │P4 腳本引擎│  │
                                              ├────┼────┼────┤
                                                   │P5 │    │
                                                   │I/O│    │
                                                   ├───┼────┤
                                                        │ P6│
                                                        │打磨│
                                                        ├───┤
```

---

### Phase 0：基礎建設 (Week 1)

> **目標**：搭建專案骨架，確保開發環境、核心依賴與基本佈局就位。

| 任務 | 說明 | 產出 |
|------|------|------|
| 專案初始化 | Vite + React 18 + TypeScript 5 腳手架 | `package.json`、`tsconfig.json`、`vite.config.ts` |
| PixiJS 整合 | PixiJS 8 + `@esotericsoftware/spine-pixi` 安裝與驗證 | Canvas 可渲染空白場景 |
| UI 框架 | Tailwind CSS + shadcn/ui 設定 | 基礎樣式系統可用 |
| JR UI Design System | 整合 glassmorphism 面板、主題系統 | 統一設計語言 |
| Store 骨架 | Zustand 建立 7 個 slice 的空殼 | `layerStore`、`reelStore`、`spineStore`、`scriptStore`、`projectStore`、`uiStore`、`canvasStore` |
| TypeScript 路徑別名 | 設定 `@pixi`、`@store`、`@ui` 等 | `tsconfig.paths.json` |
| Canvas 掛載 | Canvas mount + ResizeObserver 自動調整尺寸 | Canvas 隨視窗調整大小 |
| 三欄佈局 | 基本 Layout：左側面板 / 中央 Canvas / 右側面板 | Glass panels 可見 |

**驗收標準**：Canvas 渲染空白場景，三欄佈局可見，所有 Store slice 可存取。

---

### Phase 1：渲染核心 (Week 2–3)

> **目標**：完成 PixiJS 渲染管線與圖層管理系統。

| 任務 | 說明 | 產出 |
|------|------|------|
| PixiApp 生命週期 | init / destroy / resize 管理 | `PixiApp.ts` |
| LayerManager | Container 樹 + Zustand Store 雙向同步 | `LayerManager.ts` |
| ClipManager | rect Graphics Mask 裁切管理 | `ClipManager.ts` |
| Sprite 圖層 | 基本 Sprite 載入與顯示 | Sprite Layer 支援 |
| 圖層面板 UI | 樹狀檢視、可見性、鎖定切換 | `LayerPanel.tsx` |
| 屬性面板 UI | Transform、Alpha、Blend Mode 調整 | `PropertyPanel.tsx` |
| 狀態列 | FPS + DrawCall 即時顯示 | `StatusBar.tsx` |
| Canvas 視窗控制 | Zoom / Pan 操作 | 滾輪縮放、拖拽平移 |

**驗收標準**：可新增 / 刪除 / 重排圖層、載入 Sprite、Clipping Mask 功能正常。

---

### Phase 2：Spine 整合 (Week 4–5)

> **目標**：支援 Spine 動畫載入、預覽與控制。

| 任務 | 說明 | 產出 |
|------|------|------|
| SpineLoader | 拖拽 + URL 兩種載入方式 | `SpineLoader.ts` |
| SpineActor API | play / mix / track / skin 控制介面 | `SpineActor.ts` |
| Web Worker 解析 | `.skel` 檔案在 Worker 中解析，避免阻塞主執行緒 | `skel-parser.worker.ts` |
| Spine 控制面板 | 動畫列表、速度調整、時間軸拖曳 | `SpineControlPanel.tsx` |
| Clipping Attachment | 處理 Spine 內建的 Clipping Attachment | 正確裁切顯示 |
| Layer ↔ Spine | 圖層系統與 Spine 整合 | Spine 可作為圖層管理 |

**驗收標準**：拖拽載入 Spine 檔案，可播放 / 切換動畫、調整速度、切換 Skin。

---

### Phase 3：轉輪系統 (Week 6–7)

> **目標**：實現完整的轉輪模擬系統，支援可變列數。

| 任務 | 說明 | 產出 |
|------|------|------|
| ReelEngine | Spin / Stop / Bounce / QuickStop 狀態機 | `ReelEngine.ts` |
| ReelConfig | JSON 結構定義與驗證 | `ReelConfig.ts`、`ReelConfigValidator.ts` |
| SymbolPool | Object Pool 管理 Symbol 節點 | `SymbolPool.ts` |
| Reel Clipping | 轉輪可視區域裁切 | Mask 正確遮罩 |
| ReelGridEditor | 可變列數編輯器（如 `[3,4,5,4,3]` Megaways） | `ReelGridEditor.tsx` |
| Grid 配置 | Symbol 尺寸、間距、對齊方式 | 配置面板 |
| 轉輪控制面板 | Spin / Stop 按鈕、速度調整 | `ReelControlPanel.tsx` |

**驗收標準**：5×3 盤面模擬 Spin / Stop，可變列數功能正常。

---

### Phase 4：腳本引擎 (Week 8–9)

> **目標**：完成腳本解析、驗證與執行引擎。

| 任務 | 說明 | 產出 |
|------|------|------|
| ScriptRunner | 解析、驗證、執行 JSON 腳本 | `ScriptRunner.ts` |
| 13 種內建 Action | spin / stop / playSpine / wait / setLayer 等 | `actions/*.ts` |
| ActionRegistry | Plugin 架構，支援自訂 Action | `ActionRegistry.ts` |
| EventBus | 全域事件系統 | `EventBus.ts` |
| 腳本面板 UI | 載入、執行、進度顯示 | `ScriptPanel.tsx` |
| 範例腳本 | 預設腳本範例，方便使用者參考 | `examples/*.json` |

**驗收標準**：載入 JSON 腳本並執行完整表演序列，支援暫停 / 恢復 / 中止。

---

### Phase 5：專案 I/O (Week 10)

> **目標**：實現專案檔案匯出與匯入功能。

| 任務 | 說明 | 產出 |
|------|------|------|
| ProjectExporter | 使用 JSZip 將專案狀態打包為 `.zip` | `ProjectExporter.ts` |
| ProjectImporter | 解析 `.zip` 並還原完整狀態 | `ProjectImporter.ts` |
| Schema 版本管理 | 版本號 + 遷移框架，確保向下相容 | `SchemaMigration.ts` |
| Toolbar | New / Open / Save 工具列按鈕 | `Toolbar.tsx` |
| 未儲存變更警告 | Dirty flag 偵測 + 離開前確認 | `useUnsavedWarning.ts` |

**驗收標準**：匯出 `.zip` → 重新匯入後，所有狀態與資源完整還原。

---

### Phase 6：打磨與優化 (Week 11–12)

> **目標**：效能優化、品質提升與文件完善。

| 任務 | 說明 | 產出 |
|------|------|------|
| CacheAsTexture | 靜態圖層快取為紋理，減少 DrawCall | 效能提升 |
| Atlas Batching | 合批渲染優化 | DrawCall 降低 |
| 記憶體分析 | 使用 DevTools 識別記憶體洩漏 | 記憶體穩定 |
| 鍵盤快捷鍵 | 常用操作的快捷鍵支援 | 操作效率提升 |
| 無障礙改進 | ARIA 標籤、焦點管理 | 可及性改善 |
| 錯誤處理 | 全域 Error Boundary + Toast 通知系統 | 使用者體驗提升 |
| 測試撰寫 | Unit + Integration + E2E 測試套件 | 品質保障 |
| 文件定稿 | 所有技術文件最終審查 | 完整文件 |

**驗收標準**：效能達標、測試通過、文件完整。

---

## 2. 里程碑與交付物

| 里程碑 | 週次 | 交付物 | 驗收標準 |
|--------|------|--------|----------|
| **M0** | Week 1 | 專案骨架 | Canvas 渲染空白場景，三欄佈局可見 |
| **M1** | Week 3 | 渲染核心 | 圖層 CRUD、基本 Sprite、Clipping 功能正常 |
| **M2** | Week 5 | Spine 整合 | 拖拽載入 Spine，播放 / 切換動畫 |
| **M3** | Week 7 | 轉輪系統 | 5×3 盤面模擬 Spin / Stop，可變列數 |
| **M4** | Week 9 | 腳本引擎 | 載入 JSON 腳本並執行完整表演序列 |
| **M5** | Week 10 | 專案 I/O | 匯出 / 匯入 `.zip` 專案檔 |
| **M6** | Week 12 | 正式版 | 效能達標，測試通過，文件完整 |

### 里程碑依賴關係

```
M0（基礎建設）
 │
 ├──▶ M1（渲染核心）
 │     │
 │     ├──▶ M2（Spine 整合）
 │     │
 │     └──▶ M3（轉輪系統）
 │           │
 │           └──▶ M4（腳本引擎）
 │
 └──────────────▶ M5（專案 I/O）※ 可與 M3/M4 並行
                   │
                   └──▶ M6（正式版）
```

---

## 3. 效能指標驗收

| 指標 | 目標 | 測量條件 | 工具 |
|------|------|----------|------|
| **FPS** | ≥ 60 FPS | 5 reels × 3 rows + 2 Spine 同時運行 | PixiJS Ticker 統計 |
| **DrawCall** | < 10 DC/frame | 標準場景（5×3 盤面 + Spine） | PixiJS DevTools |
| **JS Heap** | < 200 MB | 載入完整專案後穩定值 | Chrome DevTools Memory |
| **初始載入** | < 3 秒 | 首次開啟至 Canvas 可見 | Lighthouse / 手動計時 |
| **Spine 切換** | < 16 ms | 切換動畫的回應時間 | Performance API |

### 效能監控方案

```
┌─────────────────────────────────────────────────────────┐
│  效能監控                                                │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  FPS 計數器   │  │  DrawCall    │  │  Memory      │   │
│  │  StatusBar   │  │  StatusBar   │  │  DevTools    │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                  │           │
│         ▼                 ▼                  ▼           │
│  ┌──────────────────────────────────────────────────┐   │
│  │           開發者控制台即時輸出                      │   │
│  │  • FPS 低於 55 時顯示警告                          │   │
│  │  • DrawCall 超過 15 時顯示警告                     │   │
│  │  • Heap 超過 180 MB 時顯示警告                     │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. 風險與緩解

| 風險 | 機率 | 影響 | 緩解措施 |
|------|------|------|----------|
| **spine-pixi 與 PixiJS 8 相容性** | 🟡 Medium | 🔴 High | 於 Phase 0 即進行概念驗證（PoC），鎖定相容版本組合 |
| **WebGPU 瀏覽器支援不足** | 🟢 Low | 🟡 Medium | 預設使用 WebGL2 renderer，WebGPU 作為可選升級路徑 |
| **大量 Spine 同時渲染效能** | 🟡 Medium | 🔴 High | 實作 CacheAsTexture + Atlas Batching，並設定同時載入上限 |
| **ZIP 大檔案處理效能** | 🟢 Low | 🟡 Medium | JSZip 運算放入 Web Worker + streaming 處理 |
| **Zustand Store 過度渲染** | 🟡 Medium | 🟡 Medium | 使用 selector 精準訂閱，搭配 `shallow` 比較器 |
| **Web Worker 通訊瓶頸** | 🟢 Low | 🟡 Medium | 使用 Transferable Objects 減少序列化開銷 |
| **第三方依賴 Breaking Change** | 🟡 Medium | 🟡 Medium | lockfile 鎖版，定期但謹慎升級 |

### 風險矩陣

```
        │  Low Impact    Medium Impact   High Impact
────────┼──────────────────────────────────────────
High    │                                          
Prob.   │                                          
────────┼──────────────────────────────────────────
Medium  │               • Zustand 過度   • spine-pixi 相容
Prob.   │                 渲染           • Spine 效能
        │               • 依賴 Breaking                
────────┼──────────────────────────────────────────
Low     │               • WebGPU 支援    
Prob.   │               • ZIP 效能       
        │               • Worker 瓶頸    
────────┼──────────────────────────────────────────
```

---

## 5. 未來展望 (Post v1.0)

以下為 v1.0 後的潛在功能方向，依優先順序排列：

| 優先級 | 功能 | 說明 | 預估工作量 |
|--------|------|------|-----------|
| **P1** | Electron 桌面應用打包 | 脫離瀏覽器限制，支援本地檔案系統存取 | 2–3 週 |
| **P2** | Undo / Redo 歷史紀錄 | 支援操作撤銷與重做，提升編輯體驗 | 2 週 |
| **P3** | 視覺化腳本 Timeline 編輯器 | 以時間軸 UI 編輯腳本，取代手寫 JSON | 4–6 週 |
| **P3** | 條件分支與迴圈腳本 | `if / else`、`loop`、`switch` 等控制流 | 2–3 週 |
| **P4** | 多人專案分享（雲端同步） | 即時協作或雲端專案存取 | 6–8 週 |
| **P4** | 自訂盤面形狀 | 菱形、六角形等非矩形盤面 | 3–4 週 |
| **P5** | 音效系統整合 | 為表演腳本添加音效觸發 | 2–3 週 |

### 功能演進路線

```
v1.0（MVP）                    v1.x（增強）                v2.0（進階）
┌──────────────┐              ┌──────────────┐            ┌──────────────┐
│ • 圖層管理    │              │ • Electron   │            │ • 雲端同步    │
│ • Spine 預覽  │    ──▶       │ • Undo/Redo  │    ──▶     │ • Timeline   │
│ • 轉輪模擬    │              │ • 條件腳本    │            │ • 自訂盤面    │
│ • 腳本執行    │              │ • 音效系統    │            │ • 多人協作    │
│ • 專案 I/O   │              │              │            │              │
└──────────────┘              └──────────────┘            └──────────────┘
```

---

## 附錄：技術棧版本鎖定

| 依賴 | 版本 | 說明 |
|------|------|------|
| React | ^18.3.x | 穩定版，不升級至 React 19（待觀察） |
| PixiJS | ^8.x | 主要渲染引擎 |
| spine-pixi | 與 PixiJS 8 相容版本 | 需於 Phase 0 驗證 |
| Zustand | ^5.x | 輕量狀態管理 |
| TypeScript | ^5.5.x | 嚴格模式 |
| Vite | ^6.x | 開發伺服器與建置 |
| Tailwind CSS | ^4.x | Utility-first CSS |
| Vitest | ^3.x | 單元 + 整合測試 |
| Playwright | ^1.49.x | E2E 測試 |
| JSZip | ^3.x | ZIP 檔案處理 |
