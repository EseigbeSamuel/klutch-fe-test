````markdown
# Klutch AI Front-End Engineering Test

## Setup

```bash
npm install
npm run dev
```
````

Visit http://localhost:5173 to see the component.

## Overview

A task management table with inline editing and batch operations built on top of a Mock API with realistic network delays and a 10% failure rate.

---

## Task 1: Inline Editing

### What Was Built

- Hover-to-reveal pencil icon next to each task title
- Click the icon to enter edit mode with an auto-focused, auto-selected input
- Save with the ✓ button or `Enter` key
- Cancel with the ✗ button or `Escape` key
- Spinner replaces the ✓ icon during the API call
- Inline error message below the input on failure, with edited text preserved for retry
- Row click (`selected` event) is suppressed while editing

### Interaction Pattern: Why a Hover Icon?

A dedicated pencil icon was chosen over double-click for three reasons:

1. **Discoverability** — double-click has no visual affordance. Users can't tell a field is editable without trying. The icon appears on hover, which is a pattern users already know from tools like Notion and Linear.
2. **No conflict with text selection** — double-clicking on text natively selects the word, which fights with an edit trigger on the same element.
3. **Keyboard accessibility** — the icon button is focusable and tabbable. Double-click has no keyboard equivalent.

### Error Handling

The Mock API fails ~10% of the time. On failure:

- The input border turns red
- An error message appears below the input
- The user's edited text is preserved — no re-typing needed
- Hitting `Enter` retries the save immediately

### Event Flow

`TaskTableRow` calls `getMockAPI().updateTask()`, reassigns its local `task` binding on success, then fires `dispatch('updated', updatedTask)`. `App.svelte` handles the event by replacing the task in the list via `tasks.map()`. The row component never directly mutates the shared list — the parent always owns the source of truth.

---

## Task 2: Batch Operations

### What Was Built

**Mock API (`mockApi.ts`):**

```typescript
async updateTasksBatch(
  taskIds: string[],
  updates: Partial<ListableTask>
): Promise<ListableTask[]>
```

- Validates all task IDs exist before updating any (atomic)
- Simulates 200–600ms network delay
- 10% random failure rate

**Frontend (`App.svelte`):**

- Batch toolbar appears only when tasks are selected:
  ```
  [✓ N tasks selected]  [Set status: ▼]  [Cancel]
  ```
- Status dropdown resets to placeholder after each use
- Success banner confirms how many tasks were updated and to which status
- Error banner appears above the table on failure
- Selection clears automatically after a successful batch update
- Shift+Click range selection across rows

### Atomicity

`updateTasksBatch()` runs a full validation pass over all IDs before touching any of them. This mirrors database transaction behavior — either all updates succeed or none do. Partial updates would leave the UI in an inconsistent state that's hard to recover from.

### Shift+Click Range Selection

`App.svelte` tracks `lastCheckedIndex`. When a checkbox fires with `shiftKey: true`, every row between the last index and the current one is filled with the same selection state. This matches the behavior users expect from email clients and file managers, and makes selecting large ranges practical.

### Error Handling

Batch errors appear as a persistent banner above the table (not inline) since multiple rows are affected and there's no single row to anchor the error to. The banner uses the `ValidationError.message` string from the API directly, which already contains human-readable text.

---

## Project Structure

```
src/
├── components/
│   ├── TaskTableRow.svelte       # Inline editing + checkbox selection
│   └── stubs/                    # UI components (Td, Tr, Pill, etc.)
├── types.ts                      # TypeScript types
├── mockApi.ts                    # Simulated backend + batch method
├── mockData.ts                   # Sample tasks (5 tasks)
├── App.svelte                    # Toolbar, selection state, batch update logic
└── app.css                       # Styles
```

---

## Data Flow

### Inline Edit

1. User clicks pencil icon → `isEditing = true`
2. User types and confirms → `getMockAPI().updateTask()`
3. On success → `task = updatedTask`, `dispatch('updated', updatedTask)`
4. `App.svelte` receives event → `tasks.map()` replaces the entry

### Batch Update

1. User selects tasks via checkboxes (with optional Shift+Click)
2. User picks a status from the toolbar dropdown
3. `getMockAPI().updateTasksBatch(selectedIds, { status })` is called
4. On success → tasks array is merged with updated results, selection clears
5. On failure → error banner shown, selection preserved so user can retry

---

## Mock API Behavior

| Method               | Delay     | Failure Rate | Notes                            |
| -------------------- | --------- | ------------ | -------------------------------- |
| `updateTask()`       | 200–600ms | 10%          | Rejects empty titles             |
| `updateTasksBatch()` | 200–600ms | 10%          | Atomic — validates all IDs first |

---

## TypeScript

- `TaskStatus` types the status dropdown value and batch payload — typos caught at build time
- `CustomEvent<...>` generics on all handlers mean destructured payloads are fully typed
- `ValidationError` from `types.ts` is used directly in catch blocks — no casting needed

```

```
