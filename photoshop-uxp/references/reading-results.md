# Reading results back

The agent can't read the script's stdout (no headless mode; `console.log` goes to the UXP dev console, invisible externally). So the script's last act is to persist two artifacts:
- **preview.jpg** — flattened render the agent *looks at* (vision).
- **state.json** — document + layer tree the agent *reasons over* (names, sizes, bounds, visibility, errors).

`emit()` in [`assets/harness.psjs`](../assets/harness.psjs) writes both — just call `await emit(doc, { note })` last. The rest of this file is the *why* and the disk-finding.

## Where output lands, and how to find it

Write to the **temporary folder** — no permission prompt (arbitrary paths trigger a file picker; only a plugin with a persisted token avoids that). `getTemporaryFolder()` resolves to:
```
/var/folders/.../T/Adobe/UXP/PluginsStorage/PHSP/<version>/External/<NNNNN>/PluginData/
```
`<version>` is the PS major (tracks the installed release) and **`<NNNNN>` increments every run** — never hardcode either. The find glob below stays version-agnostic; just take the newest match by mtime:
```bash
touch /tmp/marker
APP=$(ls /Applications | grep -i '^Adobe Photoshop' | head -1)
open -a "$APP" /abs/path/script.psjs                    # returns immediately; script writes in ~1-5s
find /var/folders -path '*Adobe/UXP/PluginsStorage*External*' -name state.json -newer /tmp/marker 2>/dev/null | head -1
```
Read `preview.jpg` + `state.json` from that dir. Always take the newest match.

## Preview render

```js
await doc.saveAs.jpg(file, { quality: 9 }, /* asCopy */ true);   // copy → never disturbs the doc
// lossless / transparency: saveAs.png(file, {}, true)
```
The export is document pixels, not the viewport — exactly what you want for review.

## Errors are data

Don't swallow failures — `bp()` (harness) pushes the Photoshop error shape into `errors[]`, which `emit()` writes into state.json. The agent reads it and adjusts instead of re-running blind. Remember: `errors: []` only means nothing threw — judge the preview.

## Batch

Same contract per file:
```js
for (const token of inputFileTokens) {
  const doc = await app.open(token);
  // ...edit...
  await doc.saveAs.jpg(await out.createFile(`${doc.title}.jpg`, { overwrite: true }), { quality: 9 }, true);
  await doc.closeWithoutSaving();
}
```
