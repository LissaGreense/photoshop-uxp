---
name: photoshop-uxp
description: Source-of-truth for driving Adobe Photoshop programmatically via UXP scripting (.psjs scripts using the DOM and batchPlay). Use whenever the user wants to automate, generate, edit, retouch, or inspect a Photoshop document — make a poster, social post, blog or cover image, banner, or graphic; add, style, move, or group layers, text, shapes, masks, and smart objects; place external images; apply non-destructive photo adjustments (curves, black and white, vignette, color grade); fill solid or gradient backgrounds; or batch-process and export. Covers how scripts run and trigger, the DOM/batchPlay API, sizing sense, and reading results back so an agent can verify its own work. Photoshop only — Lightroom uses a different stack.
---

# Driving Photoshop with UXP

A `.psjs` script (Photoshop 23.5+) is one JS file that talks to Photoshop two ways:
- **DOM** — `require("photoshop").app`: documents, layers, text. Covers the common ~80%.
- **batchPlay** — `require("photoshop").action.batchPlay`: everything else, as *action descriptors* (actionJSON), and the channel to read state back.

Photoshop only. Lightroom Classic = Lua SDK, Lightroom cloud = REST — these patterns don't apply there.

## Non-negotiables

- **A `.psjs` is already modal.** Call DOM/batchPlay directly; top-level `await` is fine. **Never** wrap edits in `core.executeAsModal` — that's plugins only. (Copying a plugin example? Strip the wrapper.)
- **Start from [`assets/harness.psjs`](assets/harness.psjs).** It ships the `bp()` batchPlay wrapper and `emit()` readback; put your work in the marked block. Don't re-derive boilerplate.
- **`errors: []` ≠ correct.** Fills no-op, text lands black-on-black, images place off-canvas — all with no error. **Only the rendered pixels are truth.** Every non-trivial job is a loop: attempt → export preview → *look* → fix the specific problem → re-render. Keep edits parameterized so re-running is cheap.

## Intent routing — pick the target document first, before any edit

| User says | Target |
|---|---|
| "this image", "the open file", nothing specified | `app.activeDocument` — throw if `app.documents.length === 0` |
| "open X", a path given | `await app.open(await fs.getEntryWithUrl("file:/abs/path"))` |
| "make / create a new…", a size given | `await app.createDocument({ width, height, resolution: 72, fill })` |
| many files | loop: open → edit → export → close |

Ambiguous (open doc *or* new)? **Ask** — wrong document burns a whole iteration.

## Sizing sense (so attempt #1 isn't amateur)

Work in proportions of the canvas height `H`, not magic pixels:
- **Margins:** keep content ≥ **5–8%** of the short edge off every edge.
- **Type scale:** headline ≈ **7–12% of H** and clearly the largest; subhead ≈ **3–4%**; caption ≈ **1.5–2.5%**. Keep ≥ ~1.8× jumps between levels.
- **Line length:** cap ~**20–35 chars/line**. `\n` does NOT break lines (tofu box) — use **one text layer per line**.
- **Contrast:** never trust a photo's luminance — put a scrim/band behind text, set text color explicitly, verify in preview.
- **Composition:** one consistent alignment, group related lines, one focal point. Scattered = accidental.

Starting points, not laws — render and adjust.

## Reference map

- [layers-and-composition.md](references/layers-and-composition.md) — layers, text, groups, masks, smart objects, transforms; solid/gradient **fills**; **placing images**; **adjustment layers** (curves, B&W, vignette/local grades); **selections** (combining, non-rect shapes). The main surface.
- [batchplay.md](references/batchplay.md) — descriptor anatomy, capturing descriptors, reading returns.
- [reading-results.md](references/reading-results.md) — preview + state JSON, finding output on disk.
- [running-scripts.md](references/running-scripts.md) — script vs plugin, the `open -a` trigger.
- [gotchas.md](references/gotchas.md) — modal scope, macOS networking, units, history, async. Check before debugging.

## Checklist

```
- [ ] Target document resolved explicitly (active / open / new); guarded for "none open"
- [ ] No core.executeAsModal (this is a script)
- [ ] DOM where it exists, batchPlay where it doesn't
- [ ] Multi-step edits wrapped in history suspension (undo as one unit)
- [ ] emit() preview + state at the end; rendered and actually looked at
```
