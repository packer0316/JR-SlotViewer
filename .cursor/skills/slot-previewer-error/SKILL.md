---
name: slot-previewer-error
description: Error handling and UX feedback rules for Slot Previewer. Covers error classification, graceful degradation, Toast notifications, Error Boundary, and recovery strategies. Use when implementing error handling, try-catch blocks, user feedback, or recovery logic.
---

# Slot Previewer — 錯誤處理規則

## 核心原則

- **絕不崩潰**：任何錯誤都不應導致白屏
- **優雅降級**：功能失敗時降級並繼續運作
- 面向使用者的訊息 **必須繁體中文**，說明「發生了什麼」+「可以做什麼」
- 所有 `async` 函式必須有錯誤處理

## 錯誤分級

| 級別 | 處理 | 範例 |
|------|------|------|
| Low（使用者錯誤）| Toast hint | 無效輸入、重複名稱 |
| Medium（載入/驗證）| Toast warning，skip 資源 | 檔案損壞、格式錯誤 |
| High（運行時）| catch + 恢復 | 渲染錯誤、action 失敗 |
| Critical（系統）| 錯誤對話框 | WebGL context lost、OOM |

## 關鍵規則

- Ticker 迴圈 **必須 try-catch** — 渲染迴圈永不中斷
- Spine Worker 崩潰 → 降級至主執行緒解析
- 腳本循環 `callScript` → 偵測並中止（呼叫棧追蹤）
- ZIP 匯入缺失資源 → 匯入可用部分 + 列出缺失清單
- WebGL context lost → 嘗試恢復 + Toast 提示

## Toast 回饋

- 4 種：`info`(藍)、`success`(綠)、`warning`(黃)、`error`(紅)
- info/success：5 秒自動關閉
- warning/error：手動關閉
- 遵循 JR UI Design System Toast 規範

## React Error Boundary

- 頂層 `AppErrorBoundary` 捕獲未處理的 React 錯誤
- 顯示友善的重試頁面，不顯示 stack trace 給使用者
- console.error 記錄完整錯誤供開發者除錯

## 恢復策略

| 場景 | 策略 |
|------|------|
| WebGL context lost | 自動恢復 + 重建場景 |
| 元件崩潰 | ErrorBoundary + retry |
| CDN 資源失敗 | 指數退避重試 |
| 缺失貼圖 | 佔位圖替代 |
