---
name: slot-previewer-state
description: Zustand state management rules for Slot Previewer. Covers store creation, Immer usage, selector patterns, Manager subscription, and cross-store coordination. Use when creating or modifying Zustand stores, writing selectors, or connecting stores to PixiJS Managers.
---

# Slot Previewer — 狀態管理規則

## 核心原則

- Zustand 為 **唯一真相來源**，PixiJS Scene Graph 不儲存業務狀態
- 每個領域獨立 Slice，共 7 個：`layer`, `spine`, `reel`, `script`, `preview`, `ui`, `project`
- 所有 Store 使用 **Immer middleware**，middleware 順序：外 `devtools` → 內 `immer`
- DevTools 僅 `import.meta.env.DEV` 啟用

## Store 建立範式

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';
import { devtools } from 'zustand/middleware';

const useXxxStore = create<XxxState>()(
  devtools(
    immer((set, get) => ({
      // state + actions
    })),
    { name: 'xxxStore', enabled: import.meta.env.DEV }
  )
);
```

## Selector 規則

- **必須** 使用精細化 selector，禁止 `const store = useStore()`
- 物件選取搭配 `useShallow`
- 衍生資料用 `useMemo`

```typescript
// ✅
const selectedId = useLayerStore(s => s.selectedLayerId);
// ❌
const store = useLayerStore();
```

## Manager 訂閱模式

Manager 透過 `subscribe` 被動監聽，不主動輪詢：

```typescript
class SomeManager {
  private unsub: () => void;
  init() {
    this.unsub = useXxxStore.subscribe(
      (s) => s.targetField,
      (curr, prev) => this.sync(curr, prev),
      { equalityFn: shallow }
    );
  }
  destroy() { this.unsub(); }
}
```

- subscribe callback **禁止** 再呼叫 `set()`（避免無限迴圈）
- `destroy()` 時 **必須** unsubscribe

## 跨 Store 協調

- Store 之間禁止直接 import
- 跨 Store 操作用組合 action 函式或 `getState()` 讀取
- 複雜跨 Store 流程可透過 EventBus 輔助

## Dirty 標記

- 任何 Store 變更 → `projectStore.markDirty()`
- `beforeunload` 事件提醒未儲存變更
