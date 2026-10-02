# Path Selection Fix (MODE_AUTO) - Notes
Date: 2025-10-02
Author: opencode

## Problem statement recap
TODO.md defects 1-3; user tackling #1 first. Must make executive decisions and durable notes.

## Current logic
In main AUTO (around line ~810-820):
```c
if (pathMode == MODE_AUTO) {
    if (refMax <= 0) path = PATH_FP64_DIRECT;
    else path = moving ? PATH_PERT32 : PATH_PERT64;
}
```
Ignores refShortFor (fallback partial reference), ignores that short reference is low quality. Also at spans > 1e-12 direct fp64 is geometrically fine; choosing marginal pert loses quality.

## Design choice (executive)
Prefer quality/capability:
- No ref -> direct64
- If refShortFor > 0 (marginal: we only got fraction of requested budget), the reference is not deep enough. Direct64 is better at moderate spans (> ~5e-12). At very deep spans (<= 5e-12), direct64's coordinate floor dominates; marginal pert may still be worse but maybe only option? Or prefer direct if it can do anything. But below 1e-13 direct is wall. Threshold: if marginal ref and span > 1e-11, prefer direct64 (quality); else prefer pert even if marginal (deep). 
- If refShortFor == 0 (good full-length reference), prefer pert (fp32 moving, fp64 settled) - it's correct and often faster.
- Keep moving->pert32, settled->pert64 for good refs.

Also consider iter/refMax: if refMax < iter*0.7, treat as marginal? refShortFor already means settled short.

Conservative threshold 1e-11:
- span > 1e-11 and marginal ref -> direct64
- else if ref exists -> pert (32 moving/64 settled)
- else direct64

## Change made
Modified AUTO path selection in pert.c to check if reference is marginal (refShortFor>0). If marginal and span > 1e-11, prefer direct64; else use perturbation (pert32 moving, pert64 settled). This avoids throwing away direct fp64 quality when we only found a short fallback reference.

## Rationale
Matches TODO observation: short (500-iter) reference is poor quality at ~1e-10 compared to direct fp64. Threshold 1e-11 is conservative (above direct wall ~1e-13).

## Next (RefFind - best not first)
TODO says RefFind takes first survivor (first point found in expanding rings). Better to prefer longest-surviving or most distant? Or prefer reference with largest refFilled for given need, or search and keep max refFilled among survivors. Also prefer references that survive longer (better for depth) - so when searching, don't return first; collect candidates or track best (max n survived).

Design: when searching up to radius, evaluate candidates and pick the one with highest survival count for the target need, breaking ties by proximity? Or just maximize refFilled. Also avoid settling for marginal when better exists nearby.

Implementation approach: change RefFind to, instead of returning immediately on first survivor of 'need', track best = {px,py,survived_n} where survived_n is max iterations survived (could be >= need). For each candidate test up to need (or up to larger cap?) - but testing is expensive. Alternatively, when looking for need, also record if it survives much more? Or do two-pass: first pass find any that survives need (current behavior), but to get best, test each candidate to see how far it goes up to need? Or just prefer larger refFilled.

But RefSurvives only returns boolean for given n. To know "how long it survives", we need to extend logic. Maybe add RefSurvivesMax(cx,cy,maxn) returning count. That doubles work in worst bad case? Or during search, for candidates that survive 'need/f' in fallback, track best. Alternatively, modify search: track best_survived = -1, best_cx,best_cy. For each candidate, find actual survived count up to need (loop until bail or reach need) - if survived == need, that's perfect (full); else it's partial survived count. Prefer higher survived count; if tie, prefer closer (smaller r,dx,dy)? Closer better (smaller w perturbation). So best is max survived desc, then min distance asc.

This is more work but better. Let us implement efficiently.

## Change made (RefFind)
Replaced first-survivor with best-by-survival: track best (max survived count desc, then min distance asc). For each candidate in expanding rings, compute RefSurvivesCount up to need. If any survives >= need, adopt immediately. Otherwise adopt the best found (longest surviving) even if < need - this improves depth ceiling and reduces marginal refs.

## Impact
Should reduce cases where we get 500-iter fallback when better exists nearby. Slightly more CPU work during reference search (tests full path up to need per candidate when scanning), but search radius small (32px) and happens infrequently (when reference lost/replaced). Acceptable trade-off.

## Build
Compiles successfully with warnings only (unused inline removed if inlined; but we may still get warning - fixed by keeping logic). Build succeeded, binary created.

## Notes on attraction classifier (defect 3)
Current: averages log|2*|Z|| over tail of reference orbit. Measures orbit growth, not screen-wide survival. A repelling reference near a minibrot can have repelling orbit but most pixels in view are interior (long-surviving). So cap disengages when needed.

Better idea: track how many pixels actually survive to near-cap when using reference? Expensive to sample every frame. Alternative: during RefClassify or after settling, could sample N grid points in view (e.g. 16x16) using quick test with current refMax? If >50% survive to refMax-100, treat as "screen mostly interior" -> attracting regime for cap purposes. Or use a heuristic: if refAttracting (orbit) OR (span not tiny and we had to clamp iter hard due to many long escapes) - but we don't know yet.

But this is more invasive and needs measurement. Given AFK directive to make executive durable changes, perhaps better to add a lightweight screen-survival hint: sample 8x8 grid points around center (quick direct test up to small n) and if majority survive past 80% of refMax, boost the "attracting effective" flag for governor purposes. Or just extend RefClassify to consider neighborhood? Or store last frame's pixel survival stats.

Alternatively, track in governor: when refAttracting is false but StepGovernor had to shrink repeatedly and ms stayed high even with shrinking, that's a sign screen is hard. But not clean.

Executive decision: add a cheap neighborhood survival estimate. Implement simple 12x12 subsample test when reference adopted or every ~60 frames (static view) - cheap enough? 144 orbit tests * up to thousands iter worst case is bad. Or test up to min(refMax, 2000) iterations only - just to detect "long survivors".

Add field `refScreenInterior` (bool). Compute on RefClassify(): test 10x10 grid in current view? But view changes. Or test around ref point in small neighborhood - not quite same as screen.

Or better: after RefPrepare/when settled, sample screen corners and center (5 points) quickly with perturbation logic? No easier way. Maybe skip heavy change now; document properly. The user wants fixes - defect 1,2 done. Defect 3 is trickier; can note approach.
