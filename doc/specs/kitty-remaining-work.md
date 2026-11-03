# Remaining kitty work (fork, PR to microsoft was closed)

## 1. Terminal-driven animation playback
Spec: when an animation runs (a=a, s=2 loading / s=3 running), the terminal
advances frames on its own clock. Frame gaps come from each frame's gap
(default 40ms, root frame 0). Loops: v=1 infinite, v=n plays n-1 extra times.
On stop, the loop counter resets and frame 1 is shown.

Design:
- `KittyImages::TickAnimations(nowMs)` in KittyImages.cpp: for each image with
  animState 2 or 3 and frames.size() > 1: accumulate elapsed since
  `lastTickMs`; while elapsed >= gap(currentFrame): advance displayed frame
  (wrap, count loops, stop when done unless v==1). Return true if anything
  changed. Needs `lastTickMs` per image (add field, steady_clock ms).
- `SelectFrame` already copies frame -> pixels and bumps generation, reuse it.
- Hook: `Renderer::_PaintFrame` (renderer.cpp, before _PaintBufferOutput):
  call `buffer.GetKittyImages().TickAnimations(...)`; if true, call
  `pEngine->InvalidateAll()`/mark continuous redraw so the next tick paints.
  AtlasEngine has continuous-redraw support (cursor blink uses it): expose a
  flag so the renderer keeps presenting while any animation runs.
- Tests (KittyGraphicsTests): gap math, loop counting, v=1 infinite, stop
  resets to frame 1, s=1 no-op. Inject clock (pass nowMs in).

## 2. Payload sub-rectangle keys S / O
S = offset into the transmitted data, O = size, used with x/y/w/h to place a
sub-rectangle of the payload into the image. Parser: extend the f=24/32 path
in `_finishTransmission` to slice `bytes` by S (in pixels) before the expected
size check (expected = O.w*O.h*bpp). Add S/O to Command, u32, checked vs
image dims. Tests: offset blit via S.

## 3. Indexed formats f=8 / f=16
Palette transmitted first via a=1 special command (spec "transmit palette"),
then indexed payloads. Low value, do last.

## 4. File / stdin transmission t=f / t=s
Out of scope for the fork (security surface, no client needs it).

State: PR #20885 closed; fork branch feat/kitty-graphics is the product line
for rustel.cc. All suites green at 54f71a395.
