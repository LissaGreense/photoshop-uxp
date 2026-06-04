# Layers & composition

DOM first; batchPlay ([batchplay.md](batchplay.md)) for what it lacks (fills, adjustments, masks, styles, transforms, place). All examples are `.psjs` (no `executeAsModal`). `const { app, action, constants } = require("photoshop");`

## Documents & layers

```js
const doc = await app.createDocument({ width: 1920, height: 1080, resolution: 72, fill: "white" });
// fill: "white" | "black" | "transparent" | "backgroundColor"
const doc = app.activeDocument;                 // front doc; guard: if (app.documents.length === 0) throw ...

await doc.createLayer({ name: "paint", opacity: 100, blendMode: constants.BlendMode.NORMAL });   // pixel
await doc.createTextLayer({ contents: "Headline", fontSize: 64, position: { x: 120, y: 200 }, name: "h" });
await doc.createLayerGroup({ name: "hero" });
```

Inspect: `doc.layers` (array-like, top→bottom), `layer.{name,kind,visible,opacity,blendMode,bounds}`, `layer.kind` ∈ `constants.LayerKind.*`. `doc.layers.find(l => l.name === "h")`. Make active: `doc.activeLayers = [layer]` (many batchPlay ops act on the active layer).

Move/order: `layer.translate(dx, dy)`, `await layer.duplicate()`, `layer.delete()`, `layer.move(target, constants.ElementPlacement.PLACEINSIDE)` (also `PLACEBEFORE/AFTER/ATBEGINNING/ATEND`). Style: set `layer.{opacity,blendMode,visible,name}` directly.

## Text

Entry = `layer.textItem`: `.contents`, `.characterStyle.size`. Three traps:
- **Multiline:** `\n`/`\r` do NOT break lines (tofu box, stays one line) → **one text layer per line**.
- **Tofu is general:** any glyph the font lacks (→ — • emoji, fancy quotes) renders as a box. Use ASCII, confirm the font has it, or **draw the decoration as a shape** (see Selections).
- **Nothing is inherited** — `createTextLayer` defaults to **black**, the default UI font, ~12pt. Set font + size + tracking + color **explicitly** in one `textStyleRange`. Black-on-dark (and tiny-default-size) is the #1 "where's my text" bug.

Set everything at once — and beware a partial `textStyleRange` **replaces** the range, so omitting `size` snaps it back to ~12pt:
```js
await doc.createTextLayer({ contents: txt, position: { x, y }, name });
await bp([{ _obj: "set", _target: [{ _ref: "textLayer", _enum: "ordinal", _value: "targetEnum" }],
  to: { _obj: "textLayer", textStyleRange: [{ _obj: "textStyleRange", from: 0, to: txt.length,
    textStyle: { _obj: "textStyle",
      fontPostScriptName: "Didot", fontName: "Didot",          // a REAL face — the default looks dated
      size: { _unit: "pointsUnit", _value: 92 },               // carry size or it resets to ~12pt
      tracking: -10,                                           // 1/1000 em; negative tightens display type
      color: { _obj: "RGBColor", red: 26, grain: 25, blue: 22 } } }] } }], "style");
```
Font = exact **PostScript name** (`Didot`, `HelveticaNeue-Bold`, `Baskerville-Italic`, `Georgia`); a missing face silently falls back. Picking a real display face + tracking + generous size is most of what makes output look designed rather than default. Capture exact descriptors via Alchemist when unsure.

## Solid & gradient fills — use a fill layer

Don't make an empty pixel layer + `Edit > Fill` — that **silently no-ops** (returns success, paints nothing). Fill layers paint reliably and stay editable.

```js
await bp([{ _obj: "make", _target: [{ _ref: "contentLayer" }],
  using: { _obj: "contentLayer", type: {
    _obj: "solidColorLayer",
    color: { _obj: "RGBColor", red: 30, grain: 136, blue: 229 } } } }], "bg fill");
const fillLayer = app.activeDocument.activeLayers[0];
fillLayer.opacity = 40;
```
**RGB quirk:** `RGBColor` keys are `red`, **`grain`** (green!), `blue`. Not a typo.

**Region/band:** make a selection first → the fill layer **auto-masks to it**:
```js
await bp([{ _obj: "set", _target: [{ _ref: "channel", _property: "selection" }],
  to: { _obj: "rectangle", top:{_unit:"pixelsUnit",_value:380}, left:{_unit:"pixelsUnit",_value:0},
        bottom:{_unit:"pixelsUnit",_value:700}, right:{_unit:"pixelsUnit",_value:1080} } }], "rect");
// ...make solidColorLayer (masks to rect)... then deselect: to:{_enum:"ordinal",_value:"none"}
```
Gradient background = same `make`, with a `gradientLayer` content type (colorStop `location` on a **0–4096** scale, separate `transparency[]` transferSpec array).

## Selections beyond rectangles

`set` on the selection channel **replaces** (a `selectionModifier` inside it is silently ignored). Combine with separate commands:

| `_obj` | effect |
|---|---|
| `set` | replace |
| `addTo` | union |
| `subtractFrom` | subtract |
| `intersectWith` | intersect |

All target `[{_ref:"channel",_property:"selection"}]`. Compound shape = `set` first piece, `addTo` the rest, then a fill layer (masks to the combined selection). Triangle/arrowhead = stack tapering rectangle rows, `addTo` each (a "staircase"), no rotation. **Render to confirm the shape formed** — a wrong combine yields a 1px speck.

## Adjustments — non-destructive grading

Same `make` pattern as fills, targeting `adjustmentLayer`. Non-destructive (originals untouched — the right default when told not to alter).

```js
await bp([{ _obj: "make", _target: [{ _ref: "adjustmentLayer" }],
  using: { _obj: "adjustmentLayer", type: {
    _obj: "blackAndWhite", red:40, yellow:50, green:40, cyan:50, blue:20, magenta:70 } } }], "bw");
```
Types: `brightnessContrast`, `levels`, `curves`, `hueSaturation`, `blackAndWhite`, `colorBalance`, `photoFilter`, `selectiveColor`, `channelMixer`, `vibrance`, `exposure`. Don't memorize param blocks — make it, render, tune by eye.

**Curves:** `adjustment:[{ _obj:"curvesAdjustment", channel:{_ref:"channel",_enum:"channel",_value:"composite"}, curve:[{_obj:"paint", horizontal:<in 0-255>, vertical:<out 0-255>}, ...] }]`. Lift point 1's `horizontal` → crushed blacks; S-shape → contrast.

**Local edits via the built-in mask** (vignette, graduated, dodge/burn) — `select` the layer's mask **channel**, paint, reselect RGB:
```js
await bp([{ _obj: "select", _target: [{ _ref: "channel", _enum: "channel", _value: "mask" }] }], "mask");
// paint a radial gradientClassEvent into it: center black = adjustment hidden, edge white = shown (a vignette)
await bp([{ _obj: "select", _target: [{ _ref: "channel", _enum: "channel", _value: "RGB" }] }], "rgb");
```
Use `select` on the mask channel — NOT `set`-as-selection (that just loads marching ants, won't redirect paint). On a textured photo a vignette can be **invisible if edges are already dark** — prove it on a flat field, then judge the real image; if inverted, swap the stops.

## Other batchPlay gaps (capture via Alchemist)

```js
// add a layer mask
await bp([{ _obj: "make", _new: "channel", at: { _ref: "channel", _enum: "channel", _value: "mask" },
  using: { _enum: "userMaskEnabledOptions", _value: "revealAll" } }], "mask");   // or "hideAll"
// active layer(s) → Smart Object
await bp([{ _obj: "newPlacedLayer" }], "smartobj");
// group selected layers (layerSection = group)
await bp([{ _obj: "make", _target: [{ _ref: "layerSection" }],
  from: { _ref: "layer", _enum: "ordinal", _value: "targetEnum" } }], "group");
```
- **Layer styles** (shadow/stroke/overlay): `set` the layer's `layerEffects` — large descriptors, capture them.
- **Transform** (scale/rotate/skew): `transform` descriptor. `translate` is the only DOM move.

**Place an external image** — `placeEvent` + a **session token**. Lands centered at native size as a Smart Object → scale-to-cover and recenter:
```js
const entry = await fs.getEntryWithUrl("file:/abs/path/image.jpg");   // single slash after file:
const token = await fs.createSessionToken(entry);
await bp([{ _obj: "placeEvent", null: { _path: token, _kind: "local" },   // token under the literal key "null"
  freeTransformCenterState: { _enum: "quadCenterState", _value: "QCSAverage" } }], "place");
const img = app.activeDocument.activeLayers[0];
const b = img.bounds, pct = Math.max(W/(b.right-b.left), H/(b.bottom-b.top)) * 100;   // Math.min = fit
await bp([{ _obj: "transform", _target: [{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" }],
  freeTransformCenterState: { _enum: "quadCenterState", _value: "QCSAverage" },
  width: { _unit: "percentUnit", _value: pct }, height: { _unit: "percentUnit", _value: pct },
  interfaceIconFrameDimmed: { _enum: "interpolationType", _value: "bicubic" } }], "cover");
const nb = img.bounds;
img.translate(W/2 - (nb.left+nb.right)/2, H/2 - (nb.top+nb.bottom)/2);
// then render and confirm it covered — placement/scale is a classic silent-no-op
```

Wrap multi-step edits in history suspension so they undo as one unit — see [gotchas.md](gotchas.md).
