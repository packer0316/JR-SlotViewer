# Slot Previewer — 專案 I/O 設計規格

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管

---

## 目錄

1. [專案 I/O 概覽](#1-專案-io-概覽)
2. [ZIP 檔案結構](#2-zip-檔案結構)
3. [manifest.json Schema](#3-manifestjson-schema)
4. [state.json Schema](#4-statejson-schema)
5. [匯出流程（ProjectExporter）](#5-匯出流程projectexporter)
6. [匯入流程（ProjectImporter）](#6-匯入流程projectimporter)
7. [Schema 版本遷移](#7-schema-版本遷移)
8. [資源路徑管理](#8-資源路徑管理)
9. [使用者流程](#9-使用者流程)
10. [效能考量](#10-效能考量)
11. [擴充性](#11-擴充性)

---

## 1. 專案 I/O 概覽

Slot Previewer 以 **`.zip` 專案檔** 作為唯一的持久化格式。所有專案資料——包含圖層結構、轉輪配置、腳本定義、預覽設定，以及 Spine 動畫、圖片等美術素材——皆封裝為單一 `.zip` 檔案，方便分享與備份。

### 設計原則

| 原則 | 說明 |
|------|------|
| **單檔封裝** | 一個 `.zip` 即是完整專案，不依賴外部檔案或目錄結構。 |
| **純客戶端** | 匯出 / 匯入完全在瀏覽器端完成，不需要後端伺服器。使用 [JSZip](https://stuk.github.io/jszip/) 進行 zip 生成與解析。 |
| **Schema 版本化** | `manifest.json` 包含 `schemaVersion`，確保向前相容。新版工具可透過 migration 讀取舊版專案檔。 |
| **分享友善** | 單人使用，但可將 `.zip` 分享給其他同事，對方直接匯入即可重現完整專案狀態。 |
| **資源內嵌** | 所有美術素材（Spine 骨骼動畫、靜態圖片、腳本檔案）皆存放在 zip 內，匯入時無須額外下載。 |

### 核心技術

| 用途 | 技術 |
|------|------|
| ZIP 生成 / 解析 | JSZip |
| 檔案下載觸發 | `URL.createObjectURL()` + `<a>` 元素 |
| 檔案讀取 | `FileReader` / `File` API |
| 拖拽匯入 | HTML5 Drag & Drop API |
| 資源暫存 | Blob URL（`URL.createObjectURL()`） |

### 相關原始碼

| 檔案 | 職責 |
|------|------|
| `src/io/ProjectExporter.ts` | `.zip` 匯出邏輯 |
| `src/io/ProjectImporter.ts` | `.zip` 匯入邏輯 |
| `src/io/ProjectSchema.ts` | 專案檔 JSON Schema 與 migration |
| `src/store/projectStore.ts` | 專案層級狀態（名稱、dirty、儲存時間） |

---

## 2. ZIP 檔案結構

每個 `.zip` 專案檔遵循以下固定目錄結構：

```
project.zip/
├── manifest.json               ← 專案清單（版本、名稱、日期）
├── state.json                  ← 完整專案狀態
│                               │   - layers[]
│                               │   - clips[]
│                               │   - reelConfigs[]
│                               │   - scripts[]
│                               │   - spineResources[]
│                               │   - spriteResources[]
│                               │   - preview settings
├── assets/
│   ├── spine/                  ← Spine 骨骼動畫資源
│   │   ├── hero/
│   │   │   ├── hero.skel           .skel 骨骼檔
│   │   │   ├── hero.atlas          .atlas 圖集
│   │   │   └── hero.png            貼圖
│   │   └── wild/
│   │       ├── wild.skel
│   │       ├── wild.atlas
│   │       └── wild.png
│   ├── sprites/                ← 靜態圖片資源
│   │   ├── background.png
│   │   └── frame.png
│   └── scripts/                ← 腳本檔案
│       ├── bigwin.json
│       └── freespin.json
└── thumbnail.png               ← 專案縮圖（可選，用於檔案預覽）
```

### 目錄規則

| 目錄 | 內容 | 命名慣例 |
|------|------|----------|
| `assets/spine/<name>/` | 每組 Spine 資源獨立子目錄 | 目錄名與骨骼檔同名 |
| `assets/sprites/` | 所有靜態圖片平鋪 | 保留原始檔名 |
| `assets/scripts/` | 腳本 JSON 檔案 | `<scriptName>.json` |
| 根目錄 | `manifest.json`、`state.json`、`thumbnail.png` | 固定命名 |

---

## 3. manifest.json Schema

`manifest.json` 是 zip 的根描述檔，記錄專案基本資訊與 Schema 版本號。匯入時**最先**讀取此檔案，以確認版本相容性。

### 3.1 型別定義

```typescript
interface ProjectManifest {
  /** Schema 版本號，遵循 SemVer 格式（如 "1.0.0"），用於版本遷移判斷 */
  schemaVersion: string;

  /** 專案名稱 */
  name: string;

  /** 專案描述（可選） */
  description?: string;

  /** 建立時間（ISO 8601 格式） */
  createdAt: string;

  /** 最後更新時間（ISO 8601 格式） */
  updatedAt: string;

  /** 建立者名稱（可選） */
  creator?: string;

  /** 各類資源數量統計 */
  assetCount: {
    spine: number;
    sprites: number;
    scripts: number;
  };
}
```

### 3.2 範例

```json
{
  "schemaVersion": "1.0.0",
  "name": "Dragon Fortune 主遊戲",
  "description": "Dragon Fortune Slot 的完整演出預覽",
  "createdAt": "2026-03-03T10:30:00.000Z",
  "updatedAt": "2026-03-03T14:22:15.000Z",
  "creator": "美術-小明",
  "assetCount": {
    "spine": 5,
    "sprites": 12,
    "scripts": 3
  }
}
```

### 3.3 欄位約束

| 欄位 | 型別 | 必填 | 約束 |
|------|------|------|------|
| `schemaVersion` | `string` | ✅ | SemVer 格式，如 `"1.0.0"` |
| `name` | `string` | ✅ | 長度 1–100 字元 |
| `description` | `string` | ❌ | 長度 ≤ 500 字元 |
| `createdAt` | `string` | ✅ | ISO 8601 |
| `updatedAt` | `string` | ✅ | ISO 8601，≥ `createdAt` |
| `creator` | `string` | ❌ | 長度 ≤ 50 字元 |
| `assetCount.spine` | `number` | ✅ | 非負整數 |
| `assetCount.sprites` | `number` | ✅ | 非負整數 |
| `assetCount.scripts` | `number` | ✅ | 非負整數 |

---

## 4. state.json Schema

`state.json` 儲存完整的應用程式狀態快照，是匯入後還原專案的核心資料來源。

### 4.1 型別定義

```typescript
interface ProjectState {
  /** 圖層樹（完整的 LayerNode 陣列） */
  layers: LayerNode[];

  /** 裁切遮罩定義 */
  clips: ClipMask[];

  /** 轉輪配置 */
  reelConfigs: ReelConfig[];

  /** 腳本定義 */
  scripts: ScriptDefinition[];

  /** Spine 資源參照（指向 assets/spine/* 內的檔案） */
  spineResources: SpineResourceRef[];

  /** 靜態圖片資源參照（指向 assets/sprites/* 內的檔案） */
  spriteResources: SpriteResourceRef[];

  /** 預覽設定 */
  preview: PreviewSettings;
}
```

### 4.2 資源參照型別

`state.json` 中的資源路徑皆為**相對於 zip 根目錄**的路徑，匯入時由 `ProjectImporter` 轉換為 Blob URL。

```typescript
interface SpineResourceRef {
  id: string;
  name: string;
  /** zip 內的相對路徑，如 "assets/spine/hero/hero.skel" */
  skelPath: string;
  /** zip 內的相對路徑，如 "assets/spine/hero/hero.atlas" */
  atlasPath: string;
  /** zip 內的相對路徑陣列 */
  pngPaths: string[];
  animations: string[];
  skins: string[];
}

interface SpriteResourceRef {
  id: string;
  name: string;
  /** zip 內的相對路徑，如 "assets/sprites/background.png" */
  path: string;
}
```

### 4.3 預覽設定型別

```typescript
interface PreviewSettings {
  /** 畫布寬度（px） */
  width: number;
  /** 畫布高度（px） */
  height: number;
  /** 背景色（CSS 色碼，如 "#1a1a2e"） */
  backgroundColor: string;
  /** 縮放倍率 */
  zoom: number;
  /** 水平平移偏移（px） */
  panX: number;
  /** 垂直平移偏移（px） */
  panY: number;
}
```

### 4.4 state.json 範例（精簡）

```json
{
  "layers": [
    {
      "id": "layer-001",
      "name": "背景",
      "type": "sprite",
      "visible": true,
      "locked": false,
      "alpha": 1,
      "blendMode": "normal",
      "zIndex": 0,
      "position": { "x": 0, "y": 0 },
      "scale": { "x": 1, "y": 1 },
      "rotation": 0,
      "spriteData": { "texturePath": "assets/sprites/background.png" }
    }
  ],
  "clips": [],
  "reelConfigs": [],
  "scripts": [],
  "spineResources": [
    {
      "id": "spine-001",
      "name": "Hero",
      "skelPath": "assets/spine/hero/hero.skel",
      "atlasPath": "assets/spine/hero/hero.atlas",
      "pngPaths": ["assets/spine/hero/hero.png"],
      "animations": ["idle", "win", "bigwin"],
      "skins": ["default"]
    }
  ],
  "spriteResources": [
    {
      "id": "sprite-001",
      "name": "背景",
      "path": "assets/sprites/background.png"
    }
  ],
  "preview": {
    "width": 1920,
    "height": 1080,
    "backgroundColor": "#1a1a2e",
    "zoom": 1,
    "panX": 0,
    "panY": 0
  }
}
```

---

## 5. 匯出流程（ProjectExporter）

### 5.1 流程圖

```
使用者點擊「匯出專案」
        │
        ▼
┌──────────────────┐
│ 1. 收集狀態       │  ← 從所有 Zustand Store 取得當前狀態
│    (Zustand)      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. 收集資源檔案   │  ← 蒐集已載入的 Spine 檔案、圖片、腳本
│    (Assets)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. 建構 manifest │  ← 生成 manifest.json（版本、名稱、統計）
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. 建構 state    │  ← 序列化狀態，將路徑轉換為 zip 內相對路徑
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. 建立 ZIP      │  ← JSZip 建立目錄結構
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. 寫入檔案      │  ← manifest.json、state.json、assets/*
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 7. 生成 Blob     │  ← zip.generateAsync({ type: 'blob' })
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 8. 觸發下載      │  ← URL.createObjectURL() + <a>.click()
└──────────────────┘
```

### 5.2 類別設計

```typescript
class ProjectExporter {
  /**
   * 從所有 Store 收集狀態與資源，生成 zip Blob。
   * @returns zip 檔案的 Blob 物件
   */
  static async export(stores: AllStores): Promise<Blob> {
    const manifest = this.buildManifest(stores);
    const state = this.buildState(stores);
    const zip = new JSZip();

    zip.file('manifest.json', JSON.stringify(manifest, null, 2));
    zip.file('state.json', JSON.stringify(state, null, 2));

    await this.addAssets(zip, stores);
    await this.addThumbnail(zip, stores);

    return zip.generateAsync({
      type: 'blob',
      compression: 'DEFLATE',
      compressionOptions: { level: 6 },
    });
  }

  /**
   * 匯出並直接觸發瀏覽器下載。
   * @param filename 下載檔名，預設為 "[projectName].zip"
   */
  static async exportToFile(
    stores: AllStores,
    filename?: string,
  ): Promise<void> {
    const blob = await this.export(stores);
    const name = filename ?? `${stores.project.projectName}.zip`;
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = name;
    a.click();
    URL.revokeObjectURL(url);
  }

  private static buildManifest(stores: AllStores): ProjectManifest { /* ... */ }
  private static buildState(stores: AllStores): ProjectState { /* ... */ }
  private static async addAssets(zip: JSZip, stores: AllStores): Promise<void> { /* ... */ }
  private static async addThumbnail(zip: JSZip, stores: AllStores): Promise<void> { /* ... */ }
}
```

### 5.3 路徑轉換

匯出時需將運行時路徑（Blob URL 或本地路徑）轉換為 zip 內的相對路徑：

```
運行時路徑                                     zip 內路徑
──────────────────────────────────          ─────────────────────────────
blob:http://localhost:5173/abc123    →    assets/spine/hero/hero.skel
blob:http://localhost:5173/def456    →    assets/sprites/background.png
```

轉換規則：

| 資源類型 | zip 路徑模板 |
|----------|-------------|
| Spine 骨骼檔 | `assets/spine/<name>/<name>.skel` |
| Spine 圖集 | `assets/spine/<name>/<name>.atlas` |
| Spine 貼圖 | `assets/spine/<name>/<filename>.png` |
| 靜態圖片 | `assets/sprites/<filename>` |
| 腳本 | `assets/scripts/<scriptName>.json` |

### 5.4 錯誤處理

| 情境 | 處理方式 |
|------|----------|
| 資源檔案遺失 | 發出警告並跳過該資源，不阻斷匯出流程 |
| Blob URL 失效 | 記錄錯誤日誌，標記該資源為 `missing` |
| zip 超過 100MB | 顯示提示訊息，建議使用者清理未使用的資源 |
| 生成失敗 | 顯示錯誤 Toast，記錄完整 stack trace |

---

## 6. 匯入流程（ProjectImporter）

### 6.1 流程圖

```
使用者選擇 .zip 檔案（檔案選擇器 / 拖拽）
        │
        ▼
┌──────────────────────┐
│ 1. 讀取 ZIP          │  ← JSZip.loadAsync(file)
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 2. 解析 manifest     │  ← 讀取 manifest.json，驗證格式
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 3. 版本檢查          │  ← 比對 schemaVersion 與當前版本
└────────┬─────────────┘
         │ 版本不符？
         ├── 是 ──→ 執行 Migration（見第 7 節）
         │
         ▼
┌──────────────────────┐
│ 4. 解析 state.json   │  ← JSON.parse + 型別驗證
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 5. 提取資源           │  ← 從 zip 讀取 assets/*，建立 Blob URL
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 6. 路徑映射           │  ← 將 state 中的相對路徑替換為 Blob URL
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 7. 注入 Store         │  ← 將狀態寫入所有 Zustand Store
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 8. 載入 Spine 資源    │  ← 用 Blob URL 載入 Spine 骨骼動畫
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ 9. 重建場景           │  ← PixiJS Managers 根據 Store 重建 Scene Graph
└──────────────────────┘
```

### 6.2 類別設計

```typescript
class ProjectImporter {
  /**
   * 解析 zip 檔案並回傳專案狀態（不套用至 Store）。
   * 用於預覽或驗證。
   */
  static async import(file: File): Promise<ImportResult> {
    const zip = await JSZip.loadAsync(file);

    const manifest = await this.parseManifest(zip);
    const migratedState = await this.parseAndMigrateState(zip, manifest);
    const assetMap = await this.extractAssets(zip);

    return {
      manifest,
      state: this.resolveAssetPaths(migratedState, assetMap),
      assetMap,
    };
  }

  /**
   * 解析 zip 並直接套用至所有 Zustand Store，重建完整專案。
   */
  static async importAndApply(
    file: File,
    stores: AllStores,
  ): Promise<void> {
    const result = await this.import(file);

    this.cleanupPreviousProject(stores);
    this.hydrateStores(result, stores);
    await this.loadSpineResources(result, stores);
    stores.project.markClean();
  }

  private static async parseManifest(zip: JSZip): Promise<ProjectManifest> { /* ... */ }
  private static async parseAndMigrateState(zip: JSZip, manifest: ProjectManifest): Promise<ProjectState> { /* ... */ }
  private static async extractAssets(zip: JSZip): Promise<Map<string, string>> { /* ... */ }
  private static resolveAssetPaths(state: ProjectState, assetMap: Map<string, string>): ProjectState { /* ... */ }
  private static cleanupPreviousProject(stores: AllStores): void { /* ... */ }
  private static hydrateStores(result: ImportResult, stores: AllStores): void { /* ... */ }
  private static async loadSpineResources(result: ImportResult, stores: AllStores): Promise<void> { /* ... */ }
}

interface ImportResult {
  manifest: ProjectManifest;
  state: ProjectState;
  /** 相對路徑 → Blob URL 的映射表 */
  assetMap: Map<string, string>;
}
```

### 6.3 驗證規則

匯入時依序執行以下驗證，若關鍵驗證失敗則中止匯入：

| 驗證項目 | 嚴重程度 | 失敗行為 |
|----------|----------|----------|
| `manifest.json` 存在 | 🔴 致命 | 中止匯入，顯示錯誤 |
| `state.json` 存在 | 🔴 致命 | 中止匯入，顯示錯誤 |
| `schemaVersion` 格式正確 | 🔴 致命 | 中止匯入，顯示錯誤 |
| `schemaVersion` ≤ 當前版本 | 🔴 致命 | 中止匯入，提示升級工具 |
| Layer 引用的 `clipId` 存在 | 🟡 警告 | 移除無效引用，繼續匯入 |
| Spine 資源檔案完整 | 🟡 警告 | 標記為未載入，繼續匯入 |
| 靜態圖片檔案存在 | 🟡 警告 | 使用佔位圖，繼續匯入 |
| 腳本引用的 Layer 存在 | 🟡 警告 | 記錄日誌，繼續匯入 |

### 6.4 資源提取策略

```typescript
async function extractAssets(zip: JSZip): Promise<Map<string, string>> {
  const assetMap = new Map<string, string>();

  for (const [path, zipEntry] of Object.entries(zip.files)) {
    if (zipEntry.dir || !path.startsWith('assets/')) continue;

    const blob = await zipEntry.async('blob');
    const blobUrl = URL.createObjectURL(blob);
    assetMap.set(path, blobUrl);
  }

  return assetMap;
}
```

---

## 7. Schema 版本遷移

### 7.1 設計理念

Schema 版本遷移確保**舊版專案檔可在新版工具中正確開啟**。遷移採用**鏈式策略**——每個版本間的差異由獨立的 migration 函式處理，多個 migration 可串接執行。

```
舊版專案檔（v1.0.0）
    │
    ▼ migration 1.0.0 → 1.1.0
    │
    ▼ migration 1.1.0 → 1.2.0
    │
    ▼ migration 1.2.0 → 2.0.0
    │
當前版本狀態（v2.0.0）
```

### 7.2 型別定義

```typescript
interface Migration {
  /** 來源版本 */
  fromVersion: string;
  /** 目標版本 */
  toVersion: string;
  /** 遷移函式，接收舊版 state 並回傳新版 state */
  migrate(state: any): any;
}

class MigrationRunner {
  private static migrations: Migration[] = [];

  /** 註冊一個版本遷移 */
  static register(migration: Migration): void {
    this.migrations.push(migration);
    this.migrations.sort((a, b) => a.fromVersion.localeCompare(b.fromVersion));
  }

  /**
   * 執行從 fromVersion 到 toVersion 的所有遷移。
   * 遷移函式依版本順序鏈式執行。
   * @throws 若找不到遷移路徑或目標版本高於當前版本
   */
  static migrate(state: any, fromVersion: string, toVersion: string): any {
    if (fromVersion === toVersion) return state;

    const chain = this.buildMigrationChain(fromVersion, toVersion);
    if (!chain) {
      throw new Error(
        `無法從版本 ${fromVersion} 遷移至 ${toVersion}：找不到遷移路徑`,
      );
    }

    let current = state;
    for (const migration of chain) {
      current = migration.migrate(current);
    }
    return current;
  }

  private static buildMigrationChain(
    from: string,
    to: string,
  ): Migration[] | null { /* ... */ }
}
```

### 7.3 遷移範例

```typescript
// 1.0.0 → 1.1.0：為 LayerNode 新增 rotation 欄位
MigrationRunner.register({
  fromVersion: '1.0.0',
  toVersion: '1.1.0',
  migrate(state: any) {
    state.layers = state.layers.map((layer: any) => ({
      ...layer,
      rotation: layer.rotation ?? 0,
    }));
    return state;
  },
});

// 1.1.0 → 1.2.0：將 preview.bgColor 重新命名為 preview.backgroundColor
MigrationRunner.register({
  fromVersion: '1.1.0',
  toVersion: '1.2.0',
  migrate(state: any) {
    if (state.preview?.bgColor !== undefined) {
      state.preview.backgroundColor = state.preview.bgColor;
      delete state.preview.bgColor;
    }
    return state;
  },
});
```

### 7.4 版本相容性規則

| 情境 | 行為 |
|------|------|
| 專案版本 === 當前版本 | 直接載入，無需遷移 |
| 專案版本 < 當前版本 | 執行鏈式遷移後載入 |
| 專案版本 > 當前版本 | **拒絕載入**，顯示錯誤訊息：「此專案檔由較新版本的工具建立，請更新 Slot Previewer」 |
| 遷移路徑中斷 | **拒絕載入**，顯示錯誤訊息：「無法遷移至當前版本」 |

### 7.5 Schema 設計原則

| 原則 | 說明 |
|------|------|
| 只加不刪 | 新增欄位時使用可選欄位（`?`），永遠不刪除既有欄位 |
| 預設值 | 新增欄位必須提供合理的預設值，migration 中補上 |
| 向前相容 | 舊版檔案匯入時，新欄位使用預設值填充 |
| SemVer | 非破壞性變更遞增 minor，破壞性變更遞增 major |

---

## 8. 資源路徑管理

### 8.1 路徑生命週期

資源路徑在專案的不同階段有不同的表示形式：

```
    匯入時                  運行時                     匯出時
┌────────────┐      ┌──────────────────┐       ┌────────────┐
│ ZIP 相對路徑 │  →   │   Blob URL       │   →   │ ZIP 相對路徑 │
│ assets/    │      │ blob:http://...  │       │ assets/    │
│ spine/hero/│      │ /abc123          │       │ spine/hero/│
│ hero.skel  │      │                  │       │ hero.skel  │
└────────────┘      └──────────────────┘       └────────────┘
```

### 8.2 路徑類型

| 路徑類型 | 格式 | 使用場景 |
|----------|------|----------|
| ZIP 相對路徑 | `assets/spine/hero/hero.skel` | `state.json` 內的資源引用 |
| Blob URL | `blob:http://localhost:5173/abc123` | 運行時 PixiJS 載入資源 |
| 原始路徑 | 拖拽匯入時的檔案路徑 | 首次匯入素材 |

### 8.3 路徑映射表

匯入時建立一份 `Map<string, string>` 作為路徑映射表，將 zip 內的相對路徑對應到運行時的 Blob URL：

```typescript
// 匯入時建立映射
const assetMap = new Map<string, string>();
assetMap.set('assets/spine/hero/hero.skel', 'blob:http://..../abc123');
assetMap.set('assets/spine/hero/hero.atlas', 'blob:http://..../def456');
assetMap.set('assets/sprites/background.png', 'blob:http://..../ghi789');
```

`state.json` 中所有資源路徑在注入 Store 前，皆透過此映射表替換為 Blob URL。

### 8.4 Blob URL 生命週期管理

Blob URL 佔用記憶體，必須在適當時機釋放：

| 時機 | 動作 |
|------|------|
| 匯入新專案 | 清除舊專案的所有 Blob URL（`URL.revokeObjectURL()`） |
| 關閉專案 | 清除所有 Blob URL |
| 替換單一資源 | 清除被替換資源的 Blob URL |
| 頁面卸載 | `beforeunload` 事件中清除所有 Blob URL |

```typescript
class BlobUrlManager {
  private urls = new Set<string>();

  create(blob: Blob): string {
    const url = URL.createObjectURL(blob);
    this.urls.add(url);
    return url;
  }

  revoke(url: string): void {
    URL.revokeObjectURL(url);
    this.urls.delete(url);
  }

  revokeAll(): void {
    for (const url of this.urls) {
      URL.revokeObjectURL(url);
    }
    this.urls.clear();
  }
}
```

---

## 9. 使用者流程

### 9.1 匯出專案

```
使用者                          系統
  │                              │
  │  點擊「匯出專案」             │
  │  （或 Ctrl+S）               │
  │ ────────────────────────────→│
  │                              │  收集 Store 狀態 + 資源
  │                              │  建構 manifest.json
  │                              │  建構 state.json
  │       顯示進度條              │  JSZip 打包中...
  │ ←────────────────────────────│
  │                              │  生成 Blob
  │       瀏覽器下載對話框         │
  │ ←────────────────────────────│
  │                              │  預設檔名："[專案名稱].zip"
  │  選擇儲存位置                 │
  │ ────────────────────────────→│
  │                              │  下載完成
  │       顯示成功 Toast          │
  │ ←────────────────────────────│
```

**快捷鍵**：`Ctrl+S`（macOS：`⌘+S`）

### 9.2 匯入專案

```
使用者                          系統
  │                              │
  │  點擊「開啟專案」             │
  │  （或 Ctrl+O / 拖拽 .zip）   │
  │ ────────────────────────────→│
  │                              │
  │                              │  檢查 dirty 狀態
  │      ┌─────────────────┐     │
  │      │ 是否儲存目前專案？ │     │  （若專案已修改）
  │      │ [儲存] [不儲存] [取消]│  │
  │      └─────────────────┘     │
  │  選擇「不儲存」               │
  │ ────────────────────────────→│
  │                              │
  │                              │  讀取 .zip
  │                              │  解析 manifest.json
  │                              │  版本檢查 + migration
  │                              │  解析 state.json
  │       顯示匯入進度            │  提取資源至記憶體
  │ ←────────────────────────────│
  │                              │  建立 Blob URL
  │                              │  注入 Zustand Store
  │                              │  載入 Spine 資源
  │                              │  重建 PixiJS Scene Graph
  │       UI 更新完成             │
  │ ←────────────────────────────│
  │                              │  顯示成功 Toast
```

**支援的匯入方式**：
- 工具列「開啟專案」按鈕
- 快捷鍵 `Ctrl+O`（macOS：`⌘+O`）
- 拖拽 `.zip` 檔案至畫布區域

### 9.3 拖拽匯入細節

```typescript
canvas.addEventListener('dragover', (e) => {
  e.preventDefault();
  e.dataTransfer!.dropEffect = 'copy';
  showDropZoneOverlay();
});

canvas.addEventListener('drop', async (e) => {
  e.preventDefault();
  hideDropZoneOverlay();

  const file = e.dataTransfer?.files[0];
  if (file?.name.endsWith('.zip')) {
    await ProjectImporter.importAndApply(file, stores);
  }
});
```

---

## 10. 效能考量

### 10.1 大型 ZIP 處理

| 策略 | 說明 |
|------|------|
| **延遲提取** | JSZip 支援 lazy extraction，僅在需要時解壓特定檔案，避免一次將所有資源載入記憶體。 |
| **串流式讀取** | 使用 `zip.file(path).async('blob')` 逐檔提取，而非 `zip.generateAsync()` 一次性處理。 |
| **分批載入** | Spine 資源按順序逐一載入，每載入一組即更新進度條，避免瀏覽器卡頓。 |

### 10.2 進度回報

匯出與匯入皆提供進度回報機制，透過回呼函式更新 UI：

```typescript
interface ProgressCallback {
  (progress: {
    phase: 'collecting' | 'compressing' | 'extracting' | 'loading';
    current: number;
    total: number;
    message: string;
  }): void;
}

// 使用範例
await ProjectExporter.export(stores, (progress) => {
  uiStore.setExportProgress(progress.current / progress.total);
  uiStore.setExportMessage(progress.message);
});
```

### 10.3 記憶體管理

| 策略 | 說明 |
|------|------|
| **不持有原始 ArrayBuffer** | 資源轉為 Blob URL 後，釋放 zip entry 的 ArrayBuffer 參照。 |
| **即時回收** | 專案替換時，先 `revokeAll()` 舊 Blob URL，再建立新的。 |
| **GC 友善** | 避免在全域持有大型物件的參照，使用 WeakRef 或明確的 cleanup 機制。 |

### 10.4 Web Worker（未來規劃）

對於大型專案（> 50MB），可將 zip 壓縮 / 解壓工作移至 Web Worker，避免阻塞主執行緒：

```
主執行緒                        Web Worker
    │                              │
    │  postMessage(assets)   →     │
    │                              │  JSZip 壓縮
    │                              │  ...
    │     ← postMessage(blob)      │
    │                              │
    │  觸發下載                     │
```

---

## 11. 擴充性

### 11.1 新增資源類型

當專案需要支援新的資源類型（例如音效、影片）時，遵循以下步驟：

1. 在 `assets/` 下新增對應子目錄（如 `assets/audio/`）
2. 在 `manifest.json` 的 `assetCount` 中新增統計欄位
3. 在 `state.json` 中新增對應的資源參照陣列
4. 在 `ProjectExporter` 和 `ProjectImporter` 中處理新目錄
5. 遞增 `schemaVersion` 並撰寫 migration

```
project.zip/
├── manifest.json
├── state.json
├── assets/
│   ├── spine/        ← 既有
│   ├── sprites/      ← 既有
│   ├── scripts/      ← 既有
│   ├── audio/        ← 新增
│   │   ├── bgm.mp3
│   │   └── win.ogg
│   └── video/        ← 新增
│       └── intro.mp4
└── thumbnail.png
```

### 11.2 雲端儲存（未來擴充點）

目前為純客戶端方案，但架構已預留雲端擴充點：

```
目前：Export → Blob → 瀏覽器下載
未來：Export → Blob → 上傳至雲端儲存（S3 / GCS）→ 產生分享連結
```

```typescript
interface StorageAdapter {
  save(blob: Blob, filename: string): Promise<string>;
  load(identifier: string): Promise<File>;
  list(): Promise<FileEntry[]>;
}

class LocalStorageAdapter implements StorageAdapter { /* 瀏覽器下載 */ }
class CloudStorageAdapter implements StorageAdapter { /* 雲端上傳 */ }
```

### 11.3 自動儲存草稿（未來考量）

定期將專案狀態匯出至 IndexedDB，作為瀏覽器內的自動備份：

```typescript
class AutoSaveManager {
  private interval: number = 60_000; // 每 60 秒

  start(stores: AllStores): void {
    setInterval(async () => {
      if (!stores.project.dirty) return;
      const blob = await ProjectExporter.export(stores);
      await idb.put('autosave', blob);
      stores.project.setLastAutoSave(new Date().toISOString());
    }, this.interval);
  }

  async restore(): Promise<File | null> {
    const blob = await idb.get('autosave');
    return blob ? new File([blob], 'autosave.zip') : null;
  }
}
```

### 11.4 範本專案

提供預建的 `.zip` 範本作為起始點，降低使用者上手門檻：

| 範本 | 內容 |
|------|------|
| 空白專案 | 僅含基本 manifest + 預設 preview 設定 |
| 5x3 標準轉輪 | 預設 5 軸 3 列盤面 + 基礎 Symbol |
| Megaways 風格 | 可變列數盤面 + 進階動畫腳本 |
| Big Win 演出 | 完整 Big Win 動畫流程腳本 |

---

> **上一份文件**：`09-ui-layout.md` — UI 佈局與面板設計規格
