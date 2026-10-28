# Physics-Based Chinese Brush Simulator

A real-time, physically based Chinese brush (毛笔) simulator that runs as a single
HTML file in the browser, plus an evaluation of the engine against two independent
test batteries — human-trajectory brush-tip criteria and the quantitative protocol
of Madono & Simo-Serra (Computer Graphics Forum 2023).

**Live demo:** open [`index.html`](index.html) in a browser. Works on desktop and on
iPad (Apple Pencil pressure, tilt and coalesced events are supported).

---

## What this is

Most digital brush engines store a stroke as a texture or a width profile. This one
derives stroke width from the *interaction* of the tip with the paper: the tip is
modelled as a bundle of bristles with axial bending and torsion, contacting paper
with Coulomb friction. Width follows from the contact footprint, so the parameters
have physical meaning — bristle stiffness, splay, ink capacity, paper absorbency —
rather than being arbitrary knobs.

Three input paths are handled:

| Input | Pressure source | Stroke assembly |
|---|---|---|
| Apple Pencil | per-sample pressure → penetration depth | stamped per input sample |
| Mouse / trackpad | simulated from writing speed | queued and flushed with a taper envelope |
| No pressure at all | speed-synthesised | same queue |

## Engine features

- **Pressure response** is convex: `f''(p) > 0`, so the brush widens progressively
  faster as the user presses harder. Measured widths at constant pressure
  (0.10 → 0.90) are `2, 13, 30, 57, 88` px — first differences `+11, +17, +27, +31`,
  second differences `+6, +10, +4`.
- **Turn handling**: sustained direction change (12–15° per sample at 240 Hz) is
  detected from a direction-history buffer and progressively scales down the drag
  deformation, which otherwise layers adjacent stamps out of register and produces a
  fan-shaped pale fringe at stroke turns.
- **Ink depletion**: ink drains along the writing order, so a character written without
  re-dipping fades from wet to dry the way a real brush does.
- **Brush / paper presets**: goat-hair on raw (absorbent) paper, wolf-hair on sized
  paper, and mixed-brush low-ink, each with distinct edge quality and bleed.

## Evaluation

### Test A — human-trajectory brush-tip criteria

Real digitizer strokes (pressure + tilt + timestamps) are replayed through the
engine's native pressure channel. Against gates derived from 339 real strokes:

| Stroke | Rendered width ↔ human pressure, r |
|---|---|
| A — slow entry, concealed tip | 0.73 |
| B — fast exit, exposed tip | 0.87 |
| C — long composite | 0.92 |

All clear the decisive gate (r > 0.70). Geometry gates also pass: water-drop ratio
1.31 / 1.36 / 1.38 (≥ 1.25), leap-to-landing width ratio 0.60 (≥ 0.45), hook taper
0.87 / 0.13, stroke breaks 0, taper failures 0/5.

### Test B — Madono & Simo-Serra (CGF 2023) protocol

The paper's protocol compares rendered strokes against photographed references by
LPIPS, SSIM and MSE. Its dataset is gated, so two things were done instead:

1. **Substitute dataset.** The Princeton RealBrush dataset (SIGGRAPH 2013) is
   structurally isomorphic — vector path plus a photographed alpha mask of the same
   medium. On 17 strokes: MSE `0.132`, SSIM `0.783`, LPIPS `0.340`, which places the
   engine in the same band as the commercial tool Expresii (LPIPS 0.3413) but behind
   the data-driven methods (0.2593). A metric absent from the original protocol —
   correlation between rendered radius and reference half-width — reaches `r = 0.97`.

2. **Same-input qualitative comparison.** The input stroke was recovered from column
   (a) of the paper's Figure 1 by skeletonisation (with per-point half-width from a
   distance transform) and rendered by this engine as column (g). See
   [`figures/fig_madono_compare.png`](figures/fig_madono_compare.png).

### Side-by-side against the paper's methods

| Object | Fresco | Rebelle 6 | DeepBrush | Ours (paper) | **This engine** |
|---|---|---|---|---|---|
| cursive character | 0.123 | 0.119 | 0.109 | 0.111 | 0.085 |
| smiley line drawing | 0.173 | 0.278 | 0.122 | 0.155 | **0.180** |

IoU against the paper's Reference column, at matched ink coverage (Reference 0.140).
On the line drawing the engine exceeds Fresco and the paper's own model; on the
cursive character it remains about 30 % behind. Columns (a)–(f) of the figure are
reproduced from the paper for comparison; only column (g) is this engine's output.

## Repository layout

```
index.html                      the simulator (single file, no build step)
paper/brush_eval.pdf            evaluation write-up, IEEE conference format
paper/brush_eval.tex            LaTeX source
paper/madono2023_reference.pdf  the CGF 2023 paper this work is evaluated against
figures/                        qualitative comparison figures
```

## Reproducing the evaluation

Requires Python 3.10+ with `numpy`, `scipy`, `Pillow`, and `pymupdf` for the figure
extraction, plus `pdflatex` (TeX Live or MiKTeX) to build the paper.

```bash
pip install numpy scipy pillow pymupdf
cd paper && pdflatex brush_eval.tex && pdflatex brush_eval.tex
```

## Known limitations

- Stroke skeletons recovered from raster images fragment at complex intersections.
  The engine therefore shows small gaps where a character's loops cross. The proper
  fix is original vector stroke data rather than skeletons, which the authors' gated
  dataset provides and which is pending access.
- The engine is a general physical model, not one trained on a specific medium. It
  should not be expected to match a convolutional network's texture fidelity when
  that network was trained on photographs of that same medium.
- Absolute metric values are not directly comparable with the published table: theirs
  are computed on full recursive paintings (148 strokes, mutually occluding), these
  are single strokes on a different dataset.

## Acknowledgements

Evaluation follows the protocol of:

> Koki Madono and Edgar Simo-Serra. *Data-Driven Ink Painting Brushstroke Rendering.*
> Computer Graphics Forum 42(7), e14965, 2023.

Substitute dataset:

> Jingwan Lu, Connelly Barnes, Stephen DiVerdi, Adam Finkelstein. *RealBrush: Painting
> with Examples of Physical Media.* ACM Transactions on Graphics (SIGGRAPH) 32(4), 2013.