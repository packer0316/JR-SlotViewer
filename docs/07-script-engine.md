# Slot Previewer — 腳本引擎

> **版本**：v1.0 草案
> **最後更新**：2026-03-03
> **前置閱讀**：`01-architecture-overview.md`、`02-module-design.md`

---

## 1. 腳本系統概覽

### 1.1 設計目標

腳本引擎的核心目標是讓**美術與企劃人員**無需撰寫程式碼，即可定義：

- **轉輪如何旋轉與停止**（Spin Sequence）
- **動畫如何演出**（Animation Performance）
- **多個效果如何搭配**（Parallel / Sequential Orchestration）

透過 JSON 格式的腳本檔案，使用者可以描述完整的遊戲流程，從啟動轉輪、逐軸停止、播放 Spine 動畫到疊加特效層，一切皆可在腳本中宣告。

### 1.2 設計原則

| 原則 | 說明 |
|------|------|
| **人類可讀** | JSON 純文字格式，美術可直接編輯或從範本複製修改 |
| **可擴充** | 新增 Action 只需實作介面並註冊，無須修改核心邏輯（Plugin 架構） |
| **安全執行** | 所有 Action 參數於執行前驗證，錯誤不會導致系統崩潰 |
| **可取消** | 任何正在執行的腳本皆可暫停、恢復或中止 |
| **事件驅動** | 透過 EventBus 與其他模組通訊，降低模組間耦合 |

### 1.3 系統全貌

```
JSON Script File
       │
       ▼
┌─────────────────────────────────────────────┐
│              ScriptRunner                     │
│                                               │
│  ① Parse JSON                                 │
│  ② Validate Schema & Actions                  │
│  ③ Build ActionQueue                          │
│  ④ Execute (sequential / parallel)            │
│  ⑤ Report Progress via EventBus               │
│                                               │
│  ┌──────────────────────────────────────────┐ │
│  │           ActionRegistry                   │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │ │
│  │  │ spin │ │ stop │ │spine │ │tween │... │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘    │ │
│  └──────────────────────────────────────────┘ │
└───────────────────┬─────────────────────────┘
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
    Zustand Store  PixiJS   EventBus
    (狀態更新)    Managers   (事件通知)
                (直接指令)
```

---

## 2. 腳本結構定義

### 2.1 頂層結構

每份腳本檔案為一個 JSON 物件，包含以下欄位：

```typescript
interface Script {
  name: string;           // 腳本名稱（唯一識別）
  version: number;        // 腳本格式版本號
  description?: string;   // 說明文字
  steps: Step[];          // 執行步驟陣列
}
```

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `name` | `string` | ✅ | 腳本唯一名稱，用於 `callScript` 引用 |
| `version` | `number` | ✅ | 腳本格式版本，目前為 `1` |
| `description` | `string` | ❌ | 腳本用途描述，供人類閱讀 |
| `steps` | `Step[]` | ✅ | 按順序執行的步驟陣列 |

### 2.2 步驟結構

每個步驟可以是**單一 Action** 或**並行群組**：

```typescript
type Step = ActionStep | ParallelStep;

interface ActionStep {
  action: ActionType;
  [key: string]: any;     // Action 專屬參數
}

interface ParallelStep {
  parallel: ActionStep[];  // 同時執行的 Action 陣列
}
```

### 2.3 完整範例：大獎演出腳本

```json
{
  "name": "BigWinSequence",
  "version": 1,
  "description": "大獎表演腳本",
  "steps": [
    { "action": "spin", "duration": 800 },
    { "action": "stopReel", "reel": 0, "delay": 200 },
    { "action": "stopReel", "reel": 1, "delay": 400 },
    { "action": "stopReel", "reel": 2, "delay": 600 },
    { "action": "stopReel", "reel": 3, "delay": 800 },
    { "action": "stopReel", "reel": 4, "delay": 1000 },
    { "action": "playSpine", "actor": "wild", "anim": "trigger", "track": 0 },
    { "action": "wait", "duration": 600 },
    {
      "parallel": [
        { "action": "playSpine", "actor": "bigwin", "anim": "idle", "loop": true },
        { "action": "setLayerVisible", "layer": "winOverlay", "visible": true },
        { "action": "tween", "target": "winOverlay", "props": { "alpha": 1 }, "duration": 300 }
      ]
    },
    { "action": "wait", "duration": 3000 },
    { "action": "tween", "target": "winOverlay", "props": { "alpha": 0 }, "duration": 300 },
    { "action": "setLayerVisible", "layer": "winOverlay", "visible": false }
  ]
}
```

**執行流程說明**：

1. 啟動轉輪旋轉 800ms
2. 依序停止第 0 ~ 4 軸（間隔遞增，模擬逐軸停輪效果）
3. 播放 Wild 角色的觸發動畫
4. 等待 600ms
5. **同時**：播放大獎動畫（循環）、顯示贏分覆蓋層、淡入覆蓋層
6. 等待 3 秒鐘展示大獎
7. 淡出覆蓋層
8. 隱藏贏分覆蓋層

---

## 3. 內建 Action 完整清單

### 3.1 Action 一覽表

| Action 名稱 | 說明 | 參數 | 非同步 |
|-------------|------|------|--------|
| `spin` | 啟動轉輪旋轉 | `duration`, `speed` | ✅ 是 |
| `stopReel` | 停止指定輪 | `reel`, `delay`, `result` | ✅ 是 |
| `stopAll` | 停止所有輪 | `delay` | ✅ 是 |
| `playSpine` | 播放 Spine 動畫 | `actor`, `anim`, `track`, `loop` | ✅ 是（非循環時等待播完） |
| `mixSpine` | 帶過渡切換動畫 | `actor`, `anim`, `mix` | ✅ 是 |
| `setSkin` | 切換 Spine 皮膚 | `actor`, `skin` | ❌ 否 |
| `setLayerVisible` | 圖層顯示 / 隱藏 | `layer`, `visible` | ❌ 否 |
| `setLayerAlpha` | 圖層透明度 | `layer`, `alpha` | ❌ 否 |
| `tween` | GSAP 補間動畫 | `target`, `props`, `duration`, `ease` | ✅ 是 |
| `wait` | 等待毫秒數 | `duration` | ✅ 是 |
| `waitEvent` | 等待事件觸發 | `event` | ✅ 是 |
| `emit` | 發送自定義事件 | `event`, `payload` | ❌ 否 |
| `callScript` | 嵌套呼叫腳本 | `name` | ✅ 是 |

### 3.2 各 Action 詳細規格

#### `spin` — 啟動轉輪旋轉

```json
{ "action": "spin", "duration": 800, "speed": 1.0 }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `duration` | `number` | ❌ | `1000` | 旋轉持續時間（ms） |
| `speed` | `number` | ❌ | `1.0` | 旋轉速度倍率 |

呼叫 `ReelEngine.startSpin()`，等待指定 `duration` 後 resolve。

#### `stopReel` — 停止指定輪

```json
{ "action": "stopReel", "reel": 0, "delay": 200, "result": ["wild", "scatter", "A"] }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `reel` | `number` | ✅ | — | 軸編號（從 0 開始） |
| `delay` | `number` | ❌ | `0` | 延遲後再停止（ms） |
| `result` | `string[]` | ❌ | — | 指定停輪結果 Symbol |

延遲 `delay` 毫秒後，呼叫 `ReelEngine.stopReel(reel, result)`，等待停輪動畫結束後 resolve。

#### `stopAll` — 停止所有輪

```json
{ "action": "stopAll", "delay": 100 }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `delay` | `number` | ❌ | `0` | 每軸間的延遲間隔（ms） |

依序對所有輪軸呼叫 `stopReel`，軸間間隔 `delay` 毫秒，全部停止後 resolve。

#### `playSpine` — 播放 Spine 動畫

```json
{ "action": "playSpine", "actor": "wild", "anim": "trigger", "track": 0, "loop": false }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `actor` | `string` | ✅ | — | Spine Actor 名稱 |
| `anim` | `string` | ✅ | — | 動畫名稱 |
| `track` | `number` | ❌ | `0` | 動畫軌道 |
| `loop` | `boolean` | ❌ | `false` | 是否循環播放 |

`loop = false` 時等待動畫播放完畢後 resolve；`loop = true` 時**立即 resolve**，動畫持續在背景循環。

#### `mixSpine` — 帶過渡切換動畫

```json
{ "action": "mixSpine", "actor": "wild", "anim": "idle", "mix": 0.2 }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `actor` | `string` | ✅ | — | Spine Actor 名稱 |
| `anim` | `string` | ✅ | — | 目標動畫名稱 |
| `mix` | `number` | ❌ | `0.2` | 過渡混合時間（秒） |

設定 `AnimationStateData.setMix()` 後切換動畫，等待過渡完成後 resolve。

#### `setSkin` — 切換 Spine 皮膚

```json
{ "action": "setSkin", "actor": "wild", "skin": "golden" }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `actor` | `string` | ✅ | Spine Actor 名稱 |
| `skin` | `string` | ✅ | 皮膚名稱 |

同步操作，立即套用皮膚並返回。

#### `setLayerVisible` — 圖層顯示 / 隱藏

```json
{ "action": "setLayerVisible", "layer": "winOverlay", "visible": true }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `layer` | `string` | ✅ | 圖層名稱或 ID |
| `visible` | `boolean` | ✅ | 是否可見 |

同步操作，透過 LayerManager 更新圖層 `visible` 屬性。

#### `setLayerAlpha` — 圖層透明度

```json
{ "action": "setLayerAlpha", "layer": "winOverlay", "alpha": 0.5 }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `layer` | `string` | ✅ | 圖層名稱或 ID |
| `alpha` | `number` | ✅ | 透明度（0–1） |

同步操作，透過 LayerManager 更新圖層 `alpha` 屬性。

#### `tween` — GSAP 補間動畫

```json
{ "action": "tween", "target": "winOverlay", "props": { "alpha": 1, "y": 100 }, "duration": 300, "ease": "power2.out" }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `target` | `string` | ✅ | — | 目標圖層名稱或 ID |
| `props` | `object` | ✅ | — | 要補間的屬性與目標值 |
| `duration` | `number` | ✅ | — | 動畫時長（ms） |
| `ease` | `string` | ❌ | `"power1.out"` | GSAP 緩動函式 |

透過 GSAP `gsap.to()` 驅動目標 Container 的屬性變化，動畫完成後 resolve。支援的補間屬性包括 `alpha`、`x`、`y`、`scaleX`、`scaleY`、`rotation`。

#### `wait` — 等待毫秒數

```json
{ "action": "wait", "duration": 600 }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `duration` | `number` | ✅ | 等待時間（ms） |

純粹的計時延遲，支援 `AbortSignal` 提前中斷。

#### `waitEvent` — 等待事件觸發

```json
{ "action": "waitEvent", "event": "reel:allStopped" }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `event` | `string` | ✅ | 等待的事件名稱 |

監聽 EventBus 上的指定事件，觸發後 resolve。支援 `AbortSignal` 提前中斷。

#### `emit` — 發送自定義事件

```json
{ "action": "emit", "event": "custom:bigwinStart", "payload": { "multiplier": 50 } }
```

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| `event` | `string` | ✅ | — | 事件名稱 |
| `payload` | `any` | ❌ | `undefined` | 附帶資料 |

同步操作，透過 EventBus 發送事件，可被其他模組或 `waitEvent` 接收。

#### `callScript` — 嵌套呼叫腳本

```json
{ "action": "callScript", "name": "FreespinIntro" }
```

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `name` | `string` | ✅ | 被呼叫腳本的 `name` 欄位值 |

建立子 ScriptRunner 執行目標腳本，等待其完成後 resolve。系統會偵測**循環呼叫**（A → B → A）並拋出錯誤。

---

## 4. Action Handler 架構

### 4.1 介面定義

每個 Action Handler 為一個獨立檔案，實作以下介面：

```typescript
/** Action 類型聯合 */
type ActionType =
  | 'spin' | 'stopReel' | 'stopAll'
  | 'playSpine' | 'mixSpine' | 'setSkin'
  | 'setLayerVisible' | 'setLayerAlpha'
  | 'tween' | 'wait' | 'waitEvent'
  | 'emit' | 'callScript';

/** 驗證結果 */
interface ValidationResult {
  valid: boolean;
  errors: string[];
}

/** Action Handler 介面 */
interface ActionHandler {
  /** 此 Handler 處理的 Action 類型 */
  type: ActionType;

  /** 參數驗證——於腳本載入時呼叫 */
  validate(params: Record<string, any>): ValidationResult;

  /** 執行 Action——於腳本運行時呼叫 */
  execute(params: Record<string, any>, context: ActionContext): Promise<void>;
}
```

### 4.2 執行上下文

所有 Action Handler 共享同一份 `ActionContext`，提供存取系統各子模組的能力：

```typescript
interface ActionContext {
  store: StoreAPI;          // 存取所有 Zustand Store
  reelEngine: ReelEngine;   // 轉輪引擎
  spineLoader: SpineLoader; // Spine 資源載入與管理
  layerManager: LayerManager; // 圖層管理
  eventBus: EventBus;       // 事件匯流排
  abortSignal: AbortSignal; // 取消信號（用於中止執行中的 Action）
}
```

`abortSignal` 來自 `AbortController`，當使用者呼叫 `ScriptRunner.abort()` 時觸發。所有非同步 Action 皆須監聽此信號以支援取消：

```typescript
class WaitAction implements ActionHandler {
  type = 'wait' as const;

  validate(params: Record<string, any>): ValidationResult {
    if (typeof params.duration !== 'number' || params.duration < 0) {
      return { valid: false, errors: ['duration 必須為正數'] };
    }
    return { valid: true, errors: [] };
  }

  async execute(params: Record<string, any>, context: ActionContext): Promise<void> {
    const { duration } = params;
    const { abortSignal } = context;

    return new Promise<void>((resolve, reject) => {
      const timer = setTimeout(resolve, duration);

      abortSignal.addEventListener('abort', () => {
        clearTimeout(timer);
        reject(new DOMException('Aborted', 'AbortError'));
      }, { once: true });
    });
  }
}
```

### 4.3 Action Registry

採用**註冊制**管理所有 Action Handler，實現 Plugin 架構：

```typescript
class ActionRegistry {
  private static handlers = new Map<ActionType, ActionHandler>();

  /** 註冊 Action Handler */
  static register(handler: ActionHandler): void {
    if (this.handlers.has(handler.type)) {
      console.warn(`Action "${handler.type}" 已被註冊，將被覆蓋`);
    }
    this.handlers.set(handler.type, handler);
  }

  /** 取得指定類型的 Handler */
  static get(type: ActionType): ActionHandler | undefined {
    return this.handlers.get(type);
  }

  /** 取得所有已註冊的 Handler */
  static getAll(): ActionHandler[] {
    return Array.from(this.handlers.values());
  }

  /** 檢查指定類型是否已註冊 */
  static has(type: ActionType): boolean {
    return this.handlers.has(type);
  }
}
```

### 4.4 目錄結構

```
src/script/actions/
├── index.ts                # ActionRegistry + 自動註冊所有 Handler
├── SpinAction.ts           # spin — 啟動轉輪旋轉
├── StopReelAction.ts       # stopReel — 停止指定輪
├── StopAllAction.ts        # stopAll — 停止所有輪
├── PlaySpineAction.ts      # playSpine — 播放 Spine 動畫
├── MixSpineAction.ts       # mixSpine — 帶過渡切換動畫
├── SetSkinAction.ts        # setSkin — 切換 Spine 皮膚
├── SetLayerVisibleAction.ts # setLayerVisible — 圖層顯示 / 隱藏
├── SetLayerAlphaAction.ts  # setLayerAlpha — 圖層透明度
├── TweenAction.ts          # tween — GSAP 補間動畫
├── WaitAction.ts           # wait — 等待毫秒數
├── WaitEventAction.ts      # waitEvent — 等待事件觸發
├── EmitAction.ts           # emit — 發送自定義事件
└── CallScriptAction.ts     # callScript — 嵌套呼叫腳本
```

### 4.5 自動註冊

`index.ts` 負責匯入所有 Handler 並自動完成註冊：

```typescript
// src/script/actions/index.ts
import { ActionRegistry } from '../ActionRegistry';

import { SpinAction } from './SpinAction';
import { StopReelAction } from './StopReelAction';
import { StopAllAction } from './StopAllAction';
import { PlaySpineAction } from './PlaySpineAction';
import { MixSpineAction } from './MixSpineAction';
import { SetSkinAction } from './SetSkinAction';
import { SetLayerVisibleAction } from './SetLayerVisibleAction';
import { SetLayerAlphaAction } from './SetLayerAlphaAction';
import { TweenAction } from './TweenAction';
import { WaitAction } from './WaitAction';
import { WaitEventAction } from './WaitEventAction';
import { EmitAction } from './EmitAction';
import { CallScriptAction } from './CallScriptAction';

const builtinActions = [
  new SpinAction(),
  new StopReelAction(),
  new StopAllAction(),
  new PlaySpineAction(),
  new MixSpineAction(),
  new SetSkinAction(),
  new SetLayerVisibleAction(),
  new SetLayerAlphaAction(),
  new TweenAction(),
  new WaitAction(),
  new WaitEventAction(),
  new EmitAction(),
  new CallScriptAction(),
];

builtinActions.forEach((action) => ActionRegistry.register(action));

export { ActionRegistry };
```

---

## 5. ScriptRunner 執行流程

### 5.1 解析階段

腳本載入後的完整驗證流程：

```
JSON 字串
   │
   ▼
① JSON.parse()
   │ 失敗 → 拋出 ScriptParseError
   ▼
② Schema 驗證
   │ 檢查 name (string), version (number), steps (array)
   │ 失敗 → 拋出 ScriptSchemaError
   ▼
③ 逐步驗證每個 Step
   │ ├── ActionStep: 檢查 action 類型是否已註冊
   │ │   └── 呼叫 handler.validate(params)
   │ └── ParallelStep: 遞迴驗證 parallel 陣列中的每個 ActionStep
   │ 失敗 → 收集所有錯誤，拋出 ScriptValidationError
   ▼
④ 建構 ActionQueue
   │ 將 steps 轉換為可執行的佇列結構
   ▼
⑤ 返回 ParsedScript（含 metadata 與 ActionQueue）
```

```typescript
interface ParsedScript {
  name: string;
  version: number;
  description?: string;
  queue: ActionQueueItem[];
  totalSteps: number;
}

type ActionQueueItem =
  | { type: 'sequential'; action: ActionType; params: Record<string, any> }
  | { type: 'parallel'; actions: { action: ActionType; params: Record<string, any> }[] };
```

### 5.2 執行階段

```
ActionQueue
   │
   ▼
┌──────────────────────────────────────────────────────┐
│  while (queue 不為空 && 狀態 !== ABORTED)              │
│                                                       │
│  ① Dequeue 下一個 item                                │
│     │                                                 │
│     ├── type === 'sequential'                         │
│     │   └── await handler.execute(params, context)    │
│     │                                                 │
│     └── type === 'parallel'                           │
│         └── await Promise.all(                        │
│               actions.map(a =>                        │
│                 handler.execute(a.params, context)     │
│               )                                       │
│             )                                         │
│                                                       │
│  ② currentStep++                                      │
│  ③ emit('script:step', { step: currentStep,           │
│         total: totalSteps, action: item })            │
│                                                       │
│  ④ 若狀態 === PAUSED → await resumePromise            │
│                                                       │
└──────────────────────────────────────────────────────┘
   │
   ▼
emit('script:complete', { name: scriptName })
```

核心執行邏輯：

```typescript
class ScriptRunner {
  private state: ScriptState = 'idle';
  private currentStep = 0;
  private abortController: AbortController | null = null;
  private resumeResolve: (() => void) | null = null;

  async run(script: ParsedScript, context: Omit<ActionContext, 'abortSignal'>): Promise<void> {
    this.state = 'running';
    this.currentStep = 0;
    this.abortController = new AbortController();

    const actionContext: ActionContext = {
      ...context,
      abortSignal: this.abortController.signal,
    };

    context.eventBus.emit('script:start', { name: script.name });

    try {
      for (const item of script.queue) {
        if (this.state === 'aborted') break;

        if (this.state === 'paused') {
          await new Promise<void>((resolve) => {
            this.resumeResolve = resolve;
          });
        }

        if (item.type === 'sequential') {
          const handler = ActionRegistry.get(item.action);
          if (!handler) {
            console.warn(`未知的 Action 類型: ${item.action}，跳過`);
            this.currentStep++;
            continue;
          }
          await handler.execute(item.params, actionContext);
        } else {
          const promises = item.actions.map((a) => {
            const handler = ActionRegistry.get(a.action);
            if (!handler) {
              console.warn(`未知的 Action 類型: ${a.action}，跳過`);
              return Promise.resolve();
            }
            return handler.execute(a.params, actionContext);
          });
          await Promise.all(promises);
        }

        this.currentStep++;
        context.eventBus.emit('script:step', {
          step: this.currentStep,
          total: script.totalSteps,
          action: item,
        });
      }

      this.state = 'completed';
      context.eventBus.emit('script:complete', { name: script.name });
    } catch (error) {
      if (error instanceof DOMException && error.name === 'AbortError') {
        this.state = 'aborted';
        context.eventBus.emit('script:abort', { name: script.name });
      } else {
        this.state = 'idle';
        context.eventBus.emit('script:error', { name: script.name, error });
        throw error;
      }
    }
  }
}
```

### 5.3 控制流

ScriptRunner 提供三個控制方法：

```typescript
class ScriptRunner {
  /** 暫停：在當前步驟邊界暫停，不中斷正在執行的 Action */
  pause(): void {
    if (this.state === 'running') {
      this.state = 'paused';
    }
  }

  /** 恢復：繼續執行被暫停的腳本 */
  resume(): void {
    if (this.state === 'paused') {
      this.state = 'running';
      this.resumeResolve?.();
      this.resumeResolve = null;
    }
  }

  /** 中止：清空佇列，發送 abort 信號，停止一切 */
  abort(): void {
    this.state = 'aborted';
    this.abortController?.abort();
    this.resumeResolve?.();
    this.resumeResolve = null;
  }
}
```

### 5.4 狀態機

```
                    ┌──────────────────────────────────────┐
                    │         ScriptRunner 狀態機           │
                    └──────────────────────────────────────┘

                              run()
                    IDLE ─────────────► RUNNING
                                          │
                              ┌───────────┼───────────┐
                              │           │           │
                          pause()    complete()    abort()
                              │           │           │
                              ▼           ▼           ▼
                           PAUSED    COMPLETED     ABORTED
                              │
                    ┌─────────┼─────────┐
                    │                   │
                resume()            abort()
                    │                   │
                    ▼                   ▼
                 RUNNING             ABORTED
```

| 狀態 | 說明 | 可轉換至 |
|------|------|---------|
| `IDLE` | 初始狀態，未執行任何腳本 | `RUNNING` |
| `RUNNING` | 正在執行腳本步驟 | `PAUSED`、`COMPLETED`、`ABORTED` |
| `PAUSED` | 已暫停，等待 resume 或 abort | `RUNNING`、`ABORTED` |
| `COMPLETED` | 所有步驟執行完畢 | `IDLE`（重新 run 時） |
| `ABORTED` | 被使用者中止 | `IDLE`（重新 run 時） |

```typescript
type ScriptState = 'idle' | 'running' | 'paused' | 'completed' | 'aborted';
```

---

## 6. EventBus 事件系統

### 6.1 API 定義

EventBus 是一個輕量級的發布 / 訂閱系統，用於腳本引擎與其他模組之間的解耦通訊：

```typescript
class EventBus {
  private listeners = new Map<string, Set<(...args: any[]) => void>>();

  /** 訂閱事件，回傳取消訂閱函式 */
  on(event: string, handler: (...args: any[]) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(handler);

    return () => this.off(event, handler);
  }

  /** 訂閱事件（僅觸發一次） */
  once(event: string, handler: (...args: any[]) => void): () => void {
    const wrapper = (...args: any[]) => {
      this.off(event, wrapper);
      handler(...args);
    };
    return this.on(event, wrapper);
  }

  /** 取消訂閱 */
  off(event: string, handler: (...args: any[]) => void): void {
    this.listeners.get(event)?.delete(handler);
  }

  /** 發送事件 */
  emit(event: string, ...args: any[]): void {
    this.listeners.get(event)?.forEach((handler) => {
      try {
        handler(...args);
      } catch (error) {
        console.error(`EventBus handler error on "${event}":`, error);
      }
    });
  }

  /** 移除所有監聽器 */
  removeAllListeners(event?: string): void {
    if (event) {
      this.listeners.delete(event);
    } else {
      this.listeners.clear();
    }
  }
}
```

### 6.2 內建事件清單

#### 腳本生命週期事件

| 事件名稱 | 觸發時機 | Payload |
|----------|---------|---------|
| `script:start` | 腳本開始執行 | `{ name: string }` |
| `script:step` | 每個步驟執行完畢 | `{ step: number, total: number, action: ActionQueueItem }` |
| `script:complete` | 腳本全部執行完畢 | `{ name: string }` |
| `script:abort` | 腳本被中止 | `{ name: string }` |
| `script:error` | 腳本執行出錯 | `{ name: string, error: Error }` |

#### 轉輪事件

| 事件名稱 | 觸發時機 | Payload |
|----------|---------|---------|
| `reel:spinStart` | 轉輪開始旋轉 | `{}` |
| `reel:spinStop` | 單軸停止 | `{ reel: number, result: string[] }` |
| `reel:allStopped` | 所有輪軸停止 | `{ results: string[][] }` |

#### Spine 事件

| 事件名稱 | 觸發時機 | Payload |
|----------|---------|---------|
| `spine:complete` | Spine 動畫播放完畢 | `{ actor: string, anim: string }` |
| `spine:event` | Spine 動畫觸發自訂事件 | `{ actor: string, event: string, data: any }` |

#### 自定義事件

透過 `emit` Action 發送的事件，名稱與 payload 由腳本作者自行定義：

```json
{ "action": "emit", "event": "custom:phaseChange", "payload": { "phase": "freespin" } }
```

### 6.3 使用範例

```typescript
const eventBus = new EventBus();

const unsubscribe = eventBus.on('script:step', ({ step, total }) => {
  console.log(`進度：${step} / ${total}`);
  progressStore.setProgress(step / total);
});

eventBus.once('script:complete', ({ name }) => {
  console.log(`腳本 "${name}" 執行完畢`);
});

// 元件卸載時清理
unsubscribe();
```

---

## 7. 腳本面板 UI

### 7.1 面板佈局

```
┌─────────────────────────────────────────────┐
│  腳本面板                              ≡ ✕  │
├─────────────────────────────────────────────┤
│                                             │
│  📂 載入腳本                                 │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │ ▸ BigWinSequence        ← 已選取     │   │
│  │   FreespinIntro                      │   │
│  │   BonusRoundDemo                     │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ ▶ 執行   │ │ ⏸ 暫停   │ │ ⏹ 停止   │   │
│  └──────────┘ └──────────┘ └──────────┘   │
│                                             │
│  進度 ████████████░░░░░░░░░░  5 / 12        │
│                                             │
│  步驟清單：                                   │
│  ┌──────────────────────────────────────┐   │
│  │ ✅ 1. spin (800ms)                   │   │
│  │ ✅ 2. stopReel #0 (200ms)            │   │
│  │ ✅ 3. stopReel #1 (400ms)            │   │
│  │ ✅ 4. stopReel #2 (600ms)            │   │
│  │ 🔄 5. stopReel #3 (800ms)  ← 執行中  │   │
│  │ ⬜ 6. stopReel #4 (1000ms)           │   │
│  │ ⬜ 7. playSpine "wild"               │   │
│  │ ⬜ 8. wait (600ms)                   │   │
│  │ ⬜ 9. [parallel] 3 actions           │   │
│  │ ⬜ 10. wait (3000ms)                 │   │
│  │ ⬜ 11. tween "winOverlay"            │   │
│  │ ⬜ 12. setLayerVisible               │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 7.2 功能說明

| 元件 | 功能 | 說明 |
|------|------|------|
| **載入腳本** | 開啟檔案選取器 | 支援 `.json` 檔案，載入後自動驗證 |
| **腳本清單** | 顯示已載入的腳本 | 點擊選取，顯示步驟清單 |
| **執行按鈕** | 開始執行選取的腳本 | 執行中時停用 |
| **暫停按鈕** | 暫停 / 恢復執行 | 僅在 RUNNING 狀態時可用 |
| **停止按鈕** | 中止執行 | 僅在 RUNNING 或 PAUSED 狀態時可用 |
| **進度指示** | 顯示執行進度 | 進度條 + 步驟數文字 |
| **步驟清單** | 顯示所有步驟及狀態 | 自動捲動至當前執行中步驟 |

### 7.3 步驟狀態

| 狀態 | 圖示 | 說明 |
|------|------|------|
| `pending` | ⬜ | 尚未執行 |
| `running` | 🔄 | 正在執行中（高亮顯示） |
| `completed` | ✅ | 已完成 |
| `skipped` | ⏭️ | 已跳過（Action 類型未知或驗證失敗） |
| `error` | ❌ | 執行時發生錯誤 |

---

## 8. 腳本驗證與錯誤處理

### 8.1 驗證階段

腳本在**載入時**即進行完整驗證，確保執行前發現所有問題：

```
┌─────────────────────────────────────────────────┐
│                驗證流程                            │
├─────────────────────────────────────────────────┤
│                                                  │
│  ① JSON 語法驗證                                  │
│     └── 失敗 → ScriptParseError                  │
│         「JSON 解析失敗：第 15 行語法錯誤」          │
│                                                  │
│  ② Schema 結構驗證                                │
│     ├── name 是否為字串？                          │
│     ├── version 是否為數字？                       │
│     └── steps 是否為陣列？                         │
│         └── 失敗 → ScriptSchemaError             │
│             「缺少必要欄位：name」                  │
│                                                  │
│  ③ Action 驗證                                    │
│     ├── action 類型是否已註冊？                     │
│     │   └── 未知 → ⚠️ 警告（不阻止載入）            │
│     └── 參數是否合法？                              │
│         └── 呼叫 handler.validate(params)         │
│             └── 失敗 → 收集錯誤清單                 │
│                                                  │
│  ④ 呼叫鏈驗證                                     │
│     └── callScript 是否存在循環引用？               │
│         └── 是 → ScriptValidationError           │
│             「偵測到循環呼叫：A → B → A」           │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 8.2 錯誤類型

```typescript
class ScriptParseError extends Error {
  constructor(message: string, public readonly source: string) {
    super(`腳本解析失敗：${message}`);
  }
}

class ScriptSchemaError extends Error {
  constructor(public readonly missingFields: string[]) {
    super(`腳本結構錯誤：缺少必要欄位 ${missingFields.join(', ')}`);
  }
}

class ScriptValidationError extends Error {
  constructor(public readonly errors: { step: number; message: string }[]) {
    super(`腳本驗證失敗：共 ${errors.length} 個問題`);
  }
}

class ScriptRuntimeError extends Error {
  constructor(
    message: string,
    public readonly step: number,
    public readonly action: string,
  ) {
    super(`執行錯誤 [步驟 ${step}, ${action}]：${message}`);
  }
}
```

### 8.3 執行時錯誤處理策略

| 錯誤場景 | 處理方式 | 說明 |
|----------|---------|------|
| 未知 Action 類型 | ⚠️ 警告 + 跳過 | 記錄警告日誌，標記步驟為 `skipped`，繼續執行 |
| 必要參數缺失 | ❌ 錯誤 + 中止 | 於驗證階段即阻止載入 |
| Action 執行時例外 | 🔸 捕獲 + 可配置 | 預設記錄錯誤並中止，可透過設定改為跳過 |
| `callScript` 循環引用 | ❌ 錯誤 + 中止 | 載入階段偵測，或執行時維護呼叫棧偵測 |
| `AbortError`（使用者中止） | 正常結束 | 非錯誤，狀態切換為 `ABORTED` |

### 8.4 循環呼叫偵測

```typescript
class ScriptRunner {
  private callStack: string[] = [];

  private async executeCallScript(name: string, context: ActionContext): Promise<void> {
    if (this.callStack.includes(name)) {
      const chain = [...this.callStack, name].join(' → ');
      throw new ScriptRuntimeError(
        `偵測到循環呼叫：${chain}`,
        this.currentStep,
        'callScript',
      );
    }

    this.callStack.push(name);
    try {
      const childScript = this.loadScript(name);
      const childRunner = new ScriptRunner();
      childRunner.callStack = [...this.callStack]; // 繼承呼叫棧
      await childRunner.run(childScript, context);
    } finally {
      this.callStack.pop();
    }
  }
}
```

---

## 9. 擴充性

### 9.1 新增自定義 Action

只需三步即可新增一個 Action 類型：

**步驟一**：建立 Handler 檔案

```typescript
// src/script/actions/ShakeAction.ts
import type { ActionHandler, ActionContext, ValidationResult } from '../types';

export class ShakeAction implements ActionHandler {
  type = 'shake' as const;

  validate(params: Record<string, any>): ValidationResult {
    const errors: string[] = [];
    if (typeof params.target !== 'string') errors.push('target 必須為字串');
    if (typeof params.intensity !== 'number') errors.push('intensity 必須為數字');
    if (typeof params.duration !== 'number') errors.push('duration 必須為數字');
    return { valid: errors.length === 0, errors };
  }

  async execute(params: Record<string, any>, context: ActionContext): Promise<void> {
    const { target, intensity = 5, duration = 300 } = params;
    const container = context.layerManager.getContainerByName(target);
    if (!container) return;

    const originalX = container.x;
    const originalY = container.y;
    const startTime = Date.now();

    return new Promise<void>((resolve) => {
      const shake = () => {
        const elapsed = Date.now() - startTime;
        if (elapsed >= duration || context.abortSignal.aborted) {
          container.x = originalX;
          container.y = originalY;
          resolve();
          return;
        }
        const decay = 1 - elapsed / duration;
        container.x = originalX + (Math.random() - 0.5) * intensity * 2 * decay;
        container.y = originalY + (Math.random() - 0.5) * intensity * 2 * decay;
        requestAnimationFrame(shake);
      };
      shake();
    });
  }
}
```

**步驟二**：在 `index.ts` 中匯入並註冊

```typescript
import { ShakeAction } from './ShakeAction';

builtinActions.push(new ShakeAction());
```

**步驟三**：在腳本中使用

```json
{ "action": "shake", "target": "reelContainer", "intensity": 8, "duration": 500 }
```

### 9.2 腳本模板

提供常用的預定義腳本模板，降低使用門檻：

| 模板名稱 | 說明 | 步驟數 |
|----------|------|--------|
| `BasicSpin` | 基本轉輪旋轉 + 逐軸停止 | 6 |
| `BigWinSequence` | 大獎演出（含 Spine 動畫 + 疊加層） | 12 |
| `FreespinIntro` | 免費旋轉進場演出 | 8 |
| `BonusReveal` | Bonus 揭曉動畫 | 10 |
| `QuickSpin` | 快速旋轉（所有軸同時停止） | 3 |

使用者可從模板建立腳本，再根據需求修改。

### 9.3 視覺化編輯器（P3 — 未來規劃）

未來計畫提供**時間軸式的視覺化編輯器**，讓使用者以拖拽方式編排 Action，取代手動編寫 JSON：

```
┌──────────────────────────────────────────────────────┐
│  Timeline Editor (P3)                                 │
├──────────────────────────────────────────────────────┤
│                                                       │
│  時間軸 ──────────────────────────────────────────►    │
│  0ms    500ms   1000ms  1500ms  2000ms  2500ms        │
│  │       │       │       │       │       │            │
│  ├───────┤                                            │
│  │ spin  │                                            │
│  ├───┤                                                │
│  │   ├───────┤                                        │
│  │   │stopR0 │                                        │
│  │   ├───────────┤                                    │
│  │   │  stopR1   │                                    │
│  │   ├───────────────┤                                │
│  │   │   stopR2      │                                │
│  │               ├─────────────────────────────┤      │
│  │               │  playSpine "wild"           │      │
│  │                       ├───────────────────┤        │
│  │                       │ [parallel group]  │        │
│  │                                                    │
│  ┌────────────────────────────────────────────┐       │
│  │ 屬性面板：spin                              │       │
│  │ duration: [800    ] ms                      │       │
│  │ speed:    [1.0    ]                         │       │
│  └────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────┘
```

視覺化編輯器最終產出的仍然是標準 JSON 腳本格式，確保與手動編輯完全相容。

### 9.4 條件分支（未來規劃）

規劃支援基於狀態或結果的條件分支，例如依據中獎結果決定演出流程：

```json
{
  "action": "if",
  "condition": { "store": "reelStore", "path": "totalWin", "op": ">=", "value": 1000 },
  "then": [
    { "action": "callScript", "name": "BigWinSequence" }
  ],
  "else": [
    { "action": "callScript", "name": "NormalWin" }
  ]
}
```

### 9.5 迴圈（未來規劃）

規劃支援重複執行步驟 N 次，適用於免費旋轉等場景：

```json
{
  "action": "repeat",
  "count": 10,
  "steps": [
    { "action": "callScript", "name": "BasicSpin" },
    { "action": "wait", "duration": 500 }
  ]
}
```

---

> **下一份文件**：`08-project-io.md` — 專案匯出 / 匯入設計規格
