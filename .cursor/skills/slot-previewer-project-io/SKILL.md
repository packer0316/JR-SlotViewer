---
name: slot-previewer-project-io
description: Project I/O rules for Slot Previewer. Covers ZIP project file structure, export/import flow, schema versioning, asset path management, and Blob URL lifecycle. Use when working with project save/load, ZIP generation, schema migration, or asset management.
---

# Slot Previewer — 專案 I/O 規則

## 核心規則

- `.zip` 是 **唯一持久化格式**（無 LocalStorage、無後端）
- 純客戶端處理，使用 JSZip
- Schema **必須版本化**（SemVer），匯入時最先讀 `manifest.json`

## ZIP 結構

```
project.zip/
├── manifest.json       # schemaVersion, name, dates, assetCount
├── state.json          # layers, clips, reelConfigs, scripts, preview
├── assets/
│   ├── spine/{name}/   # .skel + .atlas + .png
│   ├── sprites/        # .png
│   └── scripts/        # .json
└── thumbnail.png       # 可選
```

## 匯出流程

1. 收集所有 Store 狀態
2. 資源路徑轉為 zip 內相對路徑
3. JSZip 打包 → Blob → 瀏覽器下載
4. 缺失資源：warn + skip（不阻擋匯出）

## 匯入流程

1. 解析 zip → 讀 `manifest.json` → 驗證 schemaVersion
2. 版本 > 當前 → **拒絕載入**（提示更新應用）
3. 版本 < 當前 → **鏈式遷移**（v1→v2→v3）
4. 提取資源 → 建立 Blob URL → 灌入 Store → 重建場景

## Schema 遷移規則

- 每版本間有獨立 `Migration` 函式，鏈式執行
- Schema **只加不刪**：新欄位用 `?` 可選，確保向前相容
- 未知版本必須拒絕，不嘗試猜測

## 資源路徑生命週期

```
ZIP 相對路徑 →(匯入)→ Blob URL →(運行時使用)→(匯出)→ 轉回相對路徑
```

- Blob URL 在專案替換/卸載時 **必須** `revokeObjectURL()`
- 防止記憶體洩漏

## Dirty 標記

- Store 變更 → `dirty = true`
- 匯入/匯出前檢查 dirty，提醒未儲存
- `beforeunload` 攔截
