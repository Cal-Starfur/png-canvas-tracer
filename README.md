# png-canvas-tracer

Minimal, launcher-style tool for turning flat-illustrated PNGs into HTML5
canvas `drawArt(canvas)` code — no build step, no framework, single-file
vanilla JS, iOS-friendly. Structured the same way as
[rocket-launcher](https://github.com/Cal-Starfur/rocket-launcher): one
`index.html`, a bundled `examples/` folder, and a manual file-picker for
anything not in the bundle.

**Live:** https://cal-starfur.github.io/png-canvas-tracer/

## What it does

Pick a PNG (or tap a bundled example) and it:

1. Clusters the image's real flat colors (ignoring anti-aliasing noise)
2. Traces each color region's actual pixel boundary into a clean polygon
3. Renders the result live next to the original, with a **foreground-only**
   accuracy score (only counts pixels where either image actually has
   content — a whole-canvas average would let background agreement mask
   a genuinely bad trace)
4. Lets you tune color count, contour smoothness, and per-layer draw order
5. Copy or download the generated `drawArt(canvas)` code

## Honest accuracy expectations

Flat, simple-silhouette art (a walnut shell, an apple core) traces to
roughly **80-99%** foreground accuracy. Textured or many-color art
(broccoli's floret shading, a multi-topping pizza) tops out lower —
often 50-75% — because a handful of flat vector fills can't reproduce a
gradient or fine internal shading, no matter how the parameters are
tuned. The tool is honest about this: don't expect 95%+ on anything with
real internal texture.

## Adding your own examples

The examples grid is driven entirely by `examples/manifest.json` —
adding a new set never requires touching `index.html`.

1. Create a new folder: `examples/<your-folder-name>/`
2. Drop your PNGs in it (flat-illustrated art works best; photos and
   complex gradients won't trace well)
3. Add one entry to `examples/manifest.json`:

```json
{
  "folder": "your-folder-name",
  "label": "Your Display Label",
  "files": ["filename_one", "filename_two"]
}
```

(filenames without the `.png` extension — the tool appends it). That's
the whole contribution — no code changes, no build step. Open a PR.

## Repo structure

```
index.html                          — the whole app
examples/manifest.json              — category list, drives the examples grid
examples/trash-chunks/*.png         — 27 game-asset PNGs from Wigglers_Room,
                                       covering a spread of silhouette
                                       complexity (walnut_shell.png is simple,
                                       pizza.png and broccoli.png are busy)
```

## Implementation notes

Pure JS pipeline, zero dependencies, runs entirely client-side:

1. **Color clustering** — coarse-quantize visible pixels (alpha > 20),
   histogram-count, greedily pick the top N distinct colors, merging
   near-duplicates from anti-aliasing (merge threshold tuned to avoid
   splitting a single real color into spurious near-duplicate clusters —
   an earlier over-tightened threshold did this and caused visible
   z-order artifacts).
2. **Per-color labeling** — assign every visible pixel to its nearest
   cluster center; clusters below ~1% of the image's foreground area are
   dropped so stray anti-aliasing fragments can't become their own layer.
3. **Connected components** — flood-fill each color's binary mask into
   separate blobs.
4. **Moore-Neighbor boundary tracing** — walk each blob's actual pixel
   edge to get an ordered contour (start at the topmost-then-leftmost
   pixel; its West neighbor is guaranteed background, giving a reliable
   initial backtrack direction).
5. **Ramer-Douglas-Peucker simplification** — collapse the contour to a
   clean polygon. Epsilon 0.4 is the default; lower values (down to
   ~0.1-0.3) trace with more fidelity at the cost of more points, and
   this made a measurable accuracy difference (0.8 → 0.4 alone moved
   simple assets from ~79% to ~80%+).
6. **Code generation** — one `fillStyle` + `beginPath/moveTo/lineTo/fill`
   block per color region, in a user-adjustable draw order (layer
   up/down controls in the UI, since largest-area-first isn't always
   the correct z-order for overlapping detail colors).
7. **Scoring** — render the generated code to an offscreen canvas and
   compare against the source via `getImageData`, foreground-only.

**Dead ends worth knowing about** (so nobody re-tries them expecting a
quick win): anti-aliased/coverage-based rendering of the traced polygons
scored *worse* than a hard flat-color fill, and a full marching-squares
sub-pixel silhouette contour (used as a canvas clip) did too — both
because the source PNG's real anti-aliasing isn't a clean geometric
function of distance-to-edge, so trying to reproduce it geometrically
doesn't get closer to the source's actual pixel values. The genuine
accuracy win came from contour fidelity (epsilon), not rendering
sophistication.

This whole approach replaced an earlier one that called the Claude
vision API in a guess-and-check loop — that plateaued around 80-90%
*apparent* similarity (using an inflated whole-canvas scoring formula)
on anything with an organic/irregular silhouette, because each iteration
re-guessed the shape from a text description instead of the actual pixel
boundary. Tracing real pixels instead is deterministic and doesn't have
that ceiling.
