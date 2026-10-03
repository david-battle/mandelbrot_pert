# Recommendations to fix GPU timeout from interior-heavy frames

The core tension: perturbation needs *some* bounded (set) points to anchor, but when a large contiguous block of interior pixels exists, almost every fragment runs to `maxIter` and blows the GPU budget.

## 1) Prefer repelling references over attracting ones (highest ROI)

- **Problem:** `RefFind()` currently prefers the closest point that survives `need` iterations (tie → higher count). When zoomed into a minibrot interior, it returns an attracting reference (`refGrowth < 0`), so the whole frame is bounded and every pixel hits the cap.
- **Fix:** Track `bestGrowth` in `RefFind()` (around lines 380–410). Score candidates by `(survival desc, growth desc, dist2 asc)` when choosing among those that meet the survival threshold. Bias toward `refGrowth > +0.05` even if a few pixels farther away.
- **Impact:** Many dense interior views become cheap (repelling reference) without losing the ability to find any anchor.

## 2) Estimate interior coverage (not just a 3x3 boolean)

- **Problem:** `screenInterior` only samples 9 points (lines ~758–770). Thin filaments or partial minibrots are misclassified.
- **Fix:** Add a cheap probe once per frame before iter/path selection: sample an `N×M` grid with `N*M ≤ 256` (e.g. 16×16) using `RefSurvivesCount(..., cap_probe)`. Compute `f_in = interior_count / total`. Define `hard` if `f_in >= 0.4`, `medium` if `0.15 ≤ f_in < 0.4`.
- **Actions:**
  - Compute `cap_eff = cap * (1.0 - k*f_in)` with `k ~ 0.6–0.8`, clamped to `[cap*0.25, cap]`. Apply before `StepGovernor()` and in the predictive clamp.
  - If `f_in > 0.3` and best candidate is attracting, re-search with radius 1.5× and filter to `growth > 0.05`. Fall back to attracting only if nothing repelling exists.
- **Impact:** Makes the governor honest about coverage; fixes first-frame stalls and reduces over-clamping.

## 3) Short-circuit expensive frames earlier

- **Problem:** Governor reacts after measuring a slow frame (lines ~793–807). Multi-second frames can still occur on first entry into a dense view.
- **Fix:**
  - Use `f_in` from the previous frame (1-frame lag, cheap) to set `want = min(want, cap_eff)` *before* `StepGovernor()`.
  - If `frameMs > FRAME_BUDGET_MS*0.9` and `f_in > 0.25`, apply a larger immediate shrink (e.g. `scale = 0.4`) instead of the gradual EMA-only response.
- **Impact:** Prevents the first catastrophic frame while keeping stability.

## 4) Treat "marginal reference + dense screen" as a direct path trigger

- **Problem:** Auto path (lines ~815–838) can still choose perturbation when the reference is marginal/attracting but screen is dense.
- **Fix:** Extend the condition: if `f_in > 0.5` and (`refAttracting` or `refShortFor > 0`), prefer `PATH_FP64_DIRECT` at spans where direct is viable (`span > ~5e-12`). Keep `PATH_PERT32` for motion only when reference is repelling/coverage low.
- **Impact:** Avoids perturbation's worst-case cap cost above the direct-fp64 precision threshold.

## Concrete touchpoints (pert.c)

| Area | Location | Change |
|---|---|---|
| Reference scoring | `RefFind()` (≈360–410) | Add `bestGrowth`; prefer higher growth when survival counts are comparable |
| Coverage probe | New helper (before iter choice) | 16×16 grid via `RefSurvivesCount`; compute `f_in`, `cap_eff` |
| Predictive clamp + governor | `main()` (≈750–810) | Replace 3×3 with probe; use `cap_eff`; add previous-frame clamp; consider aggressive shrink on near-overrun |
| Path selection | Auto path (≈815–840) | Add coverage+attracting/marginal trigger to prefer direct fp64 when viable |

## Recommendation

**Adopt (1) and (2) first.** (1) fixes reference choice so dense views become repelling/cheap; (2) makes the governor coverage-aware. (3),(4) are robustness tweaks. Together these preserve anchoring (falling back to short/interior reference only if no repelling one exists) while preventing GPU timeouts.
