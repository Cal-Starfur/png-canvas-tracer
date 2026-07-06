# png-canvas-tracer

Minimal, launcher-style tool for turning flat-illustrated PNGs into HTML5
canvas `drawArt(canvas)` code — no build step, no framework, single-file
vanilla JS, iOS-friendly. Structured the same way as
[rocket-launcher](https://github.com/Cal-Starfur/rocket-launcher):

- `index.html` — the whole app, one file, minimal chrome
- `examples/` — bundled PNGs the tool can load directly by relative path
  (same pattern as rocket-launcher's `games/` folder + `FEATURED_GAMES`
  array), plus a manual file-picker for anything not in the bundle

## Status

**Framework only right now.** `index.html` is a placeholder. The actual
tracer (color clustering → connected components → Moore-Neighbor contour
tracing → Ramer-Douglas-Peucker simplification → canvas code generation
→ live similarity score) is designed and validated but not yet wired in
here — see the reference implementation notes below when it's time to
build it.

## Examples

`examples/` contains 27 trash-chunk PNGs pulled from
[Wigglers_Room](https://github.com/Cal-Starfur/Wigglers_Room)'s
`docs/art/trash-chunks/` — real game assets to test the tracer against
once it exists, covering a good spread of silhouette complexity (simple
blobs like `walnut_shell.png` up to busy ones like `pizza.png` and
`broccoli.png`).

## Reference implementation notes (for when the tool gets built)

The core pipeline, already validated in a Node.js prototype at ≥97%
pixel-similarity on test assets, pure JS with zero dependencies:

1. **Color clustering** — coarse-quantize visible pixels (alpha > 20),
   histogram-count, greedily pick the top N distinct colors, merging
   near-duplicates from anti-aliasing.
2. **Per-color labeling** — assign every visible pixel to its nearest
   cluster center.
3. **Connected components** — flood-fill each color's binary mask into
   separate blobs.
4. **Moore-Neighbor boundary tracing** — walk each blob's actual pixel
   edge to get an ordered contour (start at the topmost-then-leftmost
   pixel; its West neighbor is guaranteed background, giving a reliable
   initial backtrack direction).
5. **Ramer-Douglas-Peucker simplification** — collapse the contour to a
   clean polygon (epsilon ~0.8px works well at icon scale).
6. **Code generation** — one `fillStyle` + `beginPath/moveTo/lineTo/fill`
   block per color region, in a chosen draw order (layer order needs a
   simple UI control — largest-area-first is a reasonable default but
   not always correct for overlapping detail colors).
7. **Scoring** — render the generated code back to an offscreen canvas
   and pixel-diff against the source image via `getImageData`, entirely
   client-side.

This replaced an earlier approach that called the Claude vision API in a
guess-and-check loop — that plateaued around 80-90% similarity on
anything with an organic/irregular silhouette, because each iteration
re-guessed the shape from a text description instead of the actual pixel
boundary. The pipeline above traces real pixels instead, so it's
deterministic and hits ≥95% on the first pass on flat-illustrated art.
