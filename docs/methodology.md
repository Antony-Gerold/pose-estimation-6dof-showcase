# Methodology

Everything here follows the final report ([PDF](../report/MV_Project3_FinalReport.pdf)).

## 1. Dataset: Rutgers APC RGB-D

The Rutgers APC dataset [1] was created to support research on the Amazon Picking Challenge [2].

| Property | Value |
|---|---|
| Objects | 24 household objects |
| Images per object | 432 = 12 shelf bins × 36 viewpoints |
| Per image | Kinect v1 depth (uint16 mm), RGB, binary bin mask, ground-truth 6DOF pose |
| 3D models | STL (geometry) and OBJ with PNG textures |
| Camera | fx = 572.41, fy = 573.57, cx = 325.26, cy = 242.05 |

Five objects are traced through the report: expo eraser (both pipelines pass), cheezit (both pass, texture-rich), sharpie (V1 fails, V2 fixes), dr browns (V1 fails, V2 fixes), and kyjen eggs (V1 passes, V2 regresses).

## 2. Evaluation

Three tiers:

- **Excellent:** T < 10 mm, R < 5°
- **Acceptable:** T < 25 mm, R < 15° (the main threshold)
- **Mismatch**

Both pipelines use a depth prior of ±50 mm around the shelf depth for initial localisation, mimicking real shelf calibration [2]. No ground-truth rotation or X/Y translation is used.

## 3. Environment analysis

### 3.1 The bin mask problem

The masks outline the shelf compartment, not the object. Within one mask, three depth layers compete: shelf edge (~520 mm), object (~600 mm), and back wall (~930 mm). Five strategies to isolate the object without the depth prior were tried (P25 percentile, histogram peak, floor subtraction, largest connected component, centre blob). None matched simple depth filtering (±50 mm).

### 3.2 PnP exploration

ORB features [6] from the RGB image, 3D coordinates from the depth image at those locations, then solvePnPRansac [5]. Result: median T = 3 mm (excellent) but R = 166° (random). ORB on depth gave 1398 keypoints and 7 matches (fails); ORB on RGB gave 1744 keypoints and 16 matches (works). With only 16 sparse correspondences clustered on one face, EPnP cannot constrain the rotation.

This gave two conclusions: depth is needed for rotation, and PnP can provide translation if depth-based translation fails. V1 (depth only) was built first, then PnP was used to fix V1's translation failures in V2.

## 4. V1: depth-geometric pipeline

Four stages, each fixing a failure of the previous one.

1. **Seed finding.** Filter mask pixels by the depth prior (±50 mm), select 40 seeds, back-project to 3D with p3D = [(u − cx)Z/fx, (v − cy)Z/fy, Z]. When seeds land on shelf structure at a similar depth, everything downstream fails. This is V1's bottleneck.
2. **ICP alignment.** For each seed, extract a local scene cloud (60 px radius), try 16 initial rotations, and run 25 iterations of ICP [4] with the model downsampled to ~400 points. Select the best coverage pair.
3. **Render-and-compare.** Keep the ICP translation and search ~200 rotations by rendering model depth silhouettes:

$$S_{render} = \frac{|\{p : |D_{render}(p) - D_{obs}(p)| < 5\,\text{mm}\}|}{|\{p : D_{render}(p) > 0 \wedge D_{obs}(p) > 0\}|}$$

   This fixes rotation for asymmetric objects but creates a 180° ambiguity for symmetric silhouettes.
4. **Flip correction.** Test 180° rotations around X, Y, Z, re-score, keep the best.

**Multi-view selection.** All 432 images are evaluated independently and the best is selected. Only about 2% of images reach acceptable accuracy, but 432 per object gives enough candidates.

**V1 result: 17/24 (71%).** The 7 failures all come from seeds missing the object, for example sharpie (T = 55 mm), safety (T = 80 mm), and dr browns (T = 29 mm).

## 5. V2: PnP + depth fusion

1. **PnP translation:** ORB + depth, then solvePnPRansac, gives translation.
2. **Render-compare rotation:** the same 200-candidate search, anchored at the PnP translation.
3. **ICP refinement:** polish both T and R.
4. **Smart flip:** only flip when confidence < 0.3 and improvement > 5%.

Smart flip was critical. Without it, on cheezit config G, ICP gives R = 1° but a naive flip selects R = 179° because the flipped silhouette scores slightly higher from noise.

**V2 result: 21/24 (88%).** Per-image success went from about 2% to about 5% (2.8×).

## 6. Texture domain gap (dead end)

Model textures (brand text, barcodes) were explored as a way to resolve the 180° flip ambiguity. Scene RGB is dark Kinect lighting and model textures are studio lighting. Illumination correction raised the match from 9% to 84%, but all rotation candidates improved equally, so correct and incorrect orientations could not be distinguished. This confirmed depth render-and-compare as the right rotation approach.

Bracketed numbers refer to [references.md](references.md).
