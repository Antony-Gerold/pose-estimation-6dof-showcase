# Methodology

## 1. Dataset: Rutgers APC RGB-D

The Rutgers APC dataset [Rennie et al., 2016] was built to support work on the Amazon Picking Challenge [Correll et al., 2016]. It contains:

| Property | Value |
|---|---|
| Objects | 24 household items (cheezit, sharpie, etc.) |
| Per-object images | 432 = 12 shelf bins × 36 viewpoints |
| Modalities | Kinect v1 depth (uint16 mm) + RGB + binary bin mask + GT pose |
| 3D models | STL geometry + OBJ with PNG textures |
| Camera intrinsics (IR) | fx = 572.41, fy = 573.57, cx = 325.26, cy = 242.05 |

**Five tracer objects** are used throughout the report:
- `expo_eraser` — both pipelines pass
- `cheezit` — both pipelines pass, texture-rich (best PnP case)
- `sharpie` — V1 fails, V2 fixes
- `dr_browns` — V1 fails, V2 fixes (hardest object)
- `kyjen_eggs_holder` — V1 passes, V2 regresses (textureless)

## 2. Environment analysis (before pipeline design)

### 2.1 The bin-mask problem

The dataset's binary masks outline the shelf *compartment*, not the object. Inside one mask, depth has three peaks: shelf edge at ~520 mm, object at ~600 mm, and back wall at ~930 mm. Naive seed-finding from the mask depth picks the *shelf edge* and never reaches the object. Mitigation requires either depth-prior gating or explicit segmentation of the object inside the mask.

### 2.2 PnP exploration

Before building V1, I tested the pure-PnP approach from Lecture 8:
1. Extract ORB keypoints from RGB scene
2. Look up 3D coordinates from depth
3. Solve with `cv2.solvePnPRansac`

**Results across all 432 images of cheezit:** median translation error **3 mm** (excellent), median rotation error **166°** (essentially random).

**Why:** ORB correspondences are dense in *position* (16 matches in a small region constrain $(x, y, z)$ well) but sparse in *angular spread* (those same matches don't span enough of the object to constrain $(\phi, \theta, \psi)$). This is the well-known EPnP degeneracy for narrow-cone correspondences [Lepetit et al., 2009].

This finding drove the V2 fusion design — PnP for **translation**, depth methods for **rotation**.

### 2.3 ORB on depth vs RGB

Direct comparison on one object:
- ORB on depth: 1,398 keypoints, **7** texture matches against model → PnP fails
- ORB on scene RGB: 1,744 keypoints, **16** matches → PnP succeeds (T = 3 mm)

Depth has only edge gradients; texture has discriminative high-frequency patterns. Texture-domain gap (model vs scene illumination) still costs accuracy but is recoverable.

## 3. V1 — Depth-geometric pipeline (17/24 acceptable)

### 3.1 Stage 1 — Seed finding

Mask depth histogram → cluster peaks → pick candidates in the object depth range (relative to the calibrated shelf depth ± 50 mm). This is the V1 bottleneck — when the object is small relative to the bin mask, depth seeds land on shelf surfaces.

### 3.2 Stage 2 — ICP alignment

For each seed:
- Extract local scene cloud (60 px radius around the seed projection)
- Try **16 rotation candidates** (3 elevations × 4 azimuths + 4 axis flips)
- Run **25 iterations of point-to-point ICP** [Besl & McKay, 1992]
- Model downsampled to ~400 points via voxel grid
- Select best (T, R) by inlier coverage

### 3.3 Stage 3 — Render-and-compare

Keep ICP's T. Search **~200 rotation candidates** by rendering the model's depth silhouette with trimesh:

$$
S_{render} = \frac{\#\{p : |D_{render}(p) - D_{obs}(p)| < 5\,\text{mm}\}}{\#\{p : D_{render}(p) > 0 \wedge D_{obs}(p) > 0\}}
$$

Best score wins. Fixes rotation for asymmetric objects. Fails for objects whose depth silhouette is nearly symmetric front-to-back, creating a **180° ambiguity**.

### 3.4 Stage 4 — Flip correction

Test 180° rotations around X, Y, Z. Re-score each with render-compare, keep the best. Stages rescue each other across different images:
- `expo` config G: seed+ICP gives R = 6°, render breaks to R = 165°, flip saves to R = 3°.
- `cheezit` config A: seed+ICP gives R = 180°, flip recovers to R = 6°.

### 3.5 Stage 5 — Multi-view voting

Evaluate all 432 images of an object independently. Score each by:

$$
S_{comb} = \frac{T_{err}}{25\,\text{mm}} + \frac{R_{err}}{15°}
$$

Pick the single best image. **Why this is necessary:** per-image robot rate is only **1.8 %** (181 / 9804 across V1's runs) — but 1.8 % × 432 = ~8 acceptable candidates per object, enough to find one.

### 3.6 V1 result

**17 / 24 acceptable (71 %).** The 7 failures all share a pattern: depth seeds miss the object. The 5 worst:
- `safety_glasses` T = 80 mm
- `sharpie` T = 55 mm
- `dr_browns` T = 29 mm
- `mommys_helper` T = 17 mm, R = 21°
- `kong_frog` T = 14 mm, R = 26°

All are small objects relative to the bin mask.

## 4. V2 — PnP + depth fusion (21/24 acceptable)

### 4.1 Architecture

V2 replaces Stage 1 (depth seeds) with PnP translation:

| # | Stage | What it produces |
|---|---|---|
| 1 | PnP translation: ORB(RGB) + depth → `solvePnPRansac` | t̂ |
| 2 | Render-compare at t̂: 200 candidates → best silhouette | R̂ |
| 3 | ICP refinement | refined (R, t) |
| 4 | **Smart flip** (confidence-gated) | final (R, t) |
| 5 | Multi-view vote over 432 images | selected best |

### 4.2 The smart-flip discovery

Initial V2 with naive flip correction *systematically broke* correct poses. Example: `cheezit` config G — ICP achieves R = 1°, but the 180°-flipped silhouette scores marginally higher than the correct silhouette due to depth noise. Naive flip selects R = 179°.

**The gate, in two parts:**
1. **Skip the flip entirely** when render confidence ≥ 0.30 (the ICP pose is already trusted).
2. If running, **only accept the flip** when its score exceeds the original by more than 5 %.

This single change separated working V2 from a regression.

### 4.3 V2 result

**21 / 24 acceptable (88 %)**, with per-image robot rate **5.0 %** (vs 1.8 % for V1, a **2.8× improvement**).

- **5 V1 failures rescued:** `sharpie` (55 → 2 mm), `safety_glasses` (80 → 2 mm), `dr_browns` (29 → 11 mm), `mommys_helper` (17/21° → 5/4°), `kong_frog` (14/26° → 12/12°).
- **1 V1 success regressed:** `kyjen_eggs_holder` (8/9° → 57/34°). Cause: matte plastic with no surface texture; ORB returns no usable keypoints; PnP returns garbage; the depth-fallback path is no longer wired in.
- **2 hard failures unchanged:** `rolodex` (cylindrical symmetry causes pose ambiguity) and `munchkin` (data error — kernel crash).

### 4.4 Ablation findings

Per-image ablation across 6 objects × 10 images each (cell 45 of `04_FINAL_v1_and_ablation.ipynb`) shows that **no single pipeline stage is universally helpful**. Each of seed+ICP, render-compare, ICP-refine, and flip helps ~50 % of images and hurts the other ~50 %. They rescue *different* failure modes. Multi-view voting across 432 images selects the frames where the right stages happened to align — which quantifies why this pipeline design is structurally tied to having lots of viewpoints.

## 5. Why this was graded 9.5 / 10

ENGG\*6100 (Prof. Medhat Moussa) rewards:
- Chronological *tried → failed → learned → pivoted* narrative — V1 → diagnose → V2
- Concrete numbers at every step — 17/24, 21/24, 1.8 %, 5.0 %, T = 3 mm, R = 166°
- Comparison tables — full 24-row V1-vs-V2 comparison
- Physical reasoning — ORB clustering explains PnP rotation failure
- External references — Besl & McKay, Lepetit et al., Rublee et al., Correll et al., Rennie et al.

Lost 0.5 marks (compared with Project 2's 9.7) on discussion depth around the texture-domain-gap analysis and limitations — flagged for the next project.
