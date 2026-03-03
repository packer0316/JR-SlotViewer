# Slot Previewer — Spine 動畫整合規格

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置閱讀**：`01-architecture-overview.md`、`02-module-design.md`
> **適用對象**：開發人員、技術主管、美術（參考用）

---

## 目錄

1. [Spine 整合概覽](#1-spine-整合概覽)
2. [SpineLoader 資源載入](#2-spineloader-資源載入)
3. [SpineActor 動畫控制](#3-spineactor-動畫控制)
4. [Spine Clipping Attachment](#4-spine-clipping-attachment)
5. [Spine 控制面板 UI](#5-spine-控制面板-ui)
6. [效能考量](#6-效能考量)
7. [錯誤處理](#7-錯誤處理)
8. [擴充性](#8-擴充性)

---

## 1. Spine 整合概覽

### 1.1 為何選擇 `@pixi-spine/all-3.8`

| 考量面向 | 說明 |
|----------|------|
| **社群維護的 PixiJS 7 整合** | 專為 PixiJS 7 設計的 Spine 整合套件，原生支援 Spine 3.8.99 格式，API 穩定且相容性良好 |
| **Spine 3.8 完整功能** | 支援 Clipping Attachment、Custom Skin Batching、Mesh Deformation 等進階功能 |
| **渲染效能** | 同材質骨骼自動 Batching 為單次 Draw Call，與 PixiJS 渲染管線深度整合 |
| **版本匹配** | 與 Spine 3.8 Runtime 版本匹配，確保 `.skel` 檔格式相容 |

### 1.2 整合架構

```
┌────────────────────────────────────────────────────────────────┐
│  React UI Layer                                                 │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────────────┐   │
│  │ SpinePanel    │  │ AnimationList  │  │ TrackMixer       │   │
│  │ (控制面板)     │  │ (動畫清單)      │  │ (軌道混合器)      │   │
│  └───────┬───────┘  └───────┬────────┘  └────────┬─────────┘   │
│          │                  │                     │             │
│  ┌───────▼──────────────────▼─────────────────────▼──────────┐  │
│  │              Zustand spineStore                            │  │
│  │  resources[] │ actors{} │ loadRequests │ playbackCommands  │  │
│  └───────┬───────────────────────────────────────────────────┘  │
│          │  訂閱變化                                            │
│  ┌───────▼───────────────────────────────────────────────────┐  │
│  │              Spine Engine Layer                            │  │
│  │  SpineLoader (載入 + 快取) │ SpineActor (播放 + 控制)       │  │
│  └───────┬───────────────────────────────────────────────────┘  │
│          │  操作 Scene Graph                                    │
│  ┌───────▼───────────────────────────────────────────────────┐  │
│  │              PixiJS Application (Canvas)                   │  │
│  │  spine.Spine → Container → Stage                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 資源生命週期

```
拖拽 / URL 輸入
       │
       ▼
  ┌──────────┐     Web Worker      ┌───────────┐
  │ 檔案驗證  │ ──────────────────▶ │ .skel 解析 │
  └──────────┘                     └─────┬─────┘
                                         │ postMessage
                                         ▼
                                   ┌───────────┐
                                   │ SkeletonData│
                                   │  (快取)     │
                                   └─────┬─────┘
                                         │ 建立實例
                                         ▼
                                   ┌───────────┐
                                   │ SpineActor │ ──▶ 加入 Scene Graph
                                   │  (播放控制) │
                                   └─────┬─────┘
                                         │ 使用完畢
                                         ▼
                                   ┌───────────┐
                                   │  Dispose   │ ──▶ 釋放 GPU 資源
                                   └───────────┘
```

| 階段 | 負責模組 | 說明 |
|------|----------|------|
| **Load** | `SpineLoader` | 載入 `.skel` / `.atlas` / `.png`，支援拖拽與 URL 兩種方式 |
| **Cache** | `SpineLoader` | 解析後的 `SkeletonData` 依路徑鍵快取，避免重複載入 |
| **Create** | `SpineLoader` → `SpineActor` | 從快取建立 `SpineActor` 實例，包裝 `spine.Spine` 物件 |
| **Play** | `SpineActor` | 提供播放、暫停、混合、軌道、皮膚等高階 API |
| **Dispose** | `SpineActor` | 銷毀骨骼實例、動畫狀態，釋放 Texture 參考 |

### 1.4 相關原始碼

| 檔案 | 職責 |
|------|------|
| `src/spine/SpineLoader.ts` | 資源載入、快取管理、Web Worker 調度 |
| `src/spine/SpineActor.ts` | 動畫實例高階 API 封裝 |
| `src/spine/spineWorker.ts` | Web Worker：離線解析 `.skel` 二進位資料 |
| `src/store/spineStore.ts` | Zustand Store：資源清單、播放狀態 |
| `src/ui/panels/SpinePanel.tsx` | React 控制面板 UI |

---

## 2. SpineLoader 資源載入

### 2.1 拖拽載入（File System Access API）

美術人員可直接將 Spine 匯出的檔案拖入 Canvas 區域或專用 Drop Zone 來載入動畫資源。

#### 拖拽流程

```
使用者拖入檔案
       │
       ▼
  ┌──────────────┐
  │ 攔截 dragover │ ← preventDefault()
  │ 攔截 drop     │
  └──────┬───────┘
         │ 取得 FileList
         ▼
  ┌──────────────┐
  │ 檔案分類驗證  │ ← 依副檔名分類 .skel / .atlas / .png
  └──────┬───────┘
         │
         ├── 缺少必要檔案 → Toast 錯誤提示
         │
         ▼
  ┌──────────────┐
  │ 讀取檔案內容  │ ← FileReader.readAsArrayBuffer() / readAsText()
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐     postMessage     ┌──────────────┐
  │ 主執行緒      │ ─────────────────▶ │ Web Worker    │
  │ 傳送 .skel   │                     │ 解析 Binary   │
  │ ArrayBuffer  │ ◀───────────────── │ SkeletonData  │
  └──────┬───────┘     onMessage       └──────────────┘
         │
         ▼
  ┌──────────────┐
  │ 建立          │
  │ SpineActor   │ → 加入 Scene Graph → 更新 spineStore
  └──────────────┘
```

#### 檔案驗證規則

| 檔案類型 | 必要性 | 驗證規則 |
|----------|--------|---------|
| `.skel` 或 `.json` | **必要**（擇一） | 骨骼檔，至少一個 |
| `.atlas` | **必要** | 圖集描述檔，至少一個 |
| `.png` | **必要** | 圖集貼圖，至少一個且名稱需與 `.atlas` 中宣告的一致 |

```typescript
function validateSpineFiles(files: File[]): SpineFileSet | null {
  const skelFiles = files.filter((f) =>
    f.name.endsWith('.skel') || f.name.endsWith('.json')
  );
  const atlasFiles = files.filter((f) => f.name.endsWith('.atlas'));
  const pngFiles = files.filter((f) => f.name.endsWith('.png'));

  if (skelFiles.length === 0) {
    showToast('error', '缺少骨骼檔案（.skel 或 .json）');
    return null;
  }
  if (atlasFiles.length === 0) {
    showToast('error', '缺少圖集檔案（.atlas）');
    return null;
  }
  if (pngFiles.length === 0) {
    showToast('error', '缺少貼圖檔案（.png）');
    return null;
  }

  return {
    skelFile: skelFiles[0],
    atlasFile: atlasFiles[0],
    pngFiles,
    name: skelFiles[0].name.replace(/\.(skel|json)$/, ''),
  };
}
```

#### Drop Zone 事件處理

```typescript
function setupDropZone(element: HTMLElement): void {
  element.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.dataTransfer!.dropEffect = 'copy';
    element.classList.add('drop-active');
  });

  element.addEventListener('dragleave', () => {
    element.classList.remove('drop-active');
  });

  element.addEventListener('drop', async (e) => {
    e.preventDefault();
    element.classList.remove('drop-active');

    const files = Array.from(e.dataTransfer!.files);
    const fileSet = validateSpineFiles(files);
    if (!fileSet) return;

    try {
      const actor = await SpineLoader.loadFromFiles(fileSet);
      spineStore.getState().addResource(actor.getResource());
    } catch (err) {
      showToast('error', `載入失敗：${(err as Error).message}`);
    }
  });
}
```

### 2.2 URL 載入

從 CDN 或本地開發伺服器載入 Spine 資源，適用於已部署至伺服器的素材。

```typescript
const actor = await SpineLoader.load({
  skelPath: 'https://cdn.example.com/spine/hero.skel',
  atlasPath: 'https://cdn.example.com/spine/hero.atlas',
  name: 'hero',
});
```

#### URL 解析規則

| 情境 | 輸入 | 解析結果 |
|------|------|---------|
| 絕對 URL | `https://cdn.example.com/hero.skel` | 直接使用 |
| 相對路徑 | `./assets/hero.skel` | 以 `window.location.origin` 為基底拼接 |
| 僅指定 `.skel` | `hero.skel` | 自動推斷 `.atlas` 路徑為 `hero.atlas`（同目錄） |

```typescript
function resolveSpinePaths(config: SpineLoadConfig): { skelUrl: string; atlasUrl: string } {
  const skelUrl = new URL(config.skelPath, window.location.origin).href;
  const atlasUrl = config.atlasPath
    ? new URL(config.atlasPath, window.location.origin).href
    : skelUrl.replace(/\.(skel|json)$/, '.atlas');

  return { skelUrl, atlasUrl };
}
```

### 2.3 快取機制

載入後的 `SkeletonData` 以**路徑鍵**快取。當多個 `SpineActor` 需要使用同一骨骼時，共用同一份 `SkeletonData`，各自持有獨立的 `AnimationState`。

```
┌─────────────────────────────────────────────────┐
│  SpineLoader Cache                               │
│                                                  │
│  Map<string, CacheEntry>                         │
│                                                  │
│  "hero.skel"    → { skeletonData, atlasData }    │
│  "scatter.skel" → { skeletonData, atlasData }    │
│  "wild.skel"    → { skeletonData, atlasData }    │
└──────────────────────┬──────────────────────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   SpineActor A   SpineActor B   SpineActor C
   (hero 實例1)   (hero 實例2)   (scatter 實例)
   獨立 AnimState 獨立 AnimState  獨立 AnimState
```

```typescript
interface CacheEntry {
  skeletonData: SkeletonData;
  atlas: TextureAtlas;
  refCount: number;
}

class SpineLoaderCache {
  private cache = new Map<string, CacheEntry>();

  get(key: string): CacheEntry | undefined {
    return this.cache.get(key);
  }

  set(key: string, entry: Omit<CacheEntry, 'refCount'>): void {
    this.cache.set(key, { ...entry, refCount: 1 });
  }

  addRef(key: string): void {
    const entry = this.cache.get(key);
    if (entry) entry.refCount++;
  }

  release(key: string): void {
    const entry = this.cache.get(key);
    if (!entry) return;

    entry.refCount--;
    if (entry.refCount <= 0) {
      entry.atlas.dispose();
      this.cache.delete(key);
    }
  }

  clear(): void {
    this.cache.forEach((entry) => entry.atlas.dispose());
    this.cache.clear();
  }
}
```

| 操作 | 說明 |
|------|------|
| `get(key)` | 查詢快取，若命中則直接回傳已解析的 `SkeletonData` |
| `addRef(key)` | 增加參考計數，用於多實例共用場景 |
| `release(key)` | 減少參考計數，歸零時自動釋放 `TextureAtlas` 資源 |
| `clear()` | 清除全部快取，釋放所有 GPU 資源（專案切換時呼叫） |

### 2.4 Web Worker 解析

`.skel` 為二進位格式，解析大型骨骼檔案可能耗時 50–200ms。為避免阻塞主執行緒造成畫面卡頓，將解析作業交由 Web Worker 離線執行。

#### 架構

```
┌──────────────────┐                     ┌──────────────────┐
│   主執行緒         │                     │   Web Worker      │
│                  │                     │   (spineWorker)   │
│                  │   postMessage       │                   │
│  SpineLoader ────┼──────────────────▶ │  parseSkel()      │
│                  │   { type, buffer }  │  ↓                │
│                  │                     │  SkeletonBinary   │
│                  │                     │  .readSkeletonData│
│                  │   onMessage         │  ↓                │
│  resolve(data) ◀─┼──────────────────── │  序列化結果回傳    │
│                  │   { type, result }  │                   │
└──────────────────┘                     └──────────────────┘
```

#### Worker 端實作

```typescript
// src/spine/spineWorker.ts
import { SkeletonBinary, AtlasAttachmentLoader, TextureAtlas } from '@pixi-spine/runtime-3.8';

self.onmessage = (e: MessageEvent<WorkerMessage>) => {
  const { type, payload } = e.data;

  switch (type) {
    case 'parseSkel': {
      try {
        const { skelBuffer, atlasText } = payload;

        const atlas = new TextureAtlas(atlasText);
        const attachmentLoader = new AtlasAttachmentLoader(atlas);
        const binary = new SkeletonBinary(attachmentLoader);
        binary.scale = 1;

        const skeletonData = binary.readSkeletonData(new Uint8Array(skelBuffer));

        const result = {
          animations: skeletonData.animations.map((a) => a.name),
          skins: skeletonData.skins.map((s) => s.name),
          slots: skeletonData.slots.map((s) => s.name),
          width: skeletonData.width,
          height: skeletonData.height,
        };

        self.postMessage({ type: 'parseSkel:done', result });
      } catch (err) {
        self.postMessage({ type: 'parseSkel:error', error: (err as Error).message });
      }
      break;
    }
  }
};
```

#### 主執行緒端呼叫

```typescript
class SpineLoader {
  private static worker = new Worker(
    new URL('./spineWorker.ts', import.meta.url),
    { type: 'module' }
  );

  private static workerParse(
    skelBuffer: ArrayBuffer,
    atlasText: string
  ): Promise<WorkerParseResult> {
    return new Promise((resolve, reject) => {
      const handler = (e: MessageEvent) => {
        if (e.data.type === 'parseSkel:done') {
          SpineLoader.worker.removeEventListener('message', handler);
          resolve(e.data.result);
        } else if (e.data.type === 'parseSkel:error') {
          SpineLoader.worker.removeEventListener('message', handler);
          reject(new Error(e.data.error));
        }
      };

      SpineLoader.worker.addEventListener('message', handler);
      SpineLoader.worker.postMessage(
        { type: 'parseSkel', payload: { skelBuffer, atlasText } },
        [skelBuffer] // Transferable，零拷貝傳輸
      );
    });
  }
}
```

> **效能關鍵**：使用 `Transferable` 物件（`[skelBuffer]`）傳送 `ArrayBuffer`，避免資料拷貝。傳輸後主執行緒的 `skelBuffer` 會被清空（neutered），Worker 直接持有記憶體。

### 2.5 SpineLoader 完整 API

```typescript
class SpineLoader {
  /**
   * 從 URL 載入 Spine 資源。
   * 先檢查快取，命中則直接建立 SpineActor；否則下載並解析。
   */
  static async load(config: SpineLoadConfig): Promise<SpineActor>

  /**
   * 從本地檔案載入 Spine 資源（拖拽匯入）。
   * 使用 FileReader 讀取後交由 Web Worker 解析。
   */
  static async loadFromFiles(files: SpineFileSet): Promise<SpineActor>

  /** 取得目前的快取 Map（唯讀） */
  static getCache(): Map<string, SkeletonData>

  /** 清除所有快取並釋放關聯的 GPU 資源 */
  static clearCache(): void

  /** 從快取中移除指定鍵的資源 */
  static removeFromCache(key: string): void
}

interface SpineLoadConfig {
  skelPath: string;       // .skel 或 .json 骨骼檔路徑
  atlasPath: string;      // .atlas 圖集路徑
  name?: string;          // 資源顯示名稱（未提供時取檔名）
}

interface SpineFileSet {
  skelFile: File;         // 骨骼檔
  atlasFile: File;        // 圖集檔
  pngFiles: File[];       // 貼圖檔案陣列
  name?: string;          // 資源顯示名稱
}
```

#### 載入事件

| 事件名稱 | Payload | 說明 |
|----------|---------|------|
| `spine:loaded` | `{ id: string; actor: SpineActor }` | 資源載入完成 |
| `spine:loadError` | `{ id: string; error: Error }` | 載入失敗 |
| `spine:cacheHit` | `{ key: string }` | 快取命中（可用於 debug） |
| `spine:cacheClear` | — | 快取已清除 |

---

## 3. SpineActor 動畫控制

`SpineActor` 封裝單一 `spine.Spine` 實例的高階 API，隔離底層 `@pixi-spine/all-3.8` 的實作細節。每個 `SpineActor` 擁有獨立的 `AnimationState`，可自主控制播放行為。

### 3.1 基本播放

```typescript
const actor = await SpineLoader.load({
  skelPath: '/assets/hero.skel',
  atlasPath: '/assets/hero.atlas',
});

actor.play('idle', true);      // 循環播放 idle 動畫
actor.setSpeed(1.5);           // 1.5 倍速

actor.pause();                 // 暫停
actor.resume();                // 恢復
```

#### 播放速度範圍

| 速度 | 效果 |
|------|------|
| `0.1` | 慢動作（適合逐幀檢視） |
| `1.0` | 正常速度 |
| `2.0` | 兩倍速 |
| `4.0` | 最大速度（快速預覽） |

```typescript
actor.setSpeed(speed: number): void {
  const clamped = Math.max(0.1, Math.min(4.0, speed));
  this.spine.state.timeScale = clamped;
}
```

### 3.2 多 Track 疊加

Spine 的軌道（Track）系統允許在同一骨骼上疊加多個動畫。典型應用：

```
Track 0: walk         ← 下半身行走動畫（基底層）
Track 1: wave         ← 上半身揮手動畫（疊加層）
Track 2: blink        ← 表情眨眼（疊加層）
```

```typescript
// 在 Track 0 播放行走
actor.setTrack(0, 'walk', true);

// 在 Track 1 疊加揮手（alpha 0.8 與下層混合）
actor.setTrack(1, 'wave', true);
actor.setTrackAlpha(1, 0.8);

// 在 Track 2 疊加眨眼
actor.setTrack(2, 'blink', true);
actor.setTrackAlpha(2, 1.0);
```

#### Track 優先規則

| 規則 | 說明 |
|------|------|
| 較高 Track 覆蓋較低 Track | Track 1 的骨骼變換會覆蓋 Track 0 中相同骨骼的變換 |
| Alpha 控制混合程度 | `alpha = 0.5` 時，該 Track 的影響力為 50%，與下方 Track 混合 |
| 互不干涉的骨骼可完全獨立 | 若 Track 1 只影響手臂骨骼，Track 0 的腿部動畫不受影響 |

### 3.3 動畫過渡（Mix）

從一個動畫平滑過渡到另一個動畫，避免突兀的切換：

```typescript
// 從目前動畫淡入 'run'，過渡時長 0.3 秒
actor.mixTo('run', 0.3);
```

#### 過渡機制圖示

```
時間軸 ─────────────────────────────────────────▶
         │← 'idle' ──▶│←── mix ──▶│← 'run' ──▶
                       │ 0.3s 淡入淡出│
         alpha:  1.0    ↘ 0.5        ↗  1.0
```

```typescript
mixTo(animName: string, mixDuration: number): void {
  this.spine.state.setAnimation(0, animName, true);
  this.spine.state.data.defaultMix = mixDuration;
}
```

也可以針對特定動畫對設定預設的 mix 時長：

```typescript
// idle → run 過渡 0.3 秒，run → idle 過渡 0.5 秒
actor.setMixDuration('idle', 'run', 0.3);
actor.setMixDuration('run', 'idle', 0.5);
```

### 3.4 皮膚系統

Spine 的 Skin 系統允許在運行時切換角色外觀：

```typescript
// 取得所有可用皮膚
const skins = actor.getSkinNames();
// => ['default', 'golden', 'diamond', 'ruby']

// 切換至黃金皮膚
actor.setSkin('golden');
```

#### 運行時皮膚組合

Spine 支援合併多個 Skin，在 Slot 遊戲中可用於組合不同的 Symbol 外觀：

```typescript
actor.combineSkins(['base_body', 'golden_armor', 'fire_weapon']);
```

```typescript
combineSkins(skinNames: string[]): void {
  const combined = new Skin('combined');
  for (const name of skinNames) {
    const skin = this.spine.skeleton.data.findSkin(name);
    if (skin) combined.addSkin(skin);
  }
  this.spine.skeleton.setSkin(combined);
  this.spine.skeleton.setSlotsToSetupPose();
}
```

### 3.5 Attachment 替換

動態替換指定 Slot 中的 Attachment，適用於 Symbol 變體切換：

```typescript
// 取得所有 Slot 名稱
const slots = actor.getSlotNames();
// => ['body', 'head', 'arm_left', 'arm_right', 'symbol_display']

// 將 'symbol_display' Slot 的附件切換為 'wild_symbol'
actor.setAttachment('symbol_display', 'wild_symbol');

// 清除 Slot 上的附件（隱藏）
actor.setAttachment('symbol_display', null);
```

| 方法 | 說明 |
|------|------|
| `setAttachment(slot, attachment)` | 切換指定 Slot 的附件 |
| `setAttachment(slot, null)` | 清除附件（Slot 變為空白） |
| `getSlotNames()` | 取得所有 Slot 名稱 |

### 3.6 事件回呼

Spine 動畫中可內嵌自訂事件（如音效觸發點、粒子發射點）。`SpineActor` 提供統一的事件訂閱介面：

```typescript
// 動畫播放完成回呼
const unsubComplete = actor.onComplete(() => {
  console.log('動畫播放完成');
});

// 監聽 Spine 自訂事件
const unsubEvent = actor.onEvent('footstep', (event) => {
  console.log(`觸發事件: ${event.data.name}, 數值: ${event.intValue}`);
  playSound('footstep.mp3');
});

// 取消訂閱
unsubComplete();
unsubEvent();
```

#### 事件流轉

```
Spine AnimationState
       │ listener.complete / listener.event
       ▼
  SpineActor 內部 handler
       │
       ├── 觸發已註冊的 callback
       │
       └── 轉發至 EventBus → 可供 ScriptRunner 消費
           事件名稱格式：'spine:event:{actorId}:{eventName}'
```

### 3.7 進度查詢

```typescript
// 取得當前動畫名稱
actor.getCurrentAnimation();  // => 'idle'

// 取得動畫總時長（秒）
actor.getDuration('win');     // => 2.5

// 取得當前播放進度（0–1）
actor.getProgress();          // => 0.73
```

### 3.8 SpineActor 完整 API

```typescript
class SpineActor {
  // ── 播放控制 ──────────────────────────────────────────────
  play(animName: string, loop?: boolean): void
  mixTo(animName: string, mixDuration: number): void
  setTrack(trackIndex: number, animName: string, loop?: boolean): void
  setTrackAlpha(trackIndex: number, alpha: number): void
  setMixDuration(fromAnim: string, toAnim: string, duration: number): void
  pause(): void
  resume(): void
  setSpeed(speed: number): void

  // ── 皮膚與附件 ────────────────────────────────────────────
  setSkin(skinName: string): void
  combineSkins(skinNames: string[]): void
  setAttachment(slotName: string, attachmentName: string | null): void

  // ── 查詢 ─────────────────────────────────────────────────
  getAnimationNames(): string[]
  getSkinNames(): string[]
  getSlotNames(): string[]
  getCurrentAnimation(): string | null
  getDuration(animName: string): number
  getProgress(): number            // 0–1 當前動畫進度
  getResource(): SpineResource     // 取得關聯的資源描述

  // ── 事件 ─────────────────────────────────────────────────
  onComplete(callback: () => void): () => void          // 回傳取消訂閱函式
  onEvent(eventName: string, callback: (event: SpineEvent) => void): () => void

  // ── 生命週期 ──────────────────────────────────────────────
  dispose(): void
}
```

---

## 4. Spine Clipping Attachment

Spine 原生支援 **Clipping Attachment**——在骨骼編輯器中定義裁切區域，運行時自動對後續 Slot 進行遮罩裁切。

### 4.1 Clipping 在 Slot Previewer 中的角色

在 Slot 遊戲預覽中，Clipping Attachment 常見於以下場景：

| 場景 | 說明 |
|------|------|
| Symbol 內部遮罩 | 某些 Symbol 需要圓角、異形裁切 |
| 角色動畫遮罩 | 角色身體部位的層級遮罩 |
| 特效區域限制 | 粒子或光效僅在指定區域顯示 |

### 4.2 Spine Clipping 與 PixiJS Mask 的協作

```
┌─────────────────────────────────────────────────────┐
│  Spine Skeleton                                      │
│                                                      │
│  Slot 0: background                                  │
│  Slot 1: [Clipping Attachment] ← Spine 原生裁切       │
│  Slot 2: body       ← 受 Slot 1 裁切                 │
│  Slot 3: overlay    ← 受 Slot 1 裁切                 │
│  Slot 4: [End Slot] ← 裁切結束                        │
│  Slot 5: foreground ← 不受裁切影響                    │
│                                                      │
└──────────────────────┬──────────────────────────────┘
                       │ pixi-spine 自動處理
                       ▼
┌─────────────────────────────────────────────────────┐
│  PixiJS Container (spine.Spine)                      │
│                                                      │
│  Spine 內部 Clipping 已由 pixi-spine runtime 處理     │
│  → 透過 WebGL Stencil Buffer 實現                     │
│                                                      │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  LayerManager / ClipManager                          │
│                                                      │
│  圖層級遮罩（轉輪視窗等）獨立套用於 SpineActor 的      │
│  父容器，與 Spine 內部 Clipping 互不衝突               │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 4.3 注意事項

| 項目 | 說明 |
|------|------|
| **Clipping 與 Batching** | Clipping Attachment 會打斷 Spine 內部的 Batching，每個 Clipping 區域產生額外 Draw Call |
| **巢狀遮罩** | Spine Clipping + PixiJS Mask 可同時存在，Stencil Buffer 支援多層巢狀 |
| **效能建議** | 若 Clipping 僅為簡單矩形，建議改用 PixiJS Graphics Mask 取代（效能更佳） |
| **自動偵測** | `ClipManager` 在初始化 `SpineActor` 時自動偵測骨骼是否包含 Clipping Attachment，並在 Debug 面板中標示 |

---

## 5. Spine 控制面板 UI

Spine 控制面板為美術人員提供直覺化的動畫操作介面，所有操作透過 `spineStore` 間接驅動 `SpineActor`。

### 5.1 面板佈局

```
┌──────────────────────────────────────────────┐
│  Spine 控制面板                                │
├──────────────────────────────────────────────┤
│                                               │
│  ┌─ 動畫清單 ─────────────────────────────┐   │
│  │  ● idle           [▶ 播放] [↻ 循環]    │   │
│  │  ○ walk                                │   │
│  │  ○ run                                 │   │
│  │  ○ attack                              │   │
│  │  ○ win                                 │   │
│  │  ○ bonus_start                         │   │
│  │  ○ bonus_loop                          │   │
│  └────────────────────────────────────────┘   │
│                                               │
│  ┌─ 軌道混合器 ──────────────────────────┐    │
│  │  Track 0: idle       α ━━━━━━━○ 1.0   │   │
│  │  Track 1: (none)     α ━━○━━━━  0.5   │   │
│  │  Track 2: (none)     α ━━━━━━━○ 1.0   │   │
│  │                    [+ 新增軌道]         │   │
│  └────────────────────────────────────────┘   │
│                                               │
│  ┌─ 皮膚切換 ──────────────────────────┐      │
│  │  ▼ [default        ▾]               │      │
│  │    ├ golden                          │      │
│  │    ├ diamond                         │      │
│  │    └ ruby                            │      │
│  └────────────────────────────────────┘       │
│                                               │
│  ┌─ 時間軸 ──────────────────────────────┐    │
│  │  0:00 ━━━━━━●━━━━━━━━━━━ 2:30        │    │
│  │       ▶ 播放  ⏸ 暫停  ■ 停止          │    │
│  └────────────────────────────────────────┘   │
│                                               │
│  ┌─ 速度控制 ──────────────────────────┐      │
│  │  0.1x ━━━━━━━━━●━━━━━━━ 4.0x       │      │
│  │             [1.2x]                   │      │
│  └────────────────────────────────────┘       │
│                                               │
│  ┌─ 事件監視器 ──────────────────────────┐    │
│  │  [0:01.230] footstep  int: 1          │    │
│  │  [0:01.890] footstep  int: 2          │    │
│  │  [0:02.100] attack_hit                │    │
│  │  [0:02.500] sound  str: "clash.mp3"   │    │
│  │                         [清除紀錄]     │    │
│  └────────────────────────────────────────┘   │
│                                               │
└──────────────────────────────────────────────┘
```

### 5.2 各區塊功能說明

#### 動畫清單

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 瀏覽動畫 | 捲動列表 | 顯示骨骼內所有可用動畫名稱 |
| 點擊播放 | 單擊動畫名稱 | 立即在 Track 0 播放該動畫 |
| 循環切換 | 點擊 `↻` 圖示 | 切換循環 / 單次播放模式 |
| 搜尋過濾 | 上方搜尋框 | 輸入關鍵字即時過濾動畫名稱 |

#### 軌道混合器

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 指定 Track 動畫 | 下拉選單 | 為每個 Track 獨立選擇動畫 |
| Alpha 調整 | 拖曳滑桿 | 調整 Track 的混合權重（0–1） |
| 新增軌道 | 點擊 `+ 新增軌道` | 新增 Track（最多 4 個） |
| 清除軌道 | 右鍵選單 | 移除該 Track 的動畫 |

#### 皮膚切換

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 選擇皮膚 | 下拉選單 | 即時切換整體皮膚 |
| 預覽 | 自動 | 選擇後立即反映至 Canvas |

#### 時間軸拖曳器

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 顯示進度 | 進度條 | 即時顯示當前動畫的播放位置 |
| 拖曳跳轉 | 拖曳滑桿 | 拖到任意時間點（Scrub） |
| 顯示時間 | 左右標籤 | 顯示當前時間 / 總時長 |

#### 速度控制

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 滑桿調速 | 拖曳滑桿 | 0.1x – 4.0x 連續調整 |
| 數值顯示 | 中央標籤 | 即時顯示當前倍速值 |
| 快捷倍速 | 預設按鈕 | 0.5x / 1x / 2x 快捷切換 |

#### 事件監視器

| 功能 | 互動方式 | 說明 |
|------|---------|------|
| 即時紀錄 | 自動 | 列出 Spine 事件的觸發時間、名稱與資料 |
| 清除 | 點擊 `清除紀錄` | 清空事件紀錄列表 |
| 自動捲動 | 自動 | 新事件出現時自動捲至底部 |

### 5.3 面板與 Store 的資料流

```
使用者點擊「播放 walk」
       │
       ▼
  SpinePanel (React)
       │ dispatch action
       ▼
  spineStore.updateActor(id, { currentAnimation: 'walk', loop: true })
       │
       ▼
  SpineActor 訂閱 store → 呼叫 this.play('walk', true)
       │
       ▼
  spine.Spine.state.setAnimation(0, 'walk', true)
       │
       ▼
  PixiJS 渲染下一幀 → Canvas 顯示新動畫
```

---

## 6. 效能考量

### 6.1 Spine Batching

`@pixi-spine/all-3.8` 內建 Batching 機制：使用**相同材質**（Atlas Texture）的多個骨骼可合併為單次 Draw Call。

```
┌────────────────────────────────────────────────────┐
│  無 Batching（3 個獨立 Draw Call）                    │
│                                                     │
│  DC 1: hero.png    → draw hero skeleton             │
│  DC 2: scatter.png → draw scatter skeleton           │
│  DC 3: hero.png    → draw hero skeleton (實例 2)     │
│        ↑ 重複的 Texture！                             │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  有 Batching（2 個 Draw Call）                        │
│                                                     │
│  DC 1: hero.png    → draw hero 實例 1 + 實例 2       │
│  DC 2: scatter.png → draw scatter skeleton           │
│        ↑ 相同 Texture 合併！                          │
└────────────────────────────────────────────────────┘
```

#### Batching 中斷因素

| 因素 | 說明 | 對策 |
|------|------|------|
| 不同 Atlas 貼圖 | 不同 `.png` 無法合併 | 將相關 Symbol 打包至共用 Atlas |
| Clipping Attachment | 裁切會打斷批次 | 僅在必要時使用 Spine Clipping |
| Blend Mode 切換 | 不同混合模式無法合併 | 盡量讓相鄰骨骼使用相同 Blend Mode |

### 6.2 Atlas 最佳化

建議美術人員將所有 Symbol 的 Spine 資源打包至**共用 Atlas**，以最大化 Batching 效益：

```
❌ 每個 Symbol 獨立 Atlas（4 張貼圖 = 4 次 Draw Call）
   sym_h1.png (512x512)
   sym_h2.png (512x512)
   sym_h3.png (512x512)
   sym_h4.png (512x512)

✅ 共用 Atlas（1 張貼圖 = 1 次 Draw Call）
   symbols_atlas.png (2048x2048)
   ├── H1 區域 (0, 0, 512, 512)
   ├── H2 區域 (512, 0, 512, 512)
   ├── H3 區域 (0, 512, 512, 512)
   └── H4 區域 (512, 512, 512, 512)
```

### 6.3 實例共用

多個 `SpineActor` 可共用同一份 `SkeletonData`（透過快取機制），但各自持有獨立的 `Skeleton` 與 `AnimationState`：

| 共用 | 獨立 |
|------|------|
| `SkeletonData`（骨骼結構定義） | `Skeleton`（實例狀態） |
| `TextureAtlas`（貼圖資源） | `AnimationState`（播放狀態） |

```typescript
const actor1 = await SpineLoader.load({ skelPath: 'hero.skel', atlasPath: 'hero.atlas' });
const actor2 = await SpineLoader.load({ skelPath: 'hero.skel', atlasPath: 'hero.atlas' });
// actor1 與 actor2 共用 SkeletonData，但各自獨立播放不同動畫
actor1.play('idle', true);
actor2.play('attack', false);
```

### 6.4 資源釋放

正確的銷毀順序，避免記憶體洩漏：

```typescript
class SpineActor {
  dispose(): void {
    // 1. 停止所有動畫
    this.spine.state.clearTracks();

    // 2. 移除所有事件監聽
    this.spine.state.removeListener(this.listener);

    // 3. 從 Scene Graph 移除
    this.spine.parent?.removeChild(this.spine);

    // 4. 銷毀 spine.Spine 物件
    this.spine.destroy();

    // 5. 從快取中釋放參考計數
    SpineLoader.releaseCache(this.cacheKey);
  }
}
```

### 6.5 效能指標參考

| 指標 | 目標值 | 量測方式 |
|------|--------|---------|
| 單一 Spine 骨骼載入時間 | < 100ms | `Performance.now()` 差值 |
| 10 個同 Atlas 骨骼 Draw Call | ≤ 2 | PixiJS DevTools 或 Spector.js |
| 60fps 下最大同時骨骼數 | ≥ 20 | Ticker `deltaTime` 穩定度 |
| 記憶體（10 個骨骼） | < 50MB | Chrome DevTools Memory |

---

## 7. 錯誤處理

### 7.1 錯誤類型與處理策略

```
┌────────────────────────────────────────────────────────────┐
│                    錯誤處理流程                               │
│                                                             │
│  載入階段錯誤                                                │
│  ├── 檔案類型不符 → Toast 提示「缺少 .skel / .atlas / .png」  │
│  ├── .skel 版本不相容 → Toast 提示「不支援的 Spine 版本」      │
│  ├── Atlas 解析失敗 → Toast 提示「圖集格式錯誤」              │
│  ├── 貼圖缺失 → 使用佔位貼圖 + Console 警告                   │
│  └── 網路錯誤（URL 載入）→ Toast 提示 + 重試按鈕             │
│                                                             │
│  播放階段錯誤                                                │
│  ├── 無效動畫名稱 → Console 警告，忽略操作                     │
│  ├── 無效皮膚名稱 → Console 警告，保持原皮膚                   │
│  ├── 無效 Slot 名稱 → Console 警告，忽略操作                  │
│  └── Track 超出範圍 → Console 警告，忽略操作                  │
│                                                             │
│  Worker 錯誤                                                │
│  ├── Worker 初始化失敗 → 降級為主執行緒解析                    │
│  └── Worker 解析崩潰 → 降級為主執行緒解析 + Console 警告      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 7.2 檔案驗證

```typescript
function validateSkelVersion(buffer: ArrayBuffer): boolean {
  const view = new DataView(buffer);
  const version = readSpineVersion(view);
  const [major] = version.split('.').map(Number);

  if (major !== 3) {
    showToast('error', `不支援的 Spine 版本 ${version}，需要 3.8.x`);
    return false;
  }

  return true;
}
```

### 7.3 缺失貼圖處理

當 `.atlas` 中引用的 `.png` 檔案未被提供時，使用**棋盤格佔位貼圖**代替，並在 Console 輸出警告：

```typescript
function createPlaceholderTexture(width: number, height: number): Texture {
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d')!;

  const size = 16;
  for (let y = 0; y < height; y += size) {
    for (let x = 0; x < width; x += size) {
      ctx.fillStyle = (x + y) % (size * 2) === 0 ? '#ff00ff' : '#000000';
      ctx.fillRect(x, y, size, size);
    }
  }

  console.warn(`[SpineLoader] 缺少貼圖，已使用佔位貼圖替代`);
  return Texture.from(canvas);
}
```

### 7.4 Worker 降級

當 Web Worker 無法正常運作時（如瀏覽器不支援 module Worker），自動降級至主執行緒解析：

```typescript
class SpineLoader {
  private static worker: Worker | null = null;

  private static getWorker(): Worker | null {
    if (this.worker) return this.worker;

    try {
      this.worker = new Worker(
        new URL('./spineWorker.ts', import.meta.url),
        { type: 'module' }
      );
      return this.worker;
    } catch {
      console.warn('[SpineLoader] Web Worker 不可用，降級為主執行緒解析');
      return null;
    }
  }

  private static async parseSkel(
    skelBuffer: ArrayBuffer,
    atlasText: string
  ): Promise<WorkerParseResult> {
    const worker = this.getWorker();

    if (worker) {
      return this.workerParse(skelBuffer, atlasText);
    }

    // 降級：主執行緒解析
    return this.mainThreadParse(skelBuffer, atlasText);
  }
}
```

### 7.5 播放階段的安全守衛

```typescript
class SpineActor {
  play(animName: string, loop = true): void {
    if (!this.spine.skeleton.data.findAnimation(animName)) {
      console.warn(`[SpineActor] 動畫 "${animName}" 不存在，已忽略`);
      return;
    }
    this.spine.state.setAnimation(0, animName, loop);
  }

  setSkin(skinName: string): void {
    if (!this.spine.skeleton.data.findSkin(skinName)) {
      console.warn(`[SpineActor] 皮膚 "${skinName}" 不存在，已忽略`);
      return;
    }
    this.spine.skeleton.setSkinByName(skinName);
    this.spine.skeleton.setSlotsToSetupPose();
  }

  setAttachment(slotName: string, attachmentName: string | null): void {
    const slotIndex = this.spine.skeleton.findSlotIndex(slotName);
    if (slotIndex === -1) {
      console.warn(`[SpineActor] Slot "${slotName}" 不存在，已忽略`);
      return;
    }
    this.spine.skeleton.setAttachment(slotName, attachmentName);
  }
}
```

---

## 8. 擴充性

### 8.1 自訂 Spine 事件 → EventBus 整合

`SpineActor` 會自動將 Spine 內部事件轉發至全域 `EventBus`，使 `ScriptRunner` 可監聽並觸發後續動作：

```typescript
class SpineActor {
  private bridgeEventsToEventBus(eventBus: EventBus): void {
    this.spine.state.addListener({
      event: (_entry, event) => {
        eventBus.emit(`spine:event:${this.id}:${event.data.name}`, {
          actorId: this.id,
          eventName: event.data.name,
          intValue: event.intValue,
          floatValue: event.floatValue,
          stringValue: event.stringValue,
        });
      },
      complete: (_entry) => {
        eventBus.emit(`spine:complete:${this.id}`, {
          actorId: this.id,
          animation: _entry.animation?.name,
        });
      },
    });
  }
}
```

#### ScriptRunner 腳本中的事件監聽

```json
{
  "steps": [
    { "action": "playSpine", "actor": "hero", "anim": "attack", "loop": false },
    { "action": "waitEvent", "event": "spine:event:hero:attack_hit" },
    { "action": "playSpine", "actor": "impact_fx", "anim": "explode", "loop": false }
  ]
}
```

### 8.2 Spine Runtime 版本隔離

`SpineActor` 介面作為抽象層，將底層 Spine Runtime 的 API 細節封裝在內部。當未來需要升級至新版 Spine Runtime 時，僅需修改 `SpineActor` 與 `SpineLoader` 的內部實作，對外介面保持不變：

```
┌────────────────────────────────────────────────┐
│  外部消費者（UI、ScriptRunner、其他 Manager）     │
│                                                 │
│  呼叫: actor.play('idle', true)                  │
│  呼叫: actor.setSkin('golden')                   │
│  呼叫: actor.onEvent('hit', handler)             │
│                                                 │
│  ──── 穩定介面，不因 Runtime 版本變更而修改 ─────  │
└──────────────────────┬──────────────────────────┘
                       │
            SpineActor 抽象層
                       │
┌──────────────────────▼──────────────────────────┐
│  @pixi-spine/all-3.8 v4.0.x                     │
│  （內部實作，可替換為新版本）                       │
│                                                  │
│  spine.Spine / AnimationState / Skeleton          │
└──────────────────────────────────────────────────┘
```

### 8.3 多 Runtime 版本支援（進階）

若未來需要同時支援多個 Spine Runtime 版本（例如 3.8 與 4.x 的 `.skel` 檔案），可透過 **Factory Pattern** 根據檔案版本選擇對應的 Runtime：

```typescript
interface SpineRuntimeAdapter {
  parseSkel(buffer: ArrayBuffer, atlasText: string): SkeletonData;
  createSpine(skeletonData: SkeletonData): DisplayObject;
}

class SpineRuntimeFactory {
  private static adapters = new Map<number, SpineRuntimeAdapter>();

  static register(majorVersion: number, adapter: SpineRuntimeAdapter): void {
    this.adapters.set(majorVersion, adapter);
  }

  static getAdapter(majorVersion: number): SpineRuntimeAdapter {
    const adapter = this.adapters.get(majorVersion);
    if (!adapter) {
      throw new Error(`不支援的 Spine Runtime 版本: ${majorVersion}`);
    }
    return adapter;
  }
}

// 註冊
SpineRuntimeFactory.register(3, new SpineV3Adapter());
// 未來可擴充
// SpineRuntimeFactory.register(4, new SpineV4Adapter());
```

### 8.4 擴充點總覽

| 擴充點 | 方式 | 影響範圍 |
|--------|------|---------|
| 新 Spine Runtime 版本 | 修改 `SpineActor` / `SpineLoader` 內部 | 對外介面不變 |
| 新事件類型 | `SpineActor` 自動轉發至 `EventBus` | `ScriptRunner` 可直接消費 |
| 多版本 `.skel` | `SpineRuntimeFactory` 註冊新 Adapter | 載入邏輯自動分流 |
| 自訂 Attachment 類型 | 擴充 `setAttachment` 方法 | 僅影響 `SpineActor` |
| 新的 Skin 組合邏輯 | 擴充 `combineSkins` 方法 | 僅影響 `SpineActor` |

---

> **下一份文件**：`07-script-engine.md` — 腳本引擎設計規格
