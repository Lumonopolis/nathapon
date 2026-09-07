---
title: "Workshop #1: First Runs"
date: 2026-09-07T13:00:00+09:00
tags: ["workshop", "training", "renderer", "lightroom", "qwen3-vl"]
draft: false
---

*Late August to early September 2026. Setup, renderer, three training runs, and what they showed. Technical notes from the machinery side of the project; the narrative is in [Journal #4](/journal/two-hands/).*

## Data

- **Corpus:** 1,414 raw files with the photographer's Lightroom develop settings (XMP), harvested through the Lightroom partner API. Cameras: Fujifilm X100VI, X-E5, X-T5; Ricoh GR III; Leica D-Lux 8. 203 of the 1,414 are matched to published Instagram posts and carry a `published` tier label; the rest are `edited`. (The catalog as a whole has 219 published matches; 16 are JPEG originals outside this corpus.)
- **Labels:** the develop XMP is the label. It carries global tone (exposure, contrast, highlights, shadows, whites, blacks, texture, clarity, dehaze), point curves, HSL, colour grading, calibration primaries, grain and vignette, crop and rotation, upright/perspective, the camera profile or film-simulation Look, and parametric masks (linear and radial gradients with their local adjustments).
- **Renderable subset:** 870 of the 1,414. Excluded: 540 edits whose masks were drawn with Lightroom's AI selection tools (the mask rasters are computed on the desktop client and are not available to the cloud engine), and the 9 Leica D-Lux 8 DNGs, which the cloud engine does not process.
- **Neutral inputs:** each raw is also rendered once with no develop settings (camera DCP colour profile, lens distortion correction only) at 512 px, as the model's input image and as the reference for palette measurement.
- **Splits:** date-blocked (whole capture dates go to one side, so near-duplicate frames stay together). Imitation: 783 train / 53 valid over the 856 supported labels, holdout frames excluded (71 of the train rows are published tier). Curator: 9,870 pairs, 494 held out. Crop probes: 1,243 / 135 over all 1,414 labels; pairwise crop set: 2,638 / 327 pairs on the same split.

## Renderer

Two renderers were used.

1. **ART (open source, RawTherapee fork), locally.** Used first as the target format for the student and as the neutral renderer. Against Lightroom's own renditions on 15 geometry-matched frames, a neutral ART render sits at median mean-ΔE 33; the camera's Adobe Standard DCP profile improves that by 1.6 (most on the GR III). A gradient-free per-frame fit with ART's global tools closes it to median 14.1 (best 6.1) in six blind rounds, at minutes per frame; the remainder is local light, film-simulation colour and finish.
2. **Lightroom cloud, via the partner API.** A temporary asset is created with a develop payload declaring the XMP's SHA-256, the XMP is uploaded as external develop settings before the master, the raw is uploaded, and a 2048 px rendition is polled. Upload about 5 s for a 40 MB RAF, render 10 to 20 s; three in parallel scale linearly. Fidelity against the photographer's own Lightroom rendition of the same edit: mean ΔE 0.88 (p95 1.98, SSIM 1.00) on a GR III DNG with crop, orientation and two gradient masks; mean ΔE 1.27 (p95 3.0) on a Fujifilm frame, at JPEG-recompression scale. The output is the cropped content stretched into the full-frame aspect box, un-rotated; post-processing rotates by the catalog's user orientation and resizes to the crop aspect.

Facts about the cloud engine learned by diagnostic renders, since none are documented:

- `crs:HasCrop="False"` is ignored; the crop box values are the crop. To render uncropped, set the box to 0, 0, 1, 1.
- Hand-written parametric masks are applied, with three conditions: local values must be in Lightroom's normalised range (`LocalExposure2012` in [−1, 1], meaning ±4 stops; percentage dials in [−1, 1]), otherwise the whole correction is dropped silently; mask geometry is in full uncropped-frame coordinates, not crop-relative; a gradient is full beyond its Full point and zero beyond its Zero point, so the order of the two points is the direction.
- User orientation is not applied server-side. Crop boxes are stored in the sensor's frame, so for portrait-held frames the box must be rotated into the rendered frame before use; one label in six was affected before this was fixed.

The representation pivoted from ART recipes to Lightroom's own fields as soon as the cloud renderer was verified. The student now emits a compact JSON over the develop fields (about 480 tokens), which compiles to a valid XMP; the compiler clamps every local mask dial into the renderable range. The compact form represents 856 of the 1,414 labels fully, and the round trip XMP → JSON → XMP is exact on all 856; the rest carry AI masks, brush strokes, retouching or range-mask data, which it omits by design. Cloud round trip of a compiled target lands at the engine's own floor (ΔE 0.75 to 0.88) on the three frames bisected for it.

## Model and training

Base model Qwen3-VL-8B-Instruct, 4-bit MLX build, trained with mlx-vlm 0.6.17 on an Apple Silicon machine. LoRA rank 16, alpha 32, on the language side only (vision tower and merger frozen); Adam, learning rate 5e-5, batch 1; loss on completion tokens only. Input images are the 512 px neutral files (176 image tokens after patch merging) for every run except the 1024 px crop probe (672 tokens); the trainer's image-resize flag is a no-op for this model in that mlx-vlm version, which was discovered late and is why the "resolution" probe needed re-rendered inputs. Adapters are kept per run.

## Run 1: format SFT (retired)

Purpose: teach the ART recipe grammar. Result: the model learns an output grammar completely and quickly; every output parsed. Retired with the representation it served. The lesson carried over: a compiling recipe is no evidence of anything beyond the grammar.

## Run 2: curator (pairwise selection)

Task: given two unedited frames from the archive side by side, which one did the photographer later edit or publish rather than leave alone (tiers published > edited > ignored, pairs drawn within comparable dates). One epoch over 9,870 composites.

| metric (494 held-out pairs) | base, zero-shot | trained |
|---|---|---|
| pairwise accuracy | 0.51 | **0.79** |
| same answer when sides are swapped | 0.30 | **0.82** |

Comparative judgment is learnable from this archive with this recipe.

## Run 3: imitation in the Lightroom representation (lrtarget-sft-v0)

Task: neutral 512 px render in, compact develop JSON out. 3 epochs, 2,349 steps over 783 rows; train loss 0.67 → 0.28. Greedy decode on the 53-row valid split, base model as control.

| metric | base | v0 |
|---|---|---|
| JSON valid | 1.00 | 1.00 |
| compiles to a Lightroom-valid XMP | 0.00 | **1.00** |
| Look (film simulation) accuracy | 0.00 | 0.43 |
| crop IoU | 0.78 | 0.78 |
| crop within tolerance band (≥ 90 % of the label box kept, ≤ 2× area) | 0.81 | 0.79 |
| crop within band, heavy crops only (label keeps < 50 %, n = 10) | 0.00 | 0.00 |
| core dial MAE (±100 scale) | 22.6 | **15.5** |
| curve L1 | 9.6 | 13.4 |
| dial spread, predicted / label | — | 0.53 |
| distinct Looks, predicted / label | — | 1 / 9 |

The format is learned and the dial error halves, but the model collapsed to a modal recipe: it emits no Look for every frame (no-Look is 45 % of train labels, hence 0.43), never crops below 88 % of the frame where labels go down to 18 %, and its dial spread is half the photographer's. A constant predictor emitting the archive's median dials beats it.

## Crop probes

Six formulations of the crop alone, to separate "not learned" from "hidden by greedy decoding":

| probe | image signal |
|---|---|
| box regression, greedy | none (equals the full-frame prior) |
| box regression, sampled (T = 0.7) | none (area correlation −0.12) |
| box regression, likelihood of the label | +0.016 nats/token, P = 0.65 (shuffled-image control: +0.000, P = 0.53) |
| none / light / heavy classifier, 512 px, argmax | none (majority class) |
| classifier 512 px, ranking | AUC for heavy 0.62 (base zero-shot 0.64) |
| classifier 1024 px, ranking | AUC for heavy 0.58 (base zero-shot 0.60) |

Every formulation converges on the label prior. The untrained model already ranks heavily cropped frames slightly above chance from the image; fine-tuning erodes that rather than sharpening it. Read narrowly: the crop decision is not learnable from the unedited frame by maximum-likelihood imitation with a language-side LoRA. The main lane's ablation study gives the reason from the other side: the same frame is legitimately cropped several ways, the photographer's own tightness on one frame drifts with time, and 30 % of frames are uncropped for reasons not in the pixels.

Ruled out as routes to imitation: more epochs, other output formats, higher input resolution, per-field-group targets.

## Palette metric

Every edit's 2048 px rendition is compared to the neutral render of the same raw, cropped to the archive box and exposure-matched in linear light (the neutral base is deliberately dark; median gain +2.6 EV), in CIELAB: per source-hue bin (12), the median hue rotation and chroma ratio; the midtone a\*/b\* shift; all aggregated by Look and by scene brightness. Across the 1,264 colour edits (150 monochrome excluded): reds rotate +21° toward orange, magenta +16 to +33° toward red, cyan +12° toward blue; greens, blues and violets lose 10 to 25 % chroma; the midtone cast is +3.5 on b\* and deepens as the scene darkens (+5.1 dark, +2.4 bright). Chroma rises by ×1.24 overall (interquartile 0.98 to 1.54), ×1.32 in dark scenes and ×1.13 in bright ones, and only ×1.06 under the house film simulation (Classic Neg, 295 edits), which desaturates on its own. An earlier estimate from the main lane of a two-thirds chroma rise was made against un-matched neutral renders and is superseded. The metric needs no training and scores any rendered output.

## Next

- **Pairwise crop judge**, training now: the photographer's crop box against alternatives on the same frame (full frame, the same box translated, looser, tighter; for uncropped frames, the full frame against a random crop). The untrained model scores 0.52 on these pairs with a strong position bias.
- **Pairwise edit judge on rendered images.** Design rule: the wrong side of a pair must be a plausible alternative edit (another film simulation, another white balance, the main lane's counter-edit), not the edit switched off, or the judge learns to detect editing rather than the photographer.
- **Sampled decoding of v0** at temperature, scored by the palette metric, to tell "learned a distribution" from "learned nothing".
- **Reference tier of the main lane's own edits**, four so far, recorded as renderable XMPs alongside the photographer's on the same raws.
