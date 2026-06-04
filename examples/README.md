# Examples

Three self-contained `.psjs` scripts, each built on the canonical harness
(`../photoshop-uxp/assets/harness.psjs`) and each producing the `preview.jpg`
shown below — **real renders from Photoshop, committed next to the code**.

Each uses a **deliberately different typographic voice** — editorial serif,
bold geometric, cinematic mono — to show the range the API actually covers
(six faces across the three: Didot, Baskerville, Futura, Helvetica Neue, Menlo).

Every script ends by emitting `preview.jpg` + `state.json`, then I looked at the
preview and adjusted — the loop the skill insists on. (Example 01 first rendered
black-on-navy text at 12pt: proof that `errors: []` means nothing.)

Run any of them (Photoshop must already be running):

```bash
APP=$(ls /Applications | grep -i '^Adobe Photoshop' | head -1)
open -a "$APP" "$(pwd)/01-new-poster/poster.psjs"
```

The image scripts (`02`, `03`) place a photo, which needs an **absolute** path —
edit the `INPUT` constant near the top of those files to point at the `source.jpg`
in their folder.

---

### 01 — Editorial poster from nothing · *serif voice*
`createDocument` · solid fill layers · a hairline rule · **real type** — Didot
display, tracked Helvetica label, Baskerville italic caption — with explicit
font + size + tracking + color on a warm-paper palette.

![poster](01-new-poster/preview.jpg)

### 02 — Place a photo into a cover · *bold geometric voice*
Place an external image via a session token · scale-to-**cover** a band +
recenter · a flat color band instead of a muddy scrim · heavy condensed Futura
caps over oxblood. Same mechanics as 01, completely different look.

![cover](02-place-cover/preview.jpg)

### 03 — Grade a photo: non-destructive B&W · *cinematic / mono voice*
Place full-bleed · **non-destructive adjustment layers** — Black & White
(weighted mix), contrast Curves, and a vignette (a darkening Curves masked with
a radial gradient) · an ultra-thin Helvetica title over a Menlo metadata line.
The original pixels are never touched.

![grade](03-photo-grade/preview.jpg)

---

**What makes these look modern, not the dated first drafts:** a real display
face (not the PS default), generous margins, one restrained accent, and big type
with breathing room. None of that is in the API — it comes from rendering,
looking, and adjusting. Photo credits: [picsum.photos](https://picsum.photos).
