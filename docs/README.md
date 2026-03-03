# Slot Previewer — 架構文件索引

> **版本**：v1.0 | **最後更新**：2026/03/03 | **技術棧**：React 18 + PixiJS 8 + Spine Runtime + Zustand

本目錄包含 Slot Previewer 的完整架構設計文件。Slot Previewer 是一套基於瀏覽器的 Slot 遊戲動畫預覽工具，提供美術人員即時預覽 Spine 動畫、調整圖層結構、模擬轉輪行為，以及撰寫並執行表演腳本。

---

## 文件目錄

| # | 文件 | 說明 |
|---|------|------|
| 01 | [架構總覽](01-architecture-overview.md) | 系統總覽、設計原則、技術棧、系統架構圖、目錄結構、模組依賴、擴充性要點 |
| 02 | [模組設計](02-module-design.md) | 各模組職責、公開 API（TypeScript 簽名）、依賴關係、通訊方式、擴充指南 |
| 03 | [資料模型](03-data-models.md) | 所有 TypeScript 介面與型別定義：圖層、Clipping、轉輪、Spine、腳本、專案 I/O、Store Slice |
| 04 | [渲染管線](04-rendering-pipeline.md) | PixiJS Application 生命週期、場景圖結構、圖層系統、Clipping 系統、Blend Mode、CacheAsTexture、座標系統 |
| 05 | [轉輪系統](05-reel-system.md) | 盤面配置（可變列數）、盤面編輯器、ReelEngine 運動模型、Symbol 物件池、Reel Clipping |
| 06 | [Spine 整合](06-spine-integration.md) | SpineLoader 資源載入、SpineActor 動畫控制、Clipping Attachment、控制面板 UI、效能考量 |
| 07 | [腳本引擎](07-script-engine.md) | 腳本結構、13 種內建 Action、ActionHandler 架構、ScriptRunner 執行流程、EventBus 事件系統 |
| 08 | [狀態管理](08-state-management.md) | Zustand Store Slice 設計、Immer Middleware、Selector 最佳實踐、Manager 訂閱模式、持久化策略 |
| 09 | [UI 佈局](09-ui-layout.md) | 三欄式佈局、圖層面板、屬性面板、工具列、狀態列、React 元件架構、鍵盤快捷鍵、無障礙設計 |
| 10 | [專案 I/O](10-project-io.md) | ZIP 專案檔結構、匯出/匯入流程、Schema 版本遷移、資源路徑管理 |
| 11 | [效能策略](11-performance-strategy.md) | 效能目標、渲染/React/記憶體/載入效能策略、效能監控、已知瓶頸與對策 |
| 12 | [錯誤處理](12-error-handling.md) | 錯誤分類、各模組錯誤處理、使用者回饋機制、Error Boundary、恢復策略 |
| 13 | [測試策略](13-testing-strategy.md) | 測試金字塔、單元/整合/E2E 測試、測試命名規範、CI/CD 整合、目錄結構 |
| 14 | [開發路線圖](14-roadmap.md) | Phase 0–6（12 週）開發計畫、里程碑、效能驗收指標、風險評估、未來展望 |

---

## 建議閱讀順序

1. **快速了解**：先讀 [01 架構總覽](01-architecture-overview.md)，掌握全局
2. **深入模組**：根據開發任務讀對應文件（如要做轉輪 → 讀 05、如要做腳本 → 讀 07）
3. **開發前必讀**：[03 資料模型](03-data-models.md) + [08 狀態管理](08-state-management.md)
4. **上線前確認**：[11 效能策略](11-performance-strategy.md) + [12 錯誤處理](12-error-handling.md) + [13 測試策略](13-testing-strategy.md)

---

## 關鍵設計決策摘要

| 決策 | 選擇 | 原因 |
|------|------|------|
| 渲染引擎 | PixiJS 8 | WebGPU 支援、改進的批次渲染、Spine 官方整合 |
| 狀態管理 | Zustand + Immer | 輕量、支援細粒度訂閱、PixiJS Manager 可直接訂閱 |
| Spine Runtime | @esotericsoftware/spine-pixi | 官方 PixiJS 整合、Spine 4.x 完整功能 |
| 專案儲存 | .ZIP 檔案 | 打包美術資源 + 狀態、易於分享、無需後端 |
| 轉輪配置 | 可變列數 | 支援標準 5×3 及 Megaways 等各種盤面 |
| 腳本系統 | JSON + Action 註冊制 | 設計師可編輯、無需改程式碼、易於擴充 |
| UI 框架 | shadcn/ui + Tailwind | 元件品質高、客製性強、搭配 JR UI Design System |
| 部署方式 | 先靜態部署、未來 Electron | 降低初期複雜度、保留桌面應用選項 |

---

## 文件維護指南

- 修改架構或 API 時，同步更新對應文件
- 新增模組時，在本 README 中加入文件連結
- 版本號遵循語意化版本（SemVer）
- 所有文件以**繁體中文**撰寫
