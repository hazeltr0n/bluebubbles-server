# Clipboard Sync — Server Ring Buffer + Tray Submenu
**Pick this up on the Mac. Repo: `bluebubbles-server`, branch: `feature/my-features`**

## Context
Clipboard sync is partially built. `ClipboardService` already polls the Mac clipboard
and broadcasts `clipboard-sync` to clients. What's missing is the 20-item history
ring buffer and the tray menu submenu that lets you push any item to all devices.

## What's already built (don't re-do)
- `packages/server/src/server/services/clipboardService/index.ts` — polls clipboard
  every 500ms, broadcasts CLIPBOARD_SYNC, writes from clients, echo prevention
- `packages/server/src/server/events.ts` — `CLIPBOARD_SYNC` constant added
- `packages/server/src/server/api/http/api/v1/socketRoutes.ts` — `clipboard-sync` handler
- `packages/server/src/server/index.ts` — service init/start/stop wired in

## What to build

### 1. Add ring buffer to ClipboardService
**File: `packages/server/src/server/services/clipboardService/index.ts`**

Add a history array and expose it:
```typescript
interface ClipboardItem {
    text: string;
    timestamp: number;
}

private history: ClipboardItem[] = [];
private readonly MAX_HISTORY = 20;

getHistory(): ClipboardItem[] {
    return this.history;
}

private addToHistory(text: string) {
    // Don't add duplicates of the most recent item
    if (this.history.length > 0 && this.history[0].text === text) return;
    this.history.unshift({ text, timestamp: Date.now() });
    if (this.history.length > this.MAX_HISTORY) {
        this.history.pop();
    }
}
```

Call `this.addToHistory(current)` in `poll()` before the emit.
Also call `this.addToHistory(text)` in `writeFromClient()`.

### 2. Add Clipboard History submenu to AppTray
**File: `packages/server/src/trays/AppTray.ts`**

Import ClipboardService and Server at the top (check existing imports first).

In `buildMenu()`, add a new submenu entry before or after the server status section:
```typescript
{
    label: 'Clipboard History',
    submenu: (() => {
        const history = Server().clipboardService?.getHistory() ?? [];
        if (history.length === 0) {
            return [{ label: 'No items yet', enabled: false }];
        }
        return history.map(item => ({
            label: `${item.text.substring(0, 45)}${item.text.length > 45 ? '…' : ''}`,
            click: () => Server().emitMessage(
                CLIPBOARD_SYNC,
                { content: item.text },
                'normal',
                false
            )
        }));
    })()
}
```

Import `CLIPBOARD_SYNC` from `@server/events` at the top of AppTray.ts.
Import `Server` from `@server` if not already imported.

## Testing checklist
- [ ] Copy several things on Mac — tray submenu shows them
- [ ] Copy on iPhone — appears in tray within ~2s (via Universal Clipboard)
- [ ] Click item in tray — connected clients receive it and their clipboard updates
- [ ] History caps at 20 items, newest at top

## Key file paths
| File | Purpose |
|------|---------|
| `packages/server/src/server/services/clipboardService/index.ts` | Add ring buffer |
| `packages/server/src/trays/AppTray.ts` | Add history submenu |
| `packages/server/src/server/events.ts` | CLIPBOARD_SYNC already here |
