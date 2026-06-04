# Examples

Three self-contained `.psjs` scripts, each built on the canonical harness
(`../photoshop-uxp/assets/harness.psjs`). Each produces the `preview.jpg` shown
below — committed next to its script — and uses a different typographic voice to
demonstrate the range the API covers (Didot, Baskerville, Futura, Helvetica
Neue, Menlo across the three).

## Running

Photoshop must already be running. From this folder:

```bash
APP=$(ls /Applications | grep -i '^Adobe Photoshop' | head -1)
open -a "$APP" "$(pwd)/01-new-poster/poster.psjs"
```

Each script writes its output (`preview.jpg` + `state.json`) to Photoshop's temp
folder and closes the document it created. The image examples (`02`, `03`) place
a photo, which requires an **absolute** path — set the `INPUT` constant near the
top of those scripts to the `source.jpg` in their folder.

---

### 01 — Editorial poster · *serif voice*
Create a document, paint solid fill layers, draw a hairline rule, and set real
type (Didot display, tracked Helvetica label, Baskerville italic caption) with
explicit font, size, tracking, and color on a warm-paper palette.

![poster](01-new-poster/preview.jpg)

### 02 — Photo cover · *bold geometric voice*
Place an external image via a session token and scale it to **cover** the frame.
The sharp subject sits low, so the crop is **anchored to the bottom** to keep it,
and the caption strip goes up top over the soft area — covering nothing — with a
masthead title in heavy condensed Futura caps.

![cover](02-place-cover/preview.jpg)

### 03 — Non-destructive black & white · *cinematic / mono voice*
Place a photo full-bleed, then apply **adjustment layers** — Black & White
(weighted mix), contrast Curves, and a vignette (a darkening Curves masked with
a radial gradient) — leaving the original pixels untouched. Captioned with an
ultra-thin Helvetica title over a Menlo metadata line.

![grade](03-photo-grade/preview.jpg)

---

Photo sources: [picsum.photos](https://picsum.photos).
