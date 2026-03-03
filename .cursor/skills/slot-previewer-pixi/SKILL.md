---
name: slot-previewer-pixi
description: PixiJS rendering rules for Slot Previewer. Covers rendering pipeline, layer system, clipping, Spine integration, reel engine, object pooling, and performance targets. Use when working with PixiJS code, Canvas rendering, layers, clipping, Spine animations, reel mechanics, or performance optimization.
---

# Slot Previewer — PixiJS 渲染與效能規則

## 效能目標（不可妥協）

| 指標 | 目標 |
|------|------|
| FPS | 60（5 輪 × 3+ 列 + 2 Spine 同時） |
| DrawCall | < 10 / 幀 |
| JS Heap | < 200 MB |
| Spine 切換 | < 16ms |
| 初始載入 | < 3 秒 |

## 渲染核心規則

- React **永遠不參與** Ticker 迴圈 — 渲染完全由 PixiJS 管理
- PixiJS 7 初始化：WebGL2 renderer，`maxFPS: 60`（無 WebGPU）
- 畫布縮放用 `ResizeObserver`，不用 `window.onresize`
- GSAP **僅用於腳本層補間**，不介入渲染層

## 圖層系統 (LayerManager)

- 每個 `LayerNode` 對應一個 PixiJS `Container`
- Store 變更 → `syncFromStore()` → diff 更新 Container 樹
- 支援屬性：`visible`, `alpha`, `blendMode`, `zIndex`, `position`, `scale`, `rotation`
- 圖層類型以 `type` 聯合型別區分：`'spine' | 'sprite' | 'reel' | 'clip' | 'group'`

## Clipping (ClipManager)

| 模式 | 用途 | 效能 |
|------|------|------|
| Graphics Mask（硬邊）| 轉輪視窗 | 最佳 |
| Sprite Mask（柔邊）| 漸層遮罩 | 中 |
| Stencil Mask | 複雜聯合遮罩 | 較差 |

- 轉輪視窗 **預設** 使用 Graphics Mask
- 每個 Mask 有唯一 `id`，可動態更新

## Spine 整合

- 使用 `@pixi-spine/all-3.8`（社群維護，支援 Spine 3.8 + PixiJS 7）
- `.skel` 解析放 **Web Worker**（不阻塞主執行緒）
- Worker 崩潰 → 降級至主執行緒解析
- SkeletonData 快取共享，SpineActor 實例獨立
- SpineActor 封裝底層 Runtime，外部只用高階 API

## 轉輪系統 (ReelEngine)

- 運動由 **PixiJS Ticker** 驅動（非 requestAnimationFrame）
- 支援每輪可變列數：`rowsPerReel: number[]`（如 `[3,4,5,4,3]`）
- 完整網格配置：符號尺寸、間距、對齊方式
- Symbol **必須** 透過 ObjectPool 管理 — 動畫中禁止 `new Sprite()`

## 效能策略

- **cacheAsBitmap**：靜態圖層快取為 RenderTexture，屬性變更時失效（PixiJS 7 API）
- **Spine 合批**：同材質骨骼合併 DrawCall（避免不同 blendMode 打斷批次）
- **符號圖集**：所有轉輪符號打包成單一 Atlas
- **ObjectPool**：預分配 = rows × reels + buffer
- **React 隔離**：Zustand selector 細粒度訂閱，避免無謂 re-render
