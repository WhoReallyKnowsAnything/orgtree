# Phase 4: macOS UI & OS Integration Parity - Pattern Map

**Mapped:** 2026-09-17
**Files analyzed:** 6
**Analogs found:** 5 / 6 (all changes are edits to existing files; no CONTEXT.md/RESEARCH.md exist, this phase is Electron main-process OS-chrome work per 04-UI-SPEC.md Scope Note)

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|---|---|---|---|---|
| `apps/desktop/main/index.ts` (add `Menu.setApplicationMenu` call, UI-03) | controller (app bootstrap) | event-driven | same file, existing tray menu build (`rebuildTray`, L351-412) | exact (same file) |
| `apps/desktop/main/index.ts` (`show()` recreate-if-destroyed, UI-02) | controller | event-driven | same file, existing `show()` L221 + main-window construction L1323-1340 | exact (same file) |
| `apps/desktop/main/taskbar-attention.ts` (mac dock-bounce branch, UI-01) | utility/service (OS integration) | event-driven | itself (`TaskbarAttention` class, `flashFrame` branch) | exact - extend existing class, do not fork |
| `apps/desktop/main/index.ts` (`app.setBadgeCount`, UI-04) | controller | event-driven | same call site as taskbar-attention wiring, L1196 `desktop:pending-attention` handler | exact (same file) |
| `apps/desktop/main/index.ts` (`runtimeIcon()` Template image, UI-05) | service (icon generation) | transform | itself, `runtimeIcon()` L195-213 | exact (same file) |
| `apps/desktop/renderer/src/update-notice.tsx` (mac "View release" copy variant, UPD-01) | component | request-response | itself, `LABEL`/`updateActionTitle`/button-vs-label branch L79-100 | exact (same file) |

No brand-new files are created in this phase - every UI-0x/UPD-01 requirement is an edit to an existing file. Confirmed by grep: no `Menu.setApplicationMenu`, `app.dock`, or `setBadgeCount` calls exist anywhere in `apps/desktop/main/index.ts` today.

## Pattern Assignments

### UI-01: Dock Attention/Bounce - `apps/desktop/main/taskbar-attention.ts`

**Analog:** itself (extend, do not fork). Full file already read (70 lines).

**Existing class shape to extend** (`taskbar-attention.ts` L1-70):
```typescript
interface FlashWindow {
  isDestroyed(): boolean
  isFocused(): boolean
  flashFrame(flag: boolean): void
}

export class TaskbarAttention {
  private known = new Set<string>()
  private flashing = false
  constructor(private target: () => FlashWindow | undefined) {}

  set(ids: readonly string[]): boolean {
    const next = new Set(ids)
    const arrived = [...next].some(id => !this.known.has(id))
    this.known = next
    if (!next.size) { this.stop(); return false }
    if (!arrived) return false
    return this.start()
  }

  focused(): void { this.flashing = false }

  private stop(): void {
    if (!this.flashing) return
    this.flashing = false
    const window = this.target()
    if (window && !window.isDestroyed()) window.flashFrame(false)
  }

  private start(): boolean {
    const window = this.target()
    if (!window || window.isDestroyed() || window.isFocused()) return false
    window.flashFrame(true)
    this.flashing = true
    return true
  }
}
```

**UI-SPEC contract:** same class, same identity-set logic (`set`/`focused`) - swap only the platform call inside `start`/`stop`. On mac: `app.dock.bounce('critical')` returns a bounce id; cancel it in the mac branch of `focused()`/`stop()` via `app.dock.cancelBounce(id)`. Do not duplicate the arrival/known-set logic per platform - branch only on `process.platform === 'darwin'` inside the existing `start`/`stop` methods, mirroring the `process.platform === 'win32'` gating pattern already used at `index.ts:188` (`configureTaskbar` call) and `index.ts:506`/`586` (`readInstallScope`/`readRegisteredInstallLocation`).

**Wiring (unchanged call sites, `index.ts`):**
```typescript
// L220
const taskbarAttention = new TaskbarAttention(() => main)
// L1196
handle('desktop:pending-attention', value => { taskbarAttention.set(attentionIdentities(value)) })
// L1339
main.on('focus', () => taskbarAttention.focused())
```
UI-04's `app.setBadgeCount(ids.length)` shares this exact call site (`desktop:pending-attention` handler) - same qualifying id set, one shared source of truth, not a second counter.

---

### UI-05: Tray Template Icon - `apps/desktop/main/index.ts`

**Analog:** itself, `runtimeIcon()` (L195-213).

**Existing icon-recoloring pattern (Windows-oriented, per-provider color) to branch around, not replace:**
```typescript
const runtimeIcon = () => {
  const current = preferences?.get() as { visualTheme?: VisualTheme; visualThemeExplicit?: boolean } | undefined
  const explicit = current?.visualTheme &&
    (current.visualTheme !== 'orgtree' || current.visualThemeExplicit === true)
    ? current.visualTheme : undefined
  const theme = effectiveTheme ?? explicit ?? 'claude'
  const name = engine.status.state === 'ready' ? trayIconNames[isCustomTheme(theme) ? 'orgtree' : theme] : trayIconNames.grey
  const image = nativeImage.createFromPath(path.join(assetsPath, name))
  if (engine.status.state === 'ready' && isCustomTheme(theme) && !image.isEmpty()) {
    const bitmap = image.toBitmap(), size = image.getSize()
    const color = theme.slice(7), rgb = [1,3,5].map(i => parseInt(color.slice(i,i+2),16))
    for (let i=0;i<bitmap.length;i+=4) { bitmap[i]=rgb[2]!; bitmap[i+1]=rgb[1]!; bitmap[i+2]=rgb[0]! }
    return nativeImage.createFromBitmap(bitmap,size)
  }
  return image.isEmpty() ? nativeImage.createFromPath(iconPath) : image
}
```

**Contract:** on `process.platform === 'darwin'`, load a single monochrome (black-on-transparent) "Orgtree eye" glyph via `nativeImage.createFromPath(...)`, call `.setTemplateImage(true)` on it before assigning to `Tray`, and skip the BGRA pixel-recoloring branch above entirely (Template images discard color). Keep `tray.setToolTip(\`Orgtree - ${label()}\`)` unchanged - it remains the carrier of provider/engine-state info once color-coding is gone from the icon itself, per UI-SPEC's flagged tradeoff. Follow the same `process.platform` gating convention as `index.ts:188` (`configureTaskbar`). A new monochrome asset file will need to be added under `apps/desktop/assets/` alongside the existing `.ico` files (identify via `ctx_glob apps/desktop/assets/*.ico` - format should probably be `.png`, not `.ico`, since macOS Template images are typically PNG).

**Existing asset naming convention** (`apps/desktop/assets/*.ico`, referenced via `trayIconNames`): `orgtree-eye-tray-{claude,codex,antigravity,openrouter,orgtree,grey}.ico` - Windows-only `.ico` format, not directly reusable for a mac Template image.

---

### UI-02: Window Lifecycle - `apps/desktop/main/index.ts`

**Analog:** itself. `show()` (L221) and main-window construction (L1323-1340).

**Current `show()`** (L221, needs `!main || main.isDestroyed()` guard added):
```typescript
const show = () => { if (main && !main.isDestroyed()) { restoreWindows = true; main.show(); if (main.isMinimized()) main.restore(); if (restoreMaximized) { restoreMaximized = false; main.maximize() }; main.focus(); broadcast({ type: 'main-window-shown', data: windowState() }) } }
```

**Existing lifecycle hooks (already correct, do not change):**
```typescript
// L1078-1079
app.on('activate', show)
app.on('window-all-closed', () => { /* Tray/main remain alive by default. */ })
```

**Main window construction to reuse for recreate path** (L1323 onward):
```typescript
main = new BrowserWindow({ width: 1400, height: 900, ...savedPlacement?.bounds, minWidth: 640, minHeight: 480, frame: false, show: false, icon: iconPath, ... })
// L1340
configureWindow(main, () => engine.origin, true, register, openArtifact, undefined, popouts.track)
```
Contract: `show()` must check `!main || main.isDestroyed()` and, if true, run the exact same construction block (`new BrowserWindow(...)` + `configureWindow(...)` + persisted-layout restore), not a second divergent path. Extract this into a helper if that keeps `show()` from growing a duplicate literal.

---

### UI-03: App Menu - `apps/desktop/main/index.ts`

**Analog:** none exists (`Menu.setApplicationMenu` is never called today) - closest structural analog is the existing tray menu builder `rebuildTray()` (L351-412), which shows the project's `MenuItemConstructorOptions` array style and click-handler wiring convention.

**Tray menu construction style to mirror** (L351-370):
```typescript
const rebuildTray = () => {
  ...
  const updateRows: Electron.MenuItemConstructorOptions[] = updatesSupported ? [
    { id: 'update-status', label: 'Updates have not been checked', enabled: false },
    { id: 'update-install', label: 'Update now', visible: downloaded, enabled: !updateApplying && !quitting,
      click: () => { void requestUpdateInstall().catch(error => { void showUpdateInstallError(error) }) } },
    { id: 'update-check', label: 'Check for updates', click: () => { void checkForUpdates() } },
  ] : [...]
}
```

**Contract:** build a `Menu.setApplicationMenu(Menu.buildFromTemplate([...]))` call (import `Menu` from `electron`) near other one-time startup calls (alongside `app.on('activate', show)` at L1078). Template: `appMenu` role, `editMenu` role, `windowMenu` role, per UI-SPEC table. "Check for Updates…" click handler must call the *same* `checkForUpdates()` function the tray row already calls (L705-706) - one implementation, two entry points, exactly like the tray's `update-check` row above. "Preferences…" (`Cmd+,`) sends an IPC message to the focused window opening the existing `SettingsPanel` - no new settings surface. "Quit Orgtree" uses `role: 'quit'`, no confirmation dialog (mirrors existing no-confirm quit behavior already in this file).

---

### UPD-01: Mac Update Notice Copy - `apps/desktop/renderer/src/update-notice.tsx`

**Analog:** itself. `LABEL` table, `updateActionTitle()`, and the button-vs-label render branch (L79-100).

**Existing state-label table and action-title helper (shared with Settings' manual-check row - do not fork wording):**
```typescript
const LABEL: Record<UpdateStatus['state'], (status: UpdateStatus) => string | null> = {
  idle: () => null,
  checking: () => 'Checking for updates…',
  downloading: status => status.percent === undefined ? 'Downloading update…' : `Downloading update… ${status.percent}%`,
  'pending-idle': status => 'Update ready to install' + (...),
  'up-to-date': () => 'You’re up to date',
  unavailable: () => 'Update check unavailable',
  failed: () => 'Update download failed',
}
export function updateActionTitle(version?: string): string {
  return version ? `Update to Orgtree ${version} — installs the downloaded update and restarts` : 'Install the downloaded update and restart Orgtree'
}
```

**Existing render branch to add a mac-specific arm to** (L79-100):
```tsx
return <div className="update-notice" role="status" aria-live="polite">
  {status.state === 'pending-idle' && desktop()?.installUpdate
    ? <button disabled={applying} className={'update-now' + (applying ? '' : ' glow')}
        title={updateActionTitle(status.version)}
        onClick={() => { setApplying(true); setError(null); void desktop()!.installUpdate().catch(...) }}>
        {applying ? 'Restarting…' : 'Update now'}</button>
    : label}
  {error && <span role="alert">{error}</span>}
</div>
```

**Contract:** on macOS, `status.state === 'pending-idle'` must never render the `Update now` auto-apply button (UPD-01 forbids offering it on mac). Add a `process.platform`-equivalent renderer check (Electron exposes this via the existing `desktop` bridge, not Node's `process` directly in the renderer - check how `desktop()` or an existing IPC-exposed platform flag surfaces this; if none exists yet, the bridge needs one) and render `Orgtree {version} available — View release` instead, with a `shell.openExternal` call routed through the main process via IPC (mirrors the existing pattern of `installUpdate()` being an IPC-exposed bridge method - add an equivalent `openReleasePage()`/reuse `openArtifact`-style IPC call). On rejection, show "Couldn't open the release page — copy the link from Check for Updates and open it manually." using the same `error` state / `<span role="alert">` pattern already in this component (L101).

The mac tray row (`Orgtree {version} available — View release`) replaces the tray's existing `update-install`/`update-automatic` rows on mac - same `rebuildTray()` `updateRows` array (`index.ts` L363-370), platform-branched the same way as `updatesSupported`.

---

## Shared Patterns

### Platform gating
**Source:** `apps/desktop/main/index.ts:188, 506, 586`
**Apply to:** UI-01, UI-05, and any other mac-only branch
```typescript
if (process.platform === 'win32') configureTaskbar(window, process.execPath, iconPath, identity.appUserModelId, identity.displayName)
// ...
if (process.platform !== 'win32' || !updatesSupported) return resolve()
```
Every existing platform branch in this codebase checks `process.platform` inline at the call site rather than a wrapper/config flag - new mac branches (UI-01's `app.dock.bounce`, UI-05's Template image) should follow this same inline-check convention, not introduce an abstraction layer.

### One implementation, two entry points (menu + tray)
**Source:** `apps/desktop/main/index.ts` - `checkForUpdates()` (L705-706) called from both the tray's `update-check` row (L370-ish) and (new) the App Menu's "Check for Updates…" item.
**Apply to:** UI-03
Never duplicate a handler body between the tray menu and the new app menu; both call sites must invoke the same named function.

### Identity-set / shared counter, not per-consumer state
**Source:** `apps/desktop/main/taskbar-attention.ts` + `index.ts:1196` (`desktop:pending-attention` handler)
**Apply to:** UI-01, UI-04
The dock bounce (UI-01) and dock badge (UI-04) must read the same qualifying identity set from the same handler - not two separately-maintained counts that could drift.

## No Analog Found

| File | Role | Data Flow | Reason |
|---|---|---|---|
| `apps/desktop/main/index.ts` (App Menu construction, UI-03) | controller | event-driven | No `Menu.setApplicationMenu` call exists anywhere in the codebase today - net-new code, structurally guided by the tray menu builder's array style, not a direct copy |
| new mac Template icon asset (UI-05) | asset | n/a | No monochrome/alpha-only tray asset exists; all existing tray icons are colored Windows `.ico` files |

## Metadata

**Analog search scope:** `apps/desktop/main/` (index.ts, taskbar-attention.ts, updater.ts), `apps/desktop/renderer/src/` (update-notice.tsx), `apps/desktop/assets/`
**Files scanned:** 4 read in full/targeted-range, 1 directory listing (assets)
**Pattern extraction date:** 2026-09-17
