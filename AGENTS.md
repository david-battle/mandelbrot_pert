# mandelbrot_pert

Perturbation-theory Mandelbrot viewer. Split out of `~/raylib-test` so the deep
zoom work has its own context; `mandelbrot.c` (the plain fp64 explorer, which
stops at span ~1e-13) stays there and is *not* modified from here.

Goal: zoom far past the fp64 wall (span ~1e-13) on this hardware (1680x1050,
2080 Super, D3D12/Mesa, ~2 s GPU watchdog). Membership-exact against a
`__float128` oracle from span 1e-6 to **1e-50**.

## Conventions

- If the user sends any message after the agent started `pert`, assume they
  exited it — verify with `pgrep -x pert` instead of assuming it's still
  running. If a fresh copy needs to run and the old one is still alive, silently
  kill it first; never ask, and never offer to kill it.
- **`pkill -x pert` before verifying.** A second fullscreen instance contends
  for the GPU and makes shots segfault or dump half-rendered frames, which looks
  exactly like a numerical regression. See NOTES.md.

## Files

- `pert.c` / `pert` — the viewer. Renders `c = c0 + delta`, `z_n = Z_n + w_n`
  with `Z` (the reference orbit) shared across the frame as a `1024x1024` RGBA32F
  `sampler2D`, one `texelFetch` per iteration, NEAREST. `delta` is formed from
  the exact integer pixel offset, never `uv-0.5`. Paths: fp64 perturbation
  (settled frame), fp32 perturbation (interaction, ~60x faster), fp64 direct
  (fallback when no reference orbit exists, plus the in-app A/B oracle);
  `P` cycles. Float32 *direct* is deliberately dropped — it is the only one of
  the four that no depth helps. `--shot cx cy span iter path out.png` renders
  one frame headless (no string surgery) and writes `out.png` + `out.raw`.
  Sprite/GLSL 330 bloom is inherited from `raylib-test`'s `main.c` conventions.
- `pert_ref.c` / `pert_ref` — the CPU oracle: same recurrence and colouring in
  float32 / double / `__float128` via `-mode`, plus `-counts` for raw escape
  counts. Built with `gcc -O2 pert_ref.c -o pert_ref -lquadmath -lm`. It also
  holds the Phase 0 experiment switches (`-wprec 0..3`, `-rbase R`, `-refauto`)
  that the measurements in TODO.md came from; keep them, `-wprec 1` (pure quad)
  is the permanent proof that the recurrence is right.
- `pert_verify.sh` — renders the same view through both paths and prints
  membership / colour error. `CROP=120 SPANS="1e-6 1e-20" ./pert_verify.sh 480 270`.
- `pert_check.py` — stdlib-only diff of oracle RGB vs a shot's RGBA dump.
  Reports membership mismatches, how many interior pixels the crop actually
  contains, and mean channel error. (Copy of `raylib-test`'s
  `mandelbrot_check.py`; the two copies are independent, fix one and port it.)
- `TODO.md` — the design, the Phase 0 measurement tables, the phase results,
  the traps, and the open defects. Start here.
- `NOTES.md` — verification traps and the environment facts the constants are
  calibrated against.
- `DEEPZOOM.md` — a local, untracked personal reference copy of the Sun &
  Fraktinder deep-zoom writeup. Not our work; never committed here or upstream.
  It is excluded via `.git/info/exclude`.

## Two things not to undo

- **The reference must be a point whose orbit *survives*.** An escaping one
  silently truncates every pixel's loop to the same count, so an escaping
  reference makes every "exact N%" reading meaningless. `pert_ref` exits 3
  instead of reporting it.
- **`--shot` must not draw the HUD.** `pert_check.py` reads membership off the
  *colour* image, so glyphs inside the crop register as render errors — a clean
  29-row band of 916/14400 "membership errors" clustered exactly where the text
  sat looked precisely like a numerical failure band. This cost hours.
- **`FLAG_VSYNC_HINT` is load-bearing.** Without it the D3D12 swap returns before
  the GPU is done, `GetFrameTime()` reports only `SetTargetFPS` pacing, and the
  frame-cost governor is blind by construction.

## Reading the numbers

- Read `mem`, not `mean|d|`, past ~1e-4. At depth a hue band is only ~25
  iterations wide, so ±1 iteration of chaotic divergence moves the colour by
  tens while the sets agree exactly. `--shot ... 4` renders path 3's arithmetic
  with the iteration count in R/G and the interior flag in B — that is the
  honest deep comparison.
- But `mem 0` means nothing unless the crop straddles the boundary. At
  c=-1.7497 a 200x150 crop is 99.99% exterior (`interior 0/14400` at every
  offset), so two paths agreeing on an empty frame means both found nothing.
  Check `interior` is non-trivial, and only then believe `mem`; fall back to
  banding error otherwise.
- When the viewer stalls, flickers or renders less than it should, read the
  `state:` (every 120 frames) and `change:` (only when `iter`, `path` or
  `refMax` actually changes) trace lines before touching anything — with a
  static view those three are the only quantities that can vary, and they found
  two bugs that looked like the governor's fault.

## Building

Statically linked against the sibling raylib clone at `~/raylib`
(`src/raylib.h`, `src/libraylib.a`) — that repo is upstream `raysan5/raylib`,
not personal, do not push it:

    gcc -O2 -Wall -Wextra -I ~/raylib/src pert.c -o pert ~/raylib/src/libraylib.a \
        -lm -lpthread -ldl -lX11

Compiled binaries (`pert`, `pert_ref`, `*.o`) are gitignored; commit source
only. `pert_verify.sh` builds both of these itself.

## Sibling repos

- `~/raylib-test` — the raylib playground this came out of. `mandelbrot.c`
  (plain fp64 explorer, stops at ~1e-13) and its verification trio live there
  and are not touched from here; `main.c`, `ico.c`, `cell.c` etc. also live
  there. Its NOTES.md has the derivation of the direct-fp64 wall.
- `~/raylib` — upstream `raysan5/raylib` clone, not personal, do not push.

## Handoff procedure

Triggered by the user saying "handoff". End of a session. The agent does
everything here; the user pushes out of context afterward — **never push**.

1. Kill stray processes: `pkill -x pert` (holds an X window).
2. Triage stray files: check `git status` for untracked files and leftovers.
   For EACH one make an executive decision without asking the user: `git add`
   it if real content; add a `.gitignore` rule if a recurring build artifact or
   local scratch; delete it if leftover junk. `DEEPZOOM.md` is already excluded
   via `.git/info/exclude` and must never enter git history. Never leave
   untracked files unclassified, and never ask which to do.
3. Verify clean source: `git status` / `git diff`; commit source only.
4. Sanity-build if any C file changed (the gcc line above) so committed source
   always compiles.
5. Commit (short imperative summary, e.g. "Prefer long-surviving references").
6. The user pushes.