# Krimble — Roadmap Notes

Status: **planning only** — not started. Captured here so they don't stay stuck in chat.

## Advanced JPEG Export Module

Goal: selective/regional compression instead of one global quality slider — get
files that look uncompressed at meaningfully smaller sizes, the way tools like
Guetzli, MozJPEG (trellis quantization), and XAT-style optimizers do.

**Core mechanism:** build a perceptual quality map across the image instead of
applying one quality level everywhere:

- **Edge/detail-heavy regions** (text, logos, sharp transitions) -> higher
  quality/lower compression -- artifacts are most visible here.
- **Flat/gradient regions** (backgrounds, sky, smooth fills) -> deceptive:
  hide detail loss well, but reveal banding easily, so these need careful
  handling rather than just "compress harder because there's nothing there."
- **Texture-heavy/busy regions** -> most tolerant of aggressive compression.

Quality map can be built from Sobel/Laplacian edge detection or local
variance per block; ideally validated with a perceptual metric (SSIM or
butteraugli) comparing candidate compression levels against the source and
picking the lowest quality that stays under a visible-difference threshold,
per region.

**Grayscale/B&W-specific behavior:** B&W images should get a more aggressive
default compression curve than color images. JPEG normally spends a large
share of its bit budget on chroma subsampling, which a grayscale image
doesn't need at all -- every bit goes to luminance/edge fidelity instead.
Combined with the regional map above, flat B&W regions (line art, text,
high-contrast logo/mascot work) can take much heavier quantization than
their color equivalents before anything perceptible changes.

**Implementation notes:** likely built on libjpeg-turbo, either via per-region
quantization table manipulation if exposed directly, or a tile-based
export-and-reassemble approach if not.

## New Tools

- **Smudge tool** -- ✅ shipped (commit `7390d6b`). Dedicated toolbox tool
  wrapping Krita's existing colorsmudge paintop engine, own icon/shortcut/
  toolbox slot, auto-loads the "smudge" preset on activation.
- **Soften tool** -- not started. Photoshop's Blur tool equivalent: a
  localized blur/softening brush for softening edges/skin/detail in specific
  areas without running a full-image filter. Krita's `filterop` paintop
  engine (paint-with-any-filter) already exists and can drive this --
  the work is presetting it to Gaussian Blur with sane defaults and giving
  it its own toolbox entry, same pattern as Smudge.

## AI Upscaling Tool

Status: **not started**. Chosen approach below, after evaluating licensing.

**Model source: OpenCV's `dnn_superres` module** (opencv_contrib, Apache 2.0
license as of OpenCV 4.5+). Official project (built via Google Summer of
Code), not a personal/research repo -- both the module code and its
pretrained models are covered under OpenCV's own license terms. This was
chosen specifically because most independent FSRCNN/ESPCN reimplementations
found elsewhere have no stated license at all, making them unusable as-is.

Four models are available, covering a real quality/speed/size spectrum:

| Model | Size | Speed | Quality |
|---|---|---|---|
| ESPCN | ~100 KB | Fastest | Lowest of the four, still well above bicubic |
| FSRCNN | Small (light-weight, "-s" variant even smaller) | Fast | Decent, good speed/quality balance |
| LapSRN | Medium | Medium | Multi-scale (progressive 2x/4x/8x via Laplacian pyramid) |
| EDSR | ~38.5 MB quantized (150 MB original) | Slowest | Highest quality of the four |

**Target approach (matches the "Samsung Gallery" bar discussed):** don't
depend on the full OpenCV library on-device. Instead, take the ESPCN or
FSRCNN pretrained `.pb` model out of `dnn_superres`, convert it to
`.tflite`, and run it through TensorFlow Lite's **NNAPI delegate** --
this is what lets inference actually use the phone's NPU/DSP/GPU instead of
running CPU-bound, which is the real reason Samsung Gallery's on-device
upscaling feels fast. EDSR/LapSRN are logged here for future
higher-quality/slower options (e.g. an explicit "high quality, takes
longer" mode) but ESPCN/FSRCNN are the realistic default for a responsive
in-app tool.

**Still needed before this is buildable:**
- Add TensorFlow Lite to the deps toolchain (nothing ML-related is currently
  linked at all -- confirmed via full search of CMakeLists and
  krita-deps-management).
- Convert the chosen `.pb` model to `.tflite` and verify NNAPI delegate
  support for its specific ops.
- Tiling pipeline for images larger than the model's expected input size.
- Tool/dialog UI.

**Separately evaluated, not chosen as the primary model:** Real-ESRGAN
(xinntao/Real-ESRGAN, BSD-3-Clause, confirmed clean license) -- GAN-based,
better detail hallucination than the dnn_superres models, but heavier/slower
and a bigger integration lift. Worth revisiting later as a higher-quality
optional mode once the ESPCN/FSRCNN path is working.

## Quick Mask

Status: **not started** -- confirmed genuinely missing (searched the entire
codebase under every naming variant: quickmask, quick_mask, QuickMask --
zero hits). Not a relocation/UI-parity case like Channels or Select Opaque;
this needs building from scratch.

**What it is:** paint a selection into existence instead of using
marquee/lasso/wand. Toggle on (Q key in Photoshop) -> canvas shows a
semi-transparent red overlay (rubylith-style) over everything NOT selected
-> paint in black/white/gray with any normal brush (black removes from
selection, white adds, gray = partial/feathered) -> toggle off -> the
painted result becomes a real active selection.

**Closest existing Krita infrastructure to build on:**
- Krita's Selection Masks (grayscale mask layers) are conceptually the
  same underlying data model a Quick Mask needs -- a paintable grayscale
  layer that converts to/from a real selection.
- The Colorize Mask tool (libs/image/lazybrush/kis_colorize_mask.cpp)
  proves the "special temporary paintable overlay tied to a toggle state"
  pattern already exists in this codebase, just for a different purpose.

**Rough implementation shape:**
1. Toggle action that creates a temporary paintable grayscale device
   representing the mask, seeded from the current selection (or fully
   selected if none exists).
2. Canvas rendering: composite that grayscale device as a semi-transparent
   red overlay over the normal image while active (inverse of the
   grayscale value = overlay opacity, matching the rubylith convention).
3. While active, redirect normal paint tools to paint into the mask device
   instead of the image (grayscale in, not full color).
4. On toggle-off: convert the grayscale mask device into a real
   KisSelection (threshold/gradient -> selection, feathered where gray),
   apply it as the active selection, discard the temporary device.
5. Toolbox button (bottom of toolbox, alongside the FG/BG swap widget and
   Screen Mode) + Q shortcut, matching Photoshop's actual entry points.

Likely a genuinely medium-sized feature -- more than a UI wrapper, less
than the AI upscaler. Worth scoping properly (time estimate, step-by-step
plan) when it's next in line for actual implementation.
