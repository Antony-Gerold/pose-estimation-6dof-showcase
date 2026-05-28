# Results

## Headline

| Pipeline | Acceptable | Per-image robot rate |
|---|---|---|
| V1 (depth only) | 17 / 24 (71 %) | 1.8 % |
| **V2 (PnP + depth)** | **21 / 24 (88 %)** | **5.0 %** |

V2 vs V1: **+5 rescued, −1 regression, net +4.** Per-image robot rate improves **2.8×**.

Acceptable threshold: **T < 25 mm AND R < 15°** ("robot-grade" — the threshold below which a robot arm can grasp the object reliably).

## Full per-object comparison

T = translation error (mm), R = rotation error (degrees). Bold rows are status changes between V1 and V2.

| # | Object | V1 T | V1 R | V2 T | V2 R | Status |
|---|---|---|---|---|---|---|
| 1 | expo_eraser | 1 | 1 | 2 | 0 | Both pass |
| 2 | genuine_joe | 3 | 2 | 2 | 0 | Both pass |
| 3 | crayola | 4 | 1 | 3 | 1 | Both pass |
| 4 | highland | 4 | 2 | 4 | 2 | Both pass |
| 5 | first_years | 4 | 5 | 4 | 3 | Both pass |
| 6 | cheezit | 6 | 2 | 9 | 1 | Both pass |
| 7 | elmers | 6 | 2 | 7 | 3 | Both pass |
| 8 | champion | 8 | 2 | 4 | 2 | Both pass |
| 9 | laugh_out_loud | 7 | 3 | 1 | 2 | Both pass |
| 10 | paper_mate | 6 | 10 | 2 | 2 | Both pass |
| 11 | oreo_mega | 7 | 11 | 3 | 2 | Both pass |
| 12 | kong_duck | 5 | 8 | 10 | 6 | Both pass |
| 13 | kong_air | 14 | 6 | 5 | 8 | Both pass |
| 14 | mark_twain | 22 | 8 | 7 | 10 | Both pass |
| 15 | stanley | 10 | 6 | 5 | 7 | Both pass |
| 16 | feline_greenies | 23 | 9 | 5 | 3 | Both pass |
| **17** | **mommys_helper** | **17** | **21** | **5** | **4** | **V2 RESCUE** |
| **18** | **kong_frog** | **14** | **26** | **12** | **12** | **V2 RESCUE** |
| **19** | **sharpie** | **55** | **30** | **2** | **8** | **V2 RESCUE** |
| **20** | **safety_glasses** | **80** | **8** | **2** | **6** | **V2 RESCUE** |
| **21** | **dr_browns** | **29** | **65** | **11** | **4** | **V2 RESCUE** |
| **22** | **kyjen_eggs** | **8** | **9** | **57** | **34** | **V2 REGRESSION (textureless)** |
| 23 | rolodex | 78 | 30 | 10 | 16 | Both fail (cylindrical symmetry) |
| 24 | munchkin | crash | — | — | — | Data error |

## Pipeline ablation (cell 45 of `04_FINAL_v1_and_ablation.ipynb`)

3 objects × 10 images each, recording T and R after each cumulative stage:

| Stage | What it adds | Helps ~ |
|---|---|---|
| 1 | Seed only | — |
| 2 | + 16-rotation ICP | ~ 50 % images |
| 3 | + Render-compare (replace R) | ~ 50 % images (different set) |
| 4 | + ICP refinement | ~ 50 % images |
| 5 | + Flip correction | ~ 50 % images |

**Each stage helps ~50 % and hurts ~50 %.** They rescue different images, not the same images progressively. This is the central finding of the ablation: the pipeline is not a monotonic chain that reduces error at every step — it's a *bag* of correction strategies, and multi-view voting picks the frames where the right combination aligned.

### Two specific examples from the ablation table

| Object | Image | Seed+ICP | Render | ICP-refine | Flip | Notes |
|---|---|---|---|---|---|---|
| expo | G-1-1-0 | T = 8, R = 180° | T = 180, R = 178° | T = 15, R = 179° | **T = 15, R = 3°** | Flip is the only stage that saves it |
| cheezit | G-1-1-0 | T = 16, R = 179° | T = 16, R = 13° | **T = 10, R = 1°** | T = 10, R = 179° | Render-compare fixes it; flip *breaks* it |

The cheezit example is exactly what motivated the smart-flip gate in V2.

## Failure analysis

### V1 failures (7 objects)

All failures share a pattern: **depth seed finding misplaces the initial estimate**, and no downstream stage can recover from a seed that lands on the shelf wall or back. The 5 worst are objects that are small relative to their bin mask:
- safety_glasses (small)
- sharpie (long and thin)
- dr_browns (irregular)
- mommys_helper (small)
- kong_frog (small)

The remaining 2 failures (rolodex, munchkin) are structural — symmetry and data corruption respectively, not seed problems.

### V2 regression: kyjen_eggs_holder

`kyjen` is matte plastic with effectively no surface texture. ORB extracts ~0 usable keypoints from the scene; `solvePnPRansac` returns garbage. V1 used depth-seed translation as a fallback when PnP-ish stages failed, but V2's design has PnP as the *primary* translation source. Result: 8/9° → 57/34°. The fix is conceptually simple — fall back to V1's depth-seed approach when PnP confidence is below a threshold — but was not implemented for the report deadline.

### Unfixed failures

- **rolodex** — cylindrical pencil cup. Cylindrical symmetry means any rotation about the cylinder axis produces the same depth and texture. Pose around that axis is fundamentally ambiguous from a single view.
- **munchkin** — kernel crash on data load. Data issue, not a pipeline issue.

## Per-image rate is the right metric for understanding

The "21 / 24 acceptable" number can hide what the pipeline actually does. The honest metric is **per-image robot rate**: V1 = 1.8 %, V2 = 5.0 %. Both are tiny — most single-image predictions are wrong. The multi-view selection step is what produces the headline numbers, by harvesting the few accurate frames out of 432.

This is the project's strongest external lesson: **pose estimation from a single depth image is unreliable; data quantity is structurally necessary**. With 5 images per object, both pipelines would collapse.
