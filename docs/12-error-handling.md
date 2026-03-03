# Slot Previewer — 錯誤處理與恢復策略

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 1. 錯誤處理原則

Slot Previewer 是一款面向**美術人員**的瀏覽器端工具，使用者不具備程式除錯能力。因此，錯誤處理的首要目標是：**絕不崩潰、永遠可恢復、訊息清楚可行動**。

| 原則 | 說明 |
|------|------|
| **優雅降級（Graceful Degradation）** | 任何錯誤都不應導致應用程式白屏或無回應；若某功能失敗，應降級處理並繼續運作。 |
| **可讀的錯誤訊息** | 所有面向使用者的訊息皆使用繁體中文撰寫，避免技術術語；訊息應說明「發生了什麼」與「可以做什麼」。 |
| **分級處理** | 將錯誤分為「可恢復」與「致命」兩大類，分別採用不同的處理策略與回饋方式。 |
| **完整日誌** | 所有錯誤皆記錄至瀏覽器 Console（含堆疊追蹤），供開發人員事後偵錯。 |
| **Toast 通知** | 使用者端錯誤回饋統一透過 Toast 通知系統呈現，確保一致的視覺體驗。 |

---

## 2. 錯誤分類

| 類別 | 嚴重程度 | 處理策略 | 範例 |
|------|----------|----------|------|
| **載入錯誤** | Medium | 顯示警告，跳過該資源並繼續載入 | Spine 檔案損壞、缺少紋理 |
| **驗證錯誤** | Medium | 顯示錯誤訊息，阻止操作繼續 | 腳本 JSON 格式錯誤、檔案類型不符 |
| **運行時錯誤** | High | 攔截例外並嘗試恢復 | PixiJS 渲染錯誤、腳本 Action 執行失敗 |
| **系統錯誤** | Critical | 顯示致命錯誤對話框 | WebGL Context Lost、記憶體耗盡 |
| **使用者錯誤** | Low | 顯示提示訊息引導修正 | 輸入值無效、名稱重複 |

### 嚴重程度處理流程

```
使用者操作 / 系統事件
        │
        ▼
  ┌───────────┐
  │ 發生錯誤   │
  └─────┬─────┘
        │
        ▼
  ┌───────────────┐     ┌──────────────────────┐
  │ 嚴重程度判斷   │────▶│ Low / Medium          │
  └───────┬───────┘     │ → Toast 通知          │
          │             │ → 降級處理或阻止操作   │
          │             └──────────────────────┘
          │
          ▼
  ┌───────────────┐     ┌──────────────────────┐
  │ High          │────▶│ 攔截 + 嘗試恢復       │
  └───────┬───────┘     │ → Toast + Console Log │
          │             └──────────────────────┘
          │
          ▼
  ┌───────────────┐     ┌──────────────────────┐
  │ Critical      │────▶│ 致命錯誤對話框        │
  └───────────────┘     │ → 重試 / 重新載入      │
                        └──────────────────────┘
```

---

## 3. 各模組錯誤處理

### 3.1 PixiJS / 渲染層

渲染層是整個應用程式的基礎，必須確保渲染迴圈永不中斷。

| 錯誤場景 | 處理方式 | 使用者回饋 |
|----------|----------|------------|
| WebGL Context Lost | 嘗試自動恢復；重建 Scene Graph | Toast：「正在恢復渲染…」 |
| 紋理載入失敗 | 使用佔位紋理（placeholder）替代 | Console 警告 |
| Render Loop 例外 | 在 Ticker 回呼中 `try-catch`，記錄錯誤後繼續 | Console 錯誤 |

```typescript
// Ticker 保護範例
app.ticker.add(() => {
  try {
    renderScene();
  } catch (err) {
    console.error('[Renderer] Render loop error:', err);
  }
});

// WebGL Context Lost 恢復
canvas.addEventListener('webglcontextlost', (e) => {
  e.preventDefault();
  toastStore.warning('正在恢復渲染，請稍候…');
});

canvas.addEventListener('webglcontextrestored', () => {
  rebuildSceneGraph();
  toastStore.success('渲染已恢復');
});
```

### 3.2 Spine 載入與播放

Spine 資源由美術提供，格式與內容不可預期，需進行完整的防禦性檢查。

| 錯誤場景 | 處理方式 | 使用者回饋 |
|----------|----------|------------|
| 無效的 `.skel` 檔案 | 中止載入，不加入圖層 | Toast：「Spine 骨骼檔案格式錯誤」 |
| 缺少 `.atlas` 檔案 | 中止載入 | Toast：「缺少 Atlas 檔案」 |
| 缺少紋理圖片 | 使用佔位 Sprite 替代 | Toast：「部分紋理缺失，已使用預設圖片替代」 |
| 動畫名稱不存在 | 忽略該動畫指令 | Console 警告 |
| Worker 崩潰 | 退回主執行緒解析 | Toast：「Spine 解析改為同步模式，可能造成短暫延遲」 |

```typescript
async function loadSpine(files: SpineFiles): Promise<SpineAsset | null> {
  try {
    const asset = await spineLoader.load(files);
    return asset;
  } catch (err) {
    if (err instanceof InvalidSkelError) {
      toastStore.error('Spine 骨骼檔案格式錯誤，請確認檔案是否完整。');
    } else if (err instanceof MissingAtlasError) {
      toastStore.error('缺少 Atlas 檔案，請同時提供 .skel 與 .atlas。');
    } else {
      toastStore.error('Spine 載入失敗，請檢查檔案是否正確。');
    }
    console.error('[SpineLoader] Load failed:', err);
    return null;
  }
}
```

### 3.3 轉輪系統

轉輪系統的錯誤主要來自配置驗證與物件池管理。

| 錯誤場景 | 處理方式 | 使用者回饋 |
|----------|----------|------------|
| 無效的盤面配置 | 輸入時即時驗證，阻止儲存 | 欄位紅框 + 行內錯誤訊息 |
| Symbol 物件池耗盡 | 動態擴充物件池，記錄警告 | Console 警告 |
| 結果矩陣維度錯誤 | 驗證行列數，拒絕套用 | Toast：「結果矩陣維度與盤面配置不符」 |

```typescript
function validateReelConfig(config: ReelConfig): ValidationResult {
  const errors: string[] = [];

  if (config.reelCount < 1 || config.reelCount > 10) {
    errors.push('轉軸數量必須介於 1 到 10 之間');
  }

  for (let i = 0; i < config.rowsPerReel.length; i++) {
    if (config.rowsPerReel[i] < 1 || config.rowsPerReel[i] > 12) {
      errors.push(`第 ${i + 1} 軸列數必須介於 1 到 12 之間`);
    }
  }

  if (config.symbolIds.length === 0) {
    errors.push('至少需要一個 Symbol');
  }

  return { valid: errors.length === 0, errors };
}
```

### 3.4 腳本引擎

腳本引擎處理使用者自訂的 JSON 腳本，是最容易出現格式錯誤的模組。

| 錯誤場景 | 處理方式 | 使用者回饋 |
|----------|----------|------------|
| JSON 語法錯誤 | 解析失敗，提供行列資訊 | Toast：「腳本格式錯誤（第 X 行第 Y 列）」 |
| 未知的 Action 類型 | 跳過該 Action，繼續執行 | Console 警告 |
| 缺少必要參數 | 中止腳本執行 | Toast：「Action "xxx" 缺少必要參數 "yyy"」 |
| 循環 `callScript` | 偵測遞迴深度，超過上限中止 | Toast：「腳本呼叫層數過深，可能存在循環呼叫」 |
| Action 執行時錯誤 | 可配置：中止或跳過 | 依配置顯示 Toast 或 Console |

```typescript
const MAX_CALL_DEPTH = 16;

function executeScript(
  script: Script,
  context: ScriptContext,
  depth = 0
): void {
  if (depth > MAX_CALL_DEPTH) {
    toastStore.error('腳本呼叫層數過深，可能存在循環呼叫。');
    console.error('[ScriptEngine] Max call depth exceeded', { script, depth });
    return;
  }

  for (const action of script.actions) {
    try {
      if (action.type === 'callScript') {
        const sub = context.getScript(action.scriptId);
        if (!sub) {
          toastStore.warning(`找不到腳本 "${action.scriptId}"，已跳過。`);
          continue;
        }
        executeScript(sub, context, depth + 1);
      } else {
        executeAction(action, context);
      }
    } catch (err) {
      console.error(`[ScriptEngine] Action "${action.type}" failed:`, err);
      if (context.onError === 'abort') {
        toastStore.error(`腳本執行失敗：${action.type}`);
        return;
      }
    }
  }
}
```

### 3.5 專案 I/O（匯入 / 匯出）

專案以 `.zip` 格式進行匯入匯出，需要嚴格的檔案驗證流程。

| 錯誤場景 | 處理方式 | 使用者回饋 |
|----------|----------|------------|
| 損壞的 `.zip` 檔 | 中止匯入 | Toast：「專案檔案損壞，無法讀取」 |
| 無效的 Manifest | 中止匯入 | Toast：「專案格式不相容」+ 版本資訊 |
| ZIP 中缺少資源 | 匯入可用部分，列出遺失檔案 | Toast：「以下資源缺失：…」 |
| Schema 版本過新 | 中止匯入 | Toast：「此專案需要更新版本的應用程式」 |
| 匯出失敗 | 保留現有狀態 | Toast：「匯出失敗，請重試」+ 重試按鈕 |

```typescript
async function importProject(file: File): Promise<ImportResult> {
  // 1. 驗證 ZIP 格式
  let zip: JSZip;
  try {
    zip = await JSZip.loadAsync(file);
  } catch {
    toastStore.error('專案檔案損壞，無法讀取。');
    return { success: false };
  }

  // 2. 讀取並驗證 Manifest
  const manifestFile = zip.file('manifest.json');
  if (!manifestFile) {
    toastStore.error('專案缺少 manifest.json，格式不相容。');
    return { success: false };
  }

  const manifest = JSON.parse(await manifestFile.async('string'));

  if (manifest.schemaVersion > CURRENT_SCHEMA_VERSION) {
    toastStore.error(
      `此專案使用 v${manifest.schemaVersion} 格式，` +
      `目前應用程式僅支援 v${CURRENT_SCHEMA_VERSION}，請更新應用程式。`
    );
    return { success: false };
  }

  // 3. 載入資源，記錄遺失項目
  const missing: string[] = [];
  for (const asset of manifest.assets) {
    const assetFile = zip.file(asset.path);
    if (!assetFile) {
      missing.push(asset.path);
      continue;
    }
    await loadAsset(asset, assetFile);
  }

  if (missing.length > 0) {
    toastStore.warning(
      `匯入完成，但以下 ${missing.length} 個資源缺失：\n` +
      missing.map((p) => `• ${p}`).join('\n')
    );
  }

  return { success: true, missing };
}
```

---

## 4. 使用者回饋機制

### 4.1 Toast 通知

Toast 是最主要的使用者回饋管道，統一透過 `toastStore` 發送。

| 類型 | 顏色 | 用途 | 自動關閉 |
|------|------|------|----------|
| **Info** | 藍色 | 一般資訊提示 | 5 秒 |
| **Success** | 綠色 | 操作成功確認 | 5 秒 |
| **Warning** | 琥珀色 | 可恢復的問題提醒 | 手動關閉 |
| **Error** | 紅色 | 需要使用者注意的錯誤 | 手動關閉 |

```typescript
interface Toast {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  message: string;
  autoDismiss: boolean;
  duration?: number;        // 毫秒，僅 autoDismiss = true 時有效
  action?: {
    label: string;          // 按鈕文字，如「重試」
    onClick: () => void;
  };
}
```

**設計考量**：

- Warning 與 Error 不自動消失，確保使用者不會遺漏重要資訊
- 同時最多顯示 5 則 Toast，超過則自動折疊為「還有 N 則通知」
- Toast 可附帶操作按鈕（如「重試」、「查看詳情」）

### 4.2 對話框

用於需要使用者決策的場景，分為兩種：

#### 確認對話框（Confirmation Dialog）

在破壞性操作前要求確認：

- 刪除圖層
- 替換整個專案
- 清空腳本

```
┌─────────────────────────────────────────┐
│  ⚠️ 確認刪除                             │
│                                         │
│  確定要刪除圖層「背景動畫」嗎？            │
│  此操作無法復原。                         │
│                                         │
│              [取消]    [確認刪除]          │
└─────────────────────────────────────────┘
```

#### 錯誤對話框（Error Dialog）

用於致命或阻塞性錯誤，提供詳細資訊與可行動作：

```
┌─────────────────────────────────────────┐
│  ❌ 發生嚴重錯誤                          │
│                                         │
│  WebGL 渲染環境遺失，無法繼續顯示畫面。    │
│                                         │
│  ▶ 詳細資訊                              │
│    Error: WebGL context lost             │
│    at PixiRenderer.render (renderer.ts)  │
│                                         │
│         [重新整理頁面]    [嘗試恢復]       │
└─────────────────────────────────────────┘
```

### 4.3 狀態列提示

應用程式底部的狀態列用於持續性狀態顯示：

| 指示器 | 說明 |
|--------|------|
| 載入進度 | 顯示目前載入的資源名稱與進度百分比 |
| 錯誤指示器（紅點） | 當存在未解決的問題時持續顯示，點擊可展開錯誤列表 |
| FPS 指示器 | 顯示目前幀率，低於 30 fps 時轉為黃色警告 |

---

## 5. Error Boundary（React）

頂層 `ErrorBoundary` 捕捉所有未處理的 React 元件錯誤，避免整個應用程式白屏。

```typescript
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class AppErrorBoundary extends React.Component<
  React.PropsWithChildren,
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('[AppErrorBoundary]', error, errorInfo);
  }

  handleRetry = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): React.ReactNode {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h1>發生錯誤</h1>
          <p>應用程式遇到無法處理的問題。</p>
          <p>您可以嘗試重試，或重新整理頁面。</p>
          <button onClick={this.handleRetry}>重試</button>
          <button onClick={() => window.location.reload()}>
            重新整理頁面
          </button>
          {import.meta.env.DEV && (
            <pre>{this.state.error?.stack}</pre>
          )}
        </div>
      );
    }
    return this.props.children;
  }
}
```

**使用方式**：

```tsx
<AppErrorBoundary>
  <App />
</AppErrorBoundary>
```

**設計考量**：

- 開發模式下顯示完整堆疊追蹤（`import.meta.env.DEV`）
- 生產模式僅顯示友善訊息與操作按鈕
- 「重試」會重置 ErrorBoundary 狀態，重新渲染子元件
- 若重試仍失敗，可選擇「重新整理頁面」

---

## 6. 恢復策略

| 錯誤類型 | 恢復策略 | 說明 |
|----------|----------|------|
| **WebGL Context Lost** | 自動恢復 | 監聽 `webglcontextrestored` 事件，重建整個 Scene Graph 並恢復渲染。 |
| **React 元件崩潰** | ErrorBoundary + 重試 | 透過 ErrorBoundary 攔截錯誤，使用者可點擊「重試」重新掛載元件樹。 |
| **專案檔案損壞** | 載入最後已知正常狀態 | 若 LocalStorage 中存有自動儲存的狀態快照，提供「恢復上次儲存」選項。 |
| **CDN 資源載入失敗** | 指數退避重試 | 以 1s → 2s → 4s → 8s 間隔重試，最多 3 次；全部失敗後提示使用者檢查網路。 |

### CDN 資源重試範例

```typescript
async function fetchWithRetry(
  url: string,
  maxRetries = 3
): Promise<Response> {
  let lastError: Error | null = null;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response;
    } catch (err) {
      lastError = err as Error;
      const delay = Math.pow(2, attempt) * 1000;
      console.warn(
        `[Fetch] Retry ${attempt + 1}/${maxRetries} for ${url} in ${delay}ms`
      );
      await new Promise((r) => setTimeout(r, delay));
    }
  }

  throw lastError;
}
```

---

## 7. 開發者除錯支援

雖然使用者為美術人員，但開發階段仍需提供充足的除錯資訊。

### 7.1 Console 日誌

所有錯誤皆以結構化格式輸出至瀏覽器 Console：

```typescript
// 標準格式：[模組名稱] 錯誤描述
console.error('[SpineLoader] Failed to parse .skel file:', {
  fileName: 'character.skel',
  error: err,
  stack: err.stack,
});
```

### 7.2 錯誤代碼

每個錯誤類型分配唯一的錯誤代碼，便於查詢與回報：

| 代碼範圍 | 模組 | 範例 |
|----------|------|------|
| `ERR_RENDER_1xx` | PixiJS / 渲染層 | `ERR_RENDER_100`：WebGL Context Lost |
| `ERR_SPINE_2xx` | Spine 載入與播放 | `ERR_SPINE_200`：無效的 .skel 檔案 |
| `ERR_REEL_3xx` | 轉輪系統 | `ERR_REEL_300`：無效的盤面配置 |
| `ERR_SCRIPT_4xx` | 腳本引擎 | `ERR_SCRIPT_400`：JSON 語法錯誤 |
| `ERR_IO_5xx` | 專案 I/O | `ERR_IO_500`：ZIP 檔案損壞 |

### 7.3 詳細模式（Verbose Mode）

開發階段可透過 URL 參數或 Console 指令啟用詳細模式：

```typescript
// URL 參數啟用
// ?verbose=true

// Console 指令啟用
window.__SLOT_PREVIEWER_VERBOSE__ = true;
```

啟用後會輸出額外資訊：

- 每個 Action 的執行時間
- Spine 資源載入的詳細過程
- 物件池的建立與回收記錄
- Store 狀態變更記錄

### 7.4 效能警告

自動偵測並在 Console 中提示效能問題：

| 偵測項目 | 警告門檻 | 建議 |
|----------|----------|------|
| FPS 下降 | 連續 3 秒低於 30 fps | 檢查 Scene Graph 複雜度 |
| 記憶體尖峰 | 超過 500 MB | 檢查紋理大小與數量 |
| Spine 實例過多 | 超過 20 個同時播放 | 考慮減少同時播放的動畫數 |

---

## 8. 測試錯誤場景

以下為需要涵蓋的錯誤情境測試清單，確保每個場景都有對應的處理機制：

| # | 場景 | 觸發方式 | 預期行為 |
|---|------|----------|----------|
| 1 | 拖入無效檔案 | 拖入 `.exe` 或 `.doc` 檔案 | 顯示「不支援的檔案格式」Toast |
| 2 | 載入損壞的 `.skel` | 提供內容被截斷的 `.skel` 檔案 | 顯示「Spine 骨骼檔案格式錯誤」Toast |
| 3 | 匯入過舊的專案 | 匯入 Schema v0.1 的 `.zip` | 自動遷移或顯示版本不相容訊息 |
| 4 | 匯入過新的專案 | 匯入 Schema 版本高於目前的 `.zip` | 顯示「需要更新應用程式」Toast |
| 5 | WebGL Context Lost | 透過 DevTools 模擬 | 自動恢復渲染 + 顯示恢復中 Toast |
| 6 | 腳本中引用不存在的圖層 | 腳本 Action 指定不存在的 `layerId` | 跳過該 Action + Console Warning |
| 7 | 腳本循環呼叫 | 腳本 A 呼叫 B、B 呼叫 A | 超過深度上限後中止 + 顯示 Error Toast |
| 8 | JSON 腳本語法錯誤 | 載入含語法錯誤的 JSON | 顯示錯誤位置（行列號）的 Toast |
| 9 | 缺少 Atlas 檔案 | 僅提供 `.skel` 不提供 `.atlas` | 顯示「缺少 Atlas 檔案」Toast |
| 10 | 匯出超大專案 | 專案含大量高解析度紋理 | 匯出失敗時顯示重試按鈕 |
| 11 | Symbol 池耗盡 | 配置極大盤面（如 10×12） | 自動擴充池 + Console Warning |
| 12 | Worker 崩潰 | Spine Worker 拋出未捕捉例外 | 退回主執行緒解析 + Toast 提示 |

---

## 附錄：錯誤處理檢查清單

開發每個新功能時，請確認以下事項：

- [ ] 所有 `async` 函式都有 `try-catch` 或 `.catch()` 處理
- [ ] 使用者可見的錯誤訊息使用繁體中文
- [ ] 錯誤訊息說明「發生了什麼」與「可以做什麼」
- [ ] 所有錯誤都記錄至 Console（含模組標籤與堆疊追蹤）
- [ ] 分配了正確的錯誤代碼
- [ ] 不會因為單一資源載入失敗而阻止整個應用程式運作
- [ ] 破壞性操作前有確認對話框
- [ ] 已加入對應的錯誤場景測試
