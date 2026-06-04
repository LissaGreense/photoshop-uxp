# Gotchas — check before debugging anything weird

**Modal scope (#1 confusion).** `.psjs` is already modal → call DOM/batchPlay directly. Plugins are not → wrap mutations in `core.executeAsModal(fn, { commandName })`. Symptoms of a stray `executeAsModal` in a script: "Photoshop is busy" / "modal state" errors, or silent no-ops.

**History — make a run atomic.** Wrap multi-step edits so they undo as one step and roll back on failure:
```js
const { executionContext } = require("uxp").script;
const id = app.activeDocument.id;
await executionContext.hostControl.suspendHistory({ documentID: id, name: "Agent: build comp" });
try { /* edits */ } finally { await executionContext.hostControl.resumeHistory(id); }  // always resume
```
(In a plugin, `executionContext` is the `executeAsModal` callback arg; same `hostControl`.)

**Success ≠ effect.** A batchPlay call can return no error and change nothing (e.g. `Edit > Fill` on a fresh empty layer). `errors: []` only means nothing threw. Verify by rendering and looking; prefer constructs that can't silently no-op (solid-color **fill layers** over empty-layer+fill).

**Async.** Almost every call is async — `await` it, or it races / appears to do nothing. One script at a time; a script can't launch another.

**Tabs piling up?** Every `createDocument`/`open` leaves a document open. Closing what you created (vs. leaving the user's doc) is workflow, not a footnote — see SKILL.md "how you acquired the doc decides teardown."

**Units & coordinates.** DOM geometry is pixels at the doc's resolution; `bounds` = `{left,top,right,bottom}`, origin top-left. Many batchPlay descriptors need a unit object — `{ _unit: "pixelsUnit", _value: 100 }` (also `percentUnit`, `angleUnit`, `densityUnit`); a raw number where a unit object is expected silently fails. Read `resolution` before computing positions (100px @ 300dpi ≠ @ 72dpi).

**Filesystem.** `fs.getTemporaryFolder()` and the plugin folder are prompt-free; **arbitrary paths prompt a picker** (use a plugin + `fs.createPersistentToken` for fixed locations). File-referencing descriptors (place, save-with-path) need a **session token**: `fs.createSessionToken(entry)`.

**Dialogs.** Keep silent: `_options: { dialogOptions: "dontDisplay" }`. **Never** trigger an alert/confirm/prompt — it freezes unattended automation with no one to click.

**macOS networking.** UXP `fetch`/WebSocket: **https only** on macOS (`http` is Win32-only); a localhost relay must be wss/https. Plugins declare hosts in `manifest.json` → `requiredPermissions.network.domains` (no top-level wildcards since UXP 7.4.0).

**Version.** Needs Photoshop 23.5+; `.psjs`, `createTextLayer`, `createLayerGroup` arrived across later releases — check the [API changelog](https://developer.adobe.com/photoshop/uxp/2022/ps_reference/changelog/) if a method is missing. This is the public extensibility API, not Adobe agent tooling; `adb-mcp`/Alchemist fill gaps.
