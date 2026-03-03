# Slot Previewer — 測試策略

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管

---

## 1. 測試策略概覽

Slot Previewer 採用分層測試策略，結合 **Vitest**（單元 + 整合測試）與 **Playwright**（E2E 測試），在開發效率與品質保障之間取得平衡。

| 測試層級 | 工具 | 範圍 | 目的 |
|----------|------|------|------|
| **單元測試** | Vitest | 純邏輯函式（Store actions、ReelConfig 驗證、腳本解析） | 確保各個函式與模組的正確性 |
| **整合測試** | Vitest | 模組互動（Store → Manager 同步、ScriptRunner → Actions） | 驗證模組之間的協作行為 |
| **E2E 測試** | Playwright | 完整使用者流程（載入、操作、匯出） | 模擬真實操作，確保端到端功能正常 |
| **視覺回歸** | Screenshot 比對 | Canvas 渲染結果 | 偵測畫面意外變化（未來規劃） |

### 核心原則

| 原則 | 說明 |
|------|------|
| **快速回饋** | 單元測試應在 5 秒內完成，整合測試在 30 秒內完成 |
| **可靠性** | 測試不依賴外部服務，所有依賴皆使用 mock / fixture |
| **可維護性** | 測試命名清晰，遵循統一規範，易於定位失敗原因 |
| **持續整合** | 所有測試自動化執行，與 CI/CD 管線整合 |

---

## 2. 測試金字塔

```
                    ╱  E2E  ╲              ← 少量，關鍵路徑
                   ╱──────────╲               Playwright
                  ╱ Integration ╲          ← 中等，模組整合
                 ╱────────────────╲           Vitest
                ╱   Unit  Tests    ╲       ← 大量，純邏輯
               ╱────────────────────╲         Vitest
```

| 層級 | 佔比 | 數量級 | 執行速度 |
|------|------|--------|----------|
| E2E | ~10% | 10–20 個 | 慢（數分鐘） |
| 整合 | ~20% | 20–40 個 | 中（數十秒） |
| 單元 | ~70% | 100+ 個 | 快（數秒） |

---

## 3. 單元測試 (Vitest)

### 3.1 Store 測試

每個 Zustand Store slice 獨立測試，確保 actions 產生正確的狀態變化。

```typescript
// layerStore.test.ts
describe('layerStore', () => {
  it('addLayer 應新增一個圖層並回傳 ID', () => {
    const id = layerStore.getState().addLayer({ type: 'sprite', name: '背景' });
    expect(layerStore.getState().layers).toHaveLength(1);
    expect(layerStore.getState().layers[0].id).toBe(id);
  });

  it('removeLayer 應移除指定圖層', () => {
    const id = layerStore.getState().addLayer({ type: 'sprite', name: '前景' });
    layerStore.getState().removeLayer(id);
    expect(layerStore.getState().layers).toHaveLength(0);
  });

  it('reorderLayers 應正確調整圖層順序', () => {
    // ...
  });
});
```

**測試重點**：

| Store Slice | 測試項目 |
|-------------|----------|
| `layerStore` | addLayer / removeLayer / reorderLayers / toggleVisibility / toggleLock |
| `reelStore` | setReelConfig / setSpinState / setResultMatrix |
| `spineStore` | loadSpine / setAnimation / setSkin / setSpeed |
| `scriptStore` | loadScript / setExecutionState / abort |
| `projectStore` | setDirty / setProjectMeta |
| `uiStore` | setActivePanel / setZoom / setPan |
| `canvasStore` | setSize / setBackground |

### 3.2 ReelConfig 驗證測試

驗證 ReelConfig 結構的正確性，確保無效配置被攔截。

```typescript
// reelConfigValidator.test.ts
describe('ReelConfig 驗證', () => {
  it('合法配置應通過驗證', () => {
    const config: ReelConfig = {
      reelCount: 5,
      rowsPerReel: [3, 3, 3, 3, 3],
      symbolSize: { width: 100, height: 100 },
      strips: [/* valid strips */],
    };
    expect(validateReelConfig(config)).toEqual({ valid: true });
  });

  it('負數列數應回傳錯誤', () => {
    const config = { ...validConfig, rowsPerReel: [3, -1, 3, 3, 3] };
    const result = validateReelConfig(config);
    expect(result.valid).toBe(false);
    expect(result.errors).toContainEqual(
      expect.objectContaining({ field: 'rowsPerReel', message: expect.stringContaining('負數') })
    );
  });

  it('空 strips 陣列應回傳錯誤', () => {
    const config = { ...validConfig, strips: [] };
    const result = validateReelConfig(config);
    expect(result.valid).toBe(false);
    expect(result.errors).toContainEqual(
      expect.objectContaining({ field: 'strips' })
    );
  });

  it('reelCount 與 rowsPerReel 長度不符應回傳錯誤', () => {
    const config = { ...validConfig, reelCount: 5, rowsPerReel: [3, 3, 3] };
    const result = validateReelConfig(config);
    expect(result.valid).toBe(false);
  });
});
```

### 3.3 腳本解析測試

驗證 ScriptParser 對各種輸入的處理能力。

```typescript
// scriptParser.test.ts
describe('ScriptParser', () => {
  it('合法腳本應正確解析', () => {
    const script = '{"actions":[{"type":"spin","reelIndex":0}]}';
    const result = parseScript(script);
    expect(result.actions).toHaveLength(1);
    expect(result.actions[0].type).toBe('spin');
  });

  it('無效 JSON 應拋出 ParseError', () => {
    expect(() => parseScript('{invalid}')).toThrow(ParseError);
  });

  it('未知 action type 應以警告處理而非中斷', () => {
    const script = '{"actions":[{"type":"unknownAction"}]}';
    const result = parseScript(script);
    expect(result.warnings).toContainEqual(
      expect.objectContaining({ type: 'unknown_action' })
    );
  });

  it('循環 callScript 應偵測並回報錯誤', () => {
    const scripts = {
      'a.json': '{"actions":[{"type":"callScript","target":"b.json"}]}',
      'b.json': '{"actions":[{"type":"callScript","target":"a.json"}]}',
    };
    expect(() => validateScriptGraph(scripts)).toThrow(CircularDependencyError);
  });
});
```

### 3.4 ObjectPool 測試

驗證物件池的生命週期管理行為。

```typescript
// objectPool.test.ts
describe('ObjectPool', () => {
  it('acquire 應回傳可用物件', () => {
    const pool = new ObjectPool(() => new MockSymbol(), 5);
    const obj = pool.acquire();
    expect(obj).toBeInstanceOf(MockSymbol);
  });

  it('release 後物件應可重複使用', () => {
    const pool = new ObjectPool(() => new MockSymbol(), 5);
    const obj = pool.acquire();
    pool.release(obj);
    const reused = pool.acquire();
    expect(reused).toBe(obj);
  });

  it('池耗盡時應自動擴容', () => {
    const pool = new ObjectPool(() => new MockSymbol(), 2);
    const a = pool.acquire();
    const b = pool.acquire();
    const c = pool.acquire(); // 超過初始容量
    expect(c).toBeInstanceOf(MockSymbol);
    expect(pool.size).toBeGreaterThan(2);
  });

  it('cleanup 應銷毀閒置物件並縮減池大小', () => {
    const pool = new ObjectPool(() => new MockSymbol(), 10);
    for (let i = 0; i < 10; i++) pool.acquire();
    for (let i = 0; i < 10; i++) pool.release(pool['_active'][0]);
    pool.cleanup();
    expect(pool.size).toBeLessThan(10);
  });
});
```

---

## 4. 整合測試

### 4.1 Store → Manager 同步

驗證 Zustand Store 的狀態變化能正確反映到 PixiJS 渲染層。

```typescript
// store-sync.integration.test.ts
describe('Store → Manager 同步', () => {
  let pixiApp: Application;
  let layerManager: LayerManager;

  beforeEach(async () => {
    pixiApp = new Application();
    await pixiApp.init({ width: 800, height: 600 });
    layerManager = new LayerManager(pixiApp.stage);
  });

  afterEach(() => {
    pixiApp.destroy(true);
  });

  it('Store 新增圖層應觸發 Manager 建立對應 Container', () => {
    const id = layerStore.getState().addLayer({ type: 'sprite', name: '背景' });
    layerManager.syncFromStore(layerStore.getState());
    expect(layerManager.getContainer(id)).toBeDefined();
    expect(pixiApp.stage.children).toHaveLength(1);
  });

  it('Store 移除圖層應觸發 Manager 銷毀對應 Container', () => {
    const id = layerStore.getState().addLayer({ type: 'sprite', name: '前景' });
    layerManager.syncFromStore(layerStore.getState());
    layerStore.getState().removeLayer(id);
    layerManager.syncFromStore(layerStore.getState());
    expect(layerManager.getContainer(id)).toBeUndefined();
    expect(pixiApp.stage.children).toHaveLength(0);
  });

  it('Store 更新圖層屬性應反映到 Container', () => {
    const id = layerStore.getState().addLayer({ type: 'sprite', name: '前景' });
    layerManager.syncFromStore(layerStore.getState());
    layerStore.getState().updateLayer(id, { alpha: 0.5, visible: false });
    layerManager.syncFromStore(layerStore.getState());
    const container = layerManager.getContainer(id);
    expect(container?.alpha).toBe(0.5);
    expect(container?.visible).toBe(false);
  });
});
```

### 4.2 ScriptRunner → Actions

驗證腳本引擎的執行流程與控制行為。

```typescript
// script-execution.integration.test.ts
describe('ScriptRunner 執行流程', () => {
  it('應按順序執行所有 actions', async () => {
    const executionOrder: string[] = [];
    const script = createScript([
      { type: 'spin', onExecute: () => executionOrder.push('spin') },
      { type: 'wait', duration: 100, onExecute: () => executionOrder.push('wait') },
      { type: 'playSpine', onExecute: () => executionOrder.push('playSpine') },
    ]);
    await scriptRunner.execute(script);
    expect(executionOrder).toEqual(['spin', 'wait', 'playSpine']);
  });

  it('parallel actions 應同時執行', async () => {
    const startTimes: Record<string, number> = {};
    const script = createScript([
      {
        type: 'parallel',
        children: [
          { type: 'spin', onExecute: () => { startTimes.spin = Date.now(); } },
          { type: 'playSpine', onExecute: () => { startTimes.playSpine = Date.now(); } },
        ],
      },
    ]);
    await scriptRunner.execute(script);
    const diff = Math.abs(startTimes.spin - startTimes.playSpine);
    expect(diff).toBeLessThan(50); // 幾乎同時啟動
  });

  it('pause / resume 應正確暫停與恢復', async () => {
    let completed = false;
    const script = createScript([
      { type: 'wait', duration: 500, onComplete: () => { completed = true; } },
    ]);
    const execution = scriptRunner.execute(script);
    await delay(100);
    scriptRunner.pause();
    await delay(600);
    expect(completed).toBe(false); // 暫停期間不應完成
    scriptRunner.resume();
    await execution;
    expect(completed).toBe(true);
  });

  it('abort 應立即中止執行', async () => {
    const script = createScript([
      { type: 'wait', duration: 5000 },
      { type: 'spin' },
    ]);
    const execution = scriptRunner.execute(script);
    await delay(100);
    scriptRunner.abort();
    await expect(execution).rejects.toThrow(AbortError);
  });
});
```

---

## 5. E2E 測試 (Playwright)

### 5.1 關鍵使用者流程

以下為必須覆蓋的關鍵 E2E 測試案例：

| # | 流程 | 描述 | 驗證方式 |
|---|------|------|----------|
| 1 | **應用程式啟動** | 開啟應用程式，Canvas 正確渲染 | 檢查 Canvas 元素存在且尺寸正確 |
| 2 | **Spine 拖拽載入** | 拖放 .skel + .atlas + .png 至 Canvas | Spine 角色在畫面中可見 |
| 3 | **圖層管理** | 新增 / 移除 / 重新排序圖層 | 圖層面板反映正確狀態 |
| 4 | **轉輪配置與旋轉** | 設定 5×3 盤面，執行 Spin | 轉輪動畫播放且正確停輪 |
| 5 | **腳本載入與執行** | 載入 JSON 腳本並執行 | 依序觸發所有 actions |
| 6 | **專案匯出與匯入** | 匯出為 .zip → 重新匯入 | 匯入後狀態與匯出前一致 |

```typescript
// layer-management.e2e.ts
import { test, expect } from '@playwright/test';

test.describe('圖層管理', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:5173');
    await page.waitForSelector('canvas');
  });

  test('新增圖層應出現在圖層面板', async ({ page }) => {
    await page.click('[data-testid="add-layer-btn"]');
    await page.fill('[data-testid="layer-name-input"]', '新圖層');
    await page.click('[data-testid="confirm-btn"]');
    const layer = page.locator('[data-testid="layer-item"]').last();
    await expect(layer).toContainText('新圖層');
  });

  test('拖拽排序應更新圖層順序', async ({ page }) => {
    // 新增兩個圖層
    await addLayer(page, '圖層 A');
    await addLayer(page, '圖層 B');
    // 拖拽 B 到 A 之上
    const layerB = page.locator('[data-testid="layer-item"]').last();
    const layerA = page.locator('[data-testid="layer-item"]').first();
    await layerB.dragTo(layerA);
    // 驗證順序
    const names = await page.locator('[data-testid="layer-name"]').allTextContents();
    expect(names).toEqual(['圖層 B', '圖層 A']);
  });
});
```

### 5.2 測試環境

```
┌─────────────────────────────────────────────┐
│  Playwright Test Runner                      │
│  ┌────────────────┐    ┌──────────────────┐  │
│  │  Test Scripts   │───▶│  Chromium Browser │  │
│  └────────────────┘    └────────┬─────────┘  │
│                                │              │
│                        ┌───────▼──────────┐  │
│                        │  Vite Dev Server  │  │
│                        │  localhost:5173   │  │
│                        └──────────────────┘  │
│                                               │
│  ┌────────────────────────────────────────┐  │
│  │  Fixture Files                         │  │
│  │  tests/fixtures/                       │  │
│  │  ├── sample.skel                       │  │
│  │  ├── sample.atlas                      │  │
│  │  ├── sample.png                        │  │
│  │  ├── test-script.json                  │  │
│  │  └── test-project.zip                  │  │
│  └────────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

| 配置項目 | 設定值 |
|----------|--------|
| 瀏覽器 | Chromium（Playwright 內建） |
| 測試目標 | Vite Dev Server (`localhost:5173`) |
| 測試資產 | `tests/fixtures/` 目錄下的靜態檔案 |
| 截圖策略 | 失敗時自動截圖，存放於 `test-results/` |
| 重試策略 | CI 環境重試 2 次，本地不重試 |

---

## 6. 測試命名規範

| 層級 | 檔案命名格式 | 範例 |
|------|-------------|------|
| 單元測試 | `[module].test.ts` | `layerStore.test.ts` |
| 整合測試 | `[module].integration.test.ts` | `store-sync.integration.test.ts` |
| E2E 測試 | `[flow].e2e.ts` | `project-io.e2e.ts` |

### 測試案例命名規則

遵循 **「主詞 → 行為 → 預期結果」** 模式（中文描述）：

```typescript
// ✅ 好的命名
it('addLayer 應新增一個圖層並回傳 ID')
it('負數列數應回傳驗證錯誤')
it('循環 callScript 應偵測並拋出 CircularDependencyError')

// ❌ 不好的命名
it('test addLayer')
it('works correctly')
it('should work')
```

---

## 7. CI/CD 整合

```
┌──────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                          │
│                                                            │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   PR 建立    │───▶│  Unit Tests  │───▶│  Integration │  │
│  │             │    │   (Vitest)   │    │   (Vitest)   │  │
│  └─────────────┘    └──────┬───────┘    └──────┬───────┘  │
│                            │                    │          │
│                            ▼                    ▼          │
│                     ┌──────────────────────────────┐      │
│                     │      ✅ 通過 → 允許合併       │      │
│                     │      ❌ 失敗 → 阻止合併       │      │
│                     └──────────────┬───────────────┘      │
│                                    │                       │
│  ┌─────────────┐                   ▼                       │
│  │ 合併至 main  │───▶  ┌──────────────────┐               │
│  │             │      │   E2E Tests      │               │
│  └─────────────┘      │  (Playwright)    │               │
│                        └──────┬───────────┘               │
│                               ▼                            │
│                        ┌──────────────────┐               │
│                        │  ✅ 部署至 Staging │               │
│                        └──────────────────┘               │
└──────────────────────────────────────────────────────────┘
```

| 階段 | 觸發時機 | 執行內容 | 超時限制 |
|------|---------|----------|----------|
| **PR 檢查** | 每次 PR 建立 / 更新 | Unit + Integration Tests | 5 分鐘 |
| **合併檢查** | 合併至 `main` | E2E Tests | 15 分鐘 |
| **覆蓋率門檻** | 每次 PR | 核心邏輯模組 ≥ 80% | — |

### 覆蓋率目標

| 模組分類 | 覆蓋率目標 | 說明 |
|----------|-----------|------|
| 核心邏輯（Store, ReelEngine, ScriptRunner） | ≥ 80% | 業務關鍵模組，必須高覆蓋 |
| 工具函式（utils, validators） | ≥ 90% | 純函式，易於測試 |
| UI 元件 | ≥ 50% | 以 E2E 補足視覺驗證 |
| PixiJS 渲染層 | 不強制 | 依賴 Canvas，以 E2E + 視覺回歸覆蓋 |

---

## 8. 測試目錄結構

```
src/
├── store/
│   ├── layerStore.ts
│   ├── reelStore.ts
│   ├── spineStore.ts
│   ├── scriptStore.ts
│   └── __tests__/
│       ├── layerStore.test.ts
│       ├── reelStore.test.ts
│       ├── spineStore.test.ts
│       └── scriptStore.test.ts
├── reel/
│   ├── ReelEngine.ts
│   ├── SymbolPool.ts
│   ├── ReelConfigValidator.ts
│   └── __tests__/
│       ├── ReelEngine.test.ts
│       ├── SymbolPool.test.ts
│       └── ReelConfigValidator.test.ts
├── script/
│   ├── ScriptRunner.ts
│   ├── ScriptParser.ts
│   └── __tests__/
│       ├── ScriptRunner.test.ts
│       └── ScriptParser.test.ts
├── pixi/
│   ├── LayerManager.ts
│   ├── ClipManager.ts
│   └── __tests__/
│       ├── LayerManager.test.ts
│       └── ClipManager.test.ts
tests/
├── integration/
│   ├── store-sync.integration.test.ts
│   ├── script-execution.integration.test.ts
│   └── reel-spin.integration.test.ts
├── e2e/
│   ├── app-startup.e2e.ts
│   ├── layer-management.e2e.ts
│   ├── spine-preview.e2e.ts
│   ├── reel-spin.e2e.ts
│   ├── script-execution.e2e.ts
│   └── project-io.e2e.ts
└── fixtures/
    ├── sample.skel
    ├── sample.atlas
    ├── sample.png
    ├── valid-script.json
    ├── invalid-script.json
    ├── valid-reel-config.json
    └── test-project.zip
```

### 目錄規則

| 規則 | 說明 |
|------|------|
| **就近放置** | 單元測試放在對應模組的 `__tests__/` 資料夾中 |
| **集中管理** | 整合測試與 E2E 測試統一放在頂層 `tests/` 目錄 |
| **測試資產** | 所有測試用的靜態檔案放在 `tests/fixtures/` |
| **產出隔離** | 測試報告與截圖輸出至 `test-results/`（已列入 `.gitignore`） |

---

## 9. 工具配置參考

### Vitest 配置 (`vitest.config.ts`)

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    include: [
      'src/**/*.test.ts',
      'tests/integration/**/*.integration.test.ts',
    ],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.d.ts', 'src/**/__tests__/**'],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
    alias: {
      '@store': path.resolve(__dirname, 'src/store'),
      '@pixi': path.resolve(__dirname, 'src/pixi'),
      '@ui': path.resolve(__dirname, 'src/ui'),
    },
  },
});
```

### Playwright 配置 (`playwright.config.ts`)

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  testMatch: '**/*.e2e.ts',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html', { open: 'never' }]],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```
