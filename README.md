# photoshop-uxp

A **source-of-truth Agent Skill** for driving Adobe Photoshop programmatically via [UXP scripting](https://developer.adobe.com/photoshop/uxp/) (`.psjs` scripts using the DOM and `batchPlay`).

It gives an AI agent everything it needs to automate, generate, edit, retouch, or inspect a Photoshop document — and, crucially, to **read its own results back** so it can verify and self-correct in a render → look → adjust loop.

## Why this exists

I don't like wasting time nudging layers around in Photoshop. I'd rather tell an agent "make the cover" and let it do the pixel-pushing.

Problem is, Photoshop has almost no agent tooling. UXP scripting is the modern native path, but it's full of silent foot-guns: fills that no-op, text that lands black-on-black, images that place off-canvas — all returning `errors: []`. So I baked the hard-won facts and a result-readback convention into one skill, so the agent doesn't fly blind.

**Core principle:** `errors: []` ≠ correct. Only the rendered pixels are truth. Every non-trivial job is a loop — attempt → export preview → *look* → fix → re-render.

## Gallery

All three are **real Photoshop renders** produced by the scripts in [`examples/`](examples/) — each a different design language to show the range. Code + output committed side by side.

| Editorial serif | Bold geometric | Cinematic B&W |
|---|---|---|
| ![](examples/01-new-poster/preview.jpg) | ![](examples/02-place-cover/preview.jpg) | ![](examples/03-photo-grade/preview.jpg) |
| generate from nothing | place + cover a photo | non-destructive grade |

## What's inside

```
photoshop-uxp/
├── SKILL.md                          # entry point: API model, non-negotiables, intent routing, sizing sense
├── assets/
│   └── harness.psjs                  # canonical preamble: bp() batchPlay wrapper + emit() readback
├── references/
│   ├── layers-and-composition.md     # layers, text, fills, image placement, adjustments, selections, masks, smart objects
│   ├── batchplay.md                  # descriptor anatomy, capturing & reading descriptors
│   ├── reading-results.md            # preview.jpg + state.json, finding output on disk
│   ├── running-scripts.md            # script vs plugin, the `open -a` autonomous trigger
│   └── gotchas.md                    # modal scope, units, history, async, macOS networking
└── evals/
    └── evals.json                    # rendered-output test cases
```

`photoshop-uxp.skill` is the packaged artifact.

## Key facts it encodes

- A `.psjs` script is **already modal** — call DOM/batchPlay directly, never wrap edits in `core.executeAsModal` (that's plugins only).
- **Autonomous trigger:** `open -a "$(ls /Applications | grep -i '^Adobe Photoshop' | head -1)" /abs/path/script.psjs` executes the script with no manual clicking; Photoshop must be running.
- **Readback:** scripts can't return stdout, so they emit `preview.jpg` (vision) + `state.json` (layer tree, errors) to the temp folder; the agent finds the newest by mtime.
- Validated `batchPlay` recipes for solid/gradient fills, image placement (session tokens), non-destructive adjustments (curves, B&W, vignette), compound selections, masks, and smart objects — including quirks like the `RGBColor` `grain` (green) key and the `\n`-renders-as-tofu trap.
- Sizing heuristics (margins, type scale, line length, contrast) so the first attempt isn't amateur.

## Install

Via [skills.sh](https://skills.sh) / the `skills` CLI:

```bash
npx skills add LissaGreense/photoshop-uxp
```

Or grab it manually — drop the `photoshop-uxp/` directory into your agent's skills folder (e.g. `.claude/skills/`), or install the packaged `photoshop-uxp.skill`. The agent loads `SKILL.md`, then pulls in references progressively as a task demands.

## Requirements

- Photoshop 23.5+ (any recent release — the patterns aren't tied to a specific year)
- macOS examples use `open -a`; the patterns transfer to Windows with the equivalent launch

## Status

Built and validated end-to-end against a live Photoshop install. Recipes are captured from real executions, not guessed.
