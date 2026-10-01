# Results

All numbers are from the final report ([PDF](../report/MV_Project3_FinalReport.pdf)).

## Headline

| Pipeline | Acceptable (T < 25 mm, R < 15°) | Per-image success |
|---|---|---|
| V1, depth only | 17/24 (71%) | about 2% |
| **V2, PnP + depth fusion** | **21/24 (88%)** | **about 5%** |

V1 passes 17, kyjen regresses (−1), 5 objects are rescued (+5), giving 21. Per-image success improves 2.8×.

## All 24 objects

T in mm, R in degrees. "V2 %img" is the V2 per-image success rate for that object.

| Tier | Object | V1 T | V1 R | V2 T | V2 R | V2 %img | Change |
|---|---|---|---|---|---|---|---|
| Excellent | expo | 1 | 1 | 2 | 0 | 4.9% | |
| Excellent | genuine joe | 3 | 2 | 2 | 0 | 13.7% | |
| Excellent | crayola | 4 | 1 | 3 | 1 | 8.8% | |
| Excellent | highland | 4 | 2 | 4 | 2 | 6.9% | |
| Excellent | first years | 4 | 5 | 4 | 3 | 6.2% | |
| Excellent | cheezit | 6 | 2 | 9 | 1 | 16.9% | |
| Excellent | elmers | 6 | 2 | 7 | 3 | 2.1% | |
| Excellent | paper mate | 6 | 10 | 2 | 2 | 2.6% | |
| Excellent | laugh book | 7 | 3 | 1 | 2 | 2.1% | |
| Excellent | oreo | 7 | 11 | 3 | 2 | 6.0% | |
| Excellent | champion | 9 | 2 | 4 | 2 | 2.1% | |
| Acceptable | kong duck | 5 | 8 | 10 | 6 | 1.9% | |
| Acceptable | kong air | 14 | 6 | 5 | 8 | 1.9% | improved |
| Acceptable | stanley | 10 | 6 | 5 | 7 | 1.9% | improved |
| Acceptable | feline | 23 | 9 | 5 | 3 | 13.6% | improved |
| Acceptable | mark twain | 22 | 8 | 7 | 10 | 2.4% | improved |
| Acceptable | kong frog | 14 | 26 | 12 | 12 | 0.2% | V2 fixed |
| Acceptable | mommys | 17 | 21 | 5 | 4 | 15.0% | V2 fixed |
| Acceptable | dr browns | 29 | 65 | 11 | 4 | 3.3% | V2 fixed |
| Acceptable | sharpie | 55 | 30 | 2 | 8 | 2.3% | V2 fixed |
| Acceptable | safety | 80 | 8 | 2 | 6 | 0.7% | V2 fixed |
| Mismatch | kyjen eggs | 8 | 9 | 57 | 34 | 0% | regressed |
| Mismatch | rolodex | 78 | 30 | 10 | 16 | 0% | close miss |
| Mismatch | munchkin | 82 | 21 | n/a | n/a | n/a | processing error |

Tiers: 11 Excellent (T < 10, R < 5), 10 Acceptable (T < 25, R < 15), 3 Mismatch.

## The five traced objects

- **Expo eraser:** V1 and V2 both overlap the ground truth closely.
- **Cheezit:** both succeed. Highest per-image success rate (16.9%), because the large textured surface gives good features for both depth and RGB.
- **Sharpie:** V1 is displaced (T = 55 mm, R = 30°); V2 overlaps the ground truth (T = 2 mm, R = 8°). PnP translation fixed the seed-finding problem for this thin, colourful object.
- **Dr browns:** V1 fails (T = 29 mm, R = 65°) because the semi-transparent bottle brush gives unreliable depth seeds. V2 finds it through visible texture (T = 11 mm, R = 4°).
- **Kyjen eggs:** the opposite pattern. V1 works (T = 8 mm, R = 9°) but V2 drifts (T = 57 mm, R = 34°).

## Failure analysis

- **Kyjen eggs (V2 regression).** ORB detects 1812 keypoints on the RGB image, but all of them land on the shelf structure, not on the smooth eggs. Without valid correspondences on the object, PnP gives an incorrect translation anchor and the depth stages cannot recover. Fusion is not universally better.
- **Rolodex cup.** Misses by 1° (R = 16° vs the 15° threshold) due to cylindrical symmetry: the depth silhouette looks the same from many rotations around the vertical axis.
- **Munchkin duck.** Both pipelines crash during processing. The object gives inconsistent depth readings, likely because its reflective surface interacts poorly with the Kinect's infrared projector.

## Discussion

Depth and RGB are complementary. RGB through PnP gives translation (3 mm median); depth through render-and-compare gives rotation, because silhouettes encode 3D shape. Fusion combines both: PnP anchors translation, render-and-compare searches rotation, ICP refines.

Related work: Rennie et al. [1] evaluated LINEMOD [3] on this dataset and found translation-dominated errors from confusing clutter objects with the target. This approach uses 3D geometry directly, which avoids that but introduces a different bottleneck (seed quality) that V2 fixes. SIFT [7] might give better correspondences than ORB for rotation. The RGBTrack paper [9] also uses render-and-compare for rotation. The BOP benchmark [8] uses symmetry-aware metrics that would count rolodex (1° over) as correct.

## Limitations

- The depth prior (±50 mm) assumes shelf calibration is available, and the five clustering approaches tested could not replace it.
- Processing takes ~25 minutes per object due to the exhaustive search across 432 images × 4 stages.
- Per-image success of 2 to 5% means 432 images are necessary, not excessive.
- Reflective objects (munchkin) cause processing failures because the Kinect depth sensor gives unreliable readings on them.

Bracketed numbers refer to [references.md](references.md).
