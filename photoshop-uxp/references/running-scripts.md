# Scripts vs plugins, and triggering

## Which form

| | **`.psjs` script** | **UXP plugin** |
|---|---|---|
| Form | one JS file | folder + `manifest.json` |
| Modal | **already modal** — no `executeAsModal` | not modal — wrap every mutation in `core.executeAsModal` |
| API | LocalFileSystem, Fonts, Clipboard + full DOM & batchPlay | **all**, incl. **network** (fetch/XHR/WebSocket) |
| Lifetime | runs once, exits | persistent; can listen/poll |
| Concurrency | one at a time; can't invoke another script | full event model |

Use a `.psjs` for "do this edit, produce output." Use a **plugin** only for network or a persistent listener. Agent-generates-a-script-per-task → `.psjs`.

## Trigger autonomously (no headless mode needed)

```bash
open -a "Adobe Photoshop 2025" /abs/path/script.psjs    # app name must match the install
```
`open -a` sends the same open-document event as dropping the file on the app icon → Photoshop **executes** the `.psjs` (~1–5s). Photoshop must already be running. `open` returns immediately and does NOT wait — poll for output by mtime ([reading-results.md](reading-results.md)). Re-running re-executes in a fresh context.

Manual alternatives: File → Scripts → Browse; drag onto the dock icon; play from an Action.

## Other transports (when `open -a` doesn't fit)

- **Persistent plugin** (live bridge): a plugin opens a WebSocket/fetch **client** to a local relay (the `adb-mcp` model). UXP is never a socket *server*, so a relay always sits in the middle. For long-lived sessions vs one-shots.
- **AppleScript → Action → `.psjs`**: works, but PS 2025+ is deprecating AppleScript — prefer `open -a`.

## macOS networking gotcha

UXP `fetch` allows **https only** on macOS (`http` is Win32-only). A localhost relay must be **wss**/https — `http://localhost:PORT` fails on Mac. This is why bridges exist.
