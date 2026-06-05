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

## The structured channel: `readback()`

The preview is the pixel truth; `state.json` is the structured truth — and they catch different failures. A scrim that's slightly too dark only shows in the preview; a headline that rendered invisible (black fill on a black band) is obvious in `state.json` the instant you see `"textColor": {"red":0,"green":0,"blue":0}`. **Read both every iteration.**

`snapshotLayer()` already records each layer's bounds, opacity, blend mode, and — for text layers — contents **and fill color** (the classic black-on-black trap, now machine-checkable instead of eyeball-only).

For anything the DOM doesn't surface cleanly, `readback()` is the escape hatch — the same `multiGet` get-descriptor Alchemist/ScriptListener use, distilled to one call:
```js
const layerRef = [{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" },
                  { _ref: "document", _enum: "ordinal", _value: "targetEnum" }];
const got = await readback(layerRef, ["opacity", "bounds", "layerID"], "active layer");
```
Point the `_ref` chain at anything — `application`, `document`, `layer` (by `_id`, `_index`, `_name`, or ordinal), `historyState`, `channel`. Same call, different target; that's the whole "inspect from various places" trick. Fold the result into `emit(doc, { readback: got })` so it lands in `state.json` next to the rest.

**This is raw AM, not DOM — two traps (verified by probe):**
- **Values are unit-wrapped**, not bare numbers. `got.width` is `{_unit:"distanceUnit", _value:800}`, not `800`; `bounds` is `{_obj:"rectangle", top:{_unit,_value}, left:{…}, …}`; `mode` is `{_enum:"colorSpace", _value:"RGBColor"}`. Read `._value`, don't compare the object.
- **`opacity` is 0–255 here**, not the DOM's 0–100 (a fully opaque layer reads `255`). Don't reuse a DOM threshold.

So `readback()` is the right tool for *presence/identity* checks (does this layer exist, what's its id/name) and exact geometry once you unwrap `._value`. For everyday opacity/bounds/blend/text-color, the DOM-based `snapshotLayer()` fields already in `state.json` are flatter and easier — reach for `readback()` when the DOM doesn't expose what you need.

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
