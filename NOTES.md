# NOTES — `pert.c`, perturbation Mandelbrot

Four things here cost hours and are all traps in any GL/oracle diff work. Read
them before re-investigating a "mismatch" in this repo.

- **The HUD was the bug, twice.** `pert_check.py`'s `mem` number is
  derived from the *colour* image, not from counts, so anything drawn on top of
  the frame counts as a render error. `--shot` was drawing four HUD lines whose
  glyphs landed inside the comparison crop: a clean 29-row band of
  "membership errors" (916 of 14400, all one-sided, clustered exactly where the
  text sits), which looks precisely like a numerical failure band and sent me
  hunting for a shader bug that did not exist. `--shot` now suppresses the HUD.
  *When a disagreement is spatially banded and one-sided, suspect an overlay
  before suspecting arithmetic.*
- **Read `mem` off the count encoder, not the colour.** `--shot ... 4` renders
  path 3's arithmetic with the iteration count in R/G and the interior flag in
  B. With that, every one of 108521 comparable pixels matched the oracle's
  count exactly and membership was 0/108521 — the renderer had been correct the
  whole time. The colour channel's per-iteration hue bands are why `mean|d|`
  sits at 70-130 at depth even when the sets agree perfectly.
- **Mesa's GLSL compiler constant-folds literal-only expressions in fp32**, even
  in a `#version 400 core` shader on a context that advertises
  `GL_ARB_gpu_shader_fp64`. Probed directly: `0.1 + 0.2 == 0.3` and
  `2.0 + 2^-30 == 2.0` both evaluate *true* (fp32 answers), while
  `double`-typed runtime work (`double w; for (10) w += 0.1; w != 1.0`, and
  `float h = 1.0; double(h) + double(2^-30) != 1.0`) is correct to fp64. So
  runtime fp64 is real, and this is very likely the same folding that makes
  mandelbrot.c's double-single `twoSum` degrade to fp32 there. Never write an
  fp64 assertion out of literals: force the values through a `float` variable.
- **raylib's `GetShaderLocation` ids are not GL uniform locations.** Passing them
  to `glGetUniform*` reads unrelated locations and returns plausible garbage —
  it cost a while because the numbers looked like real uniform values. Use
  `rlGetLocationId(shader.id, name)` if a GL location is genuinely needed.

And one confirmation worth keeping: the GPU's reference table is exact. A
fixed-delta run (delta as a uniform, so every fragment iterates the same orbit)
returned the same escape iteration as the `__float128` oracle to the last count,
which pins the table layout, the hi/lo float round-trip, and the fp64
recurrence all at once — and exonerated `delta`, the pixel offset, the uniforms
and the rasterizer in one shot. Isolating with a fixed delta is the move when a
per-pixel loop disagrees with an oracle.

## Environment facts the constants are calibrated against

Moved here from `raylib-test`'s NOTES.md because `pert.c`'s
`FRAME_BUDGET_MS`/`ITER_SLOPE` were tuned against them, and re-deriving them
costs a night.

- **Hardware/driver**: WSLg, 2080 Super, Mesa D3D12. Measured at 1680x1050.
- **The render target is 1680x1050, not 1920x1080.** raylib logs
  `Using monitor 1: rdp-2` at 1920x1080 while `GetRenderWidth/Height` report
  1680x1050. Anything that scales by pixel count (worst-case iteration
  estimates, the shot sizes in `pert_verify.sh`) must use
  `GetRenderWidth/Height()`, never `GetScreenWidth()`.
- **There is a ~2 s GPU watchdog that kills the GL context silently.** The
  context resets, every later frame renders nothing, and there is no driver
  message. Measured on `mandelbrot.c`'s fp64 path at 1680x1050, span 1e-6:
  8000 iters -> 0.58 s (renders), 12000 -> 0.84 s (renders), 20000 -> 1.37 s
  (renders), 30000 -> ~2.2 s (**black, context gone**), 40000 -> 2.35 s (black).
  Perturbation's watchdog characterisation (silent loss between 1.0 and 1.2
  s/frame) is the same wall seen from the other side. **Treat a black deep view
  as a suspected watchdog reset before suspecting the shader** — and note that a
  genuinely all-interior view is also legitimately black, so the frame time is
  what distinguishes the two. That is why `FRAME_BUDGET_MS` is 700 here, not
  1200: the budget must leave room for the frame that trips it.
- **`FLAG_VSYNC_HINT` is load-bearing, not cosmetic.** Without vsync the
  WSLg/D3D12 swap returns before the GPU is done, so `GetFrameTime()` reports
  only `SetTargetFPS` pacing and the frame-cost governor is blind *by
  construction*. Symptom when it is off: `iter` tracking `want` 1:1, every log
  line reading ~17 ms including frames that really took 747 ms, process idle in
  `hrtimer_nanosleep` while the GPU fell behind.
- **Do not force a sync with `glFinish()`.** `external/glad.h` is included for
  `glUniform2d`, but `gladLoadGLLoader` is never called, so glad's `glFinish`
  is an uninitialised pointer and the call jumps to 0 (ASan: `SEGV, pc 0x0`).
  `glUniform2d` survives only because rlgl happens to load it.
- **Never verify with the interactive viewer running.** A second fullscreen
  instance contends for the GPU; shots then segfault or dump a partially
  rendered frame, which reads exactly like a numerical regression. Observed as
  span 1e-20 banding drifting 8.17 -> 15.98 and 1e-50 segfaulting, both purely
  from running `pert_verify.sh` next to a live `./pert`. `pkill -x pert` first;
  the warning is at the top of `pert_verify.sh`.
- **On screen capture: nothing X-side can see these pixels.** ffmpeg
  `x11grab`, raylib `TakeScreenshot` and `XGetImage` all return black — even for
  plain non-GL Xlib clients — because on WSLg the real pixels live in the
  Windows-side compositor. Backbuffer dumps are the only ground truth; for
  eyeballing a window use Windows-side capture (Win+Shift+S).
- **For comparison with `mandelbrot.c`:** direct fp64 dies at span ~1e-13 from
  the `ulp(0.74)` add in the orbit, so this program is the only way past it.
  Both facts are derived in `raylib-test`'s NOTES.md section
  "Where direct fp64 actually stops working".
