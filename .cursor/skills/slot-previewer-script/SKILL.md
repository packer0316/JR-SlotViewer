---
name: slot-previewer-script
description: Script engine rules for Slot Previewer. Covers JSON script structure, Action handler architecture, plugin registration, ScriptRunner execution, and EventBus. Use when creating script actions, modifying script execution, working with EventBus, or extending the script system.
---

# Slot Previewer — 腳本引擎規則

## 核心設計

- JSON 格式腳本描述轉輪行為與動畫表演
- **Action 註冊制（Plugin 架構）**：新增 Action 不修改 ScriptRunner
- ScriptRunner 是唯一允許跨層操作的模組（透過 ActionContext）

## 腳本結構

```json
{
  "name": "ScriptName",
  "version": 1,
  "steps": [
    { "action": "spin", "duration": 800 },
    { "parallel": [
      { "action": "playSpine", "actor": "wild", "anim": "trigger" },
      { "action": "tween", "target": "overlay", "props": { "alpha": 1 }, "duration": 300 }
    ]}
  ]
}
```

- `steps` 為循序執行；`parallel` 內為 `Promise.all` 並行

## Action Handler 架構

```typescript
interface ActionHandler {
  type: ActionType;
  validate(params: Record<string, any>): ValidationResult;
  execute(params: Record<string, any>, ctx: ActionContext): Promise<void>;
}

interface ActionContext {
  store: StoreAPI;
  reelEngine: ReelEngine;
  spineLoader: SpineLoader;
  layerManager: LayerManager;
  eventBus: EventBus;
  abortSignal: AbortSignal;
}
```

每個 Action 獨立檔案於 `src/script/actions/`，透過 `ActionRegistry.register()` 註冊。

## 新增 Action 步驟

1. 在 `src/script/actions/` 建立 `XxxAction.ts`，實作 `ActionHandler`
2. 在 `actions/index.ts` 註冊
3. 擴充 `ActionType` 聯合型別（`src/types/`）

## ScriptRunner 狀態機

```
IDLE →(run)→ RUNNING →(complete)→ COMPLETED
                ↕ pause/resume
              PAUSED
RUNNING/PAUSED →(abort)→ ABORTED
```

- `pause()`：在 step 邊界暫停（不中斷執行中 action）
- `abort()`：發送 AbortSignal + 清除佇列
- 循環 `callScript` **必須偵測並中止**（呼叫棧追蹤）

## EventBus

- 僅限 Manager 間鬆耦合通訊，UI 不直接訂閱
- 內建事件：`script:start/step/complete/abort`、`reel:spinStart/spinStop`、`spine:complete/event`
- 自定義事件透過 `emit` action 發送
