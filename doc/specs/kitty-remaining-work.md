# Remaining kitty work (fork, PR to microsoft was closed)

## 0. APC/OSC/DCS passthrough mod (do FIRST, perf win, lhecker point 2)
Today every APC char goes: stateMachine _apcStringHandler(wch) at
stateMachine.cpp:1891 -> std::function lambda in AdaptDispatch ->
per-char append into KittyGraphicsParser (wstring payload). MB of base64
per frame pays a virtual call + push_back PER CHARACTER.
Plan (mirror of upstream OSC/DCS passthrough technique):
- StateMachine: in the ApcString state, instead of per-char handler calls,
  accumulate into a til::u16string buffer and flush in bulk. Minimum viable:
  ProcessString already sees string_view runs; make ApcString consume runs
  (stateMachine.cpp:1891 area, _Action... / _ParseLoop) by appending the
  whole run to a buffer, calling the parser once per run via a new
  engine hook (ActionApcRun(string_view)) alongside the per-char one.
- AdaptDispatch::ActionApcDispatch lambda: keep for foreign APCs; when
  kitty is enabled, route runs straight to KittyGraphicsParser.ParseRun.
- KittyGraphicsParser: add ParseRun(string_view) that loops chars WITHOUT
  the StringHandler indirection and reserves payload capacity up front
  (payload reserve = run length). Per-char Parse() stays for tests.
- Same treatment for OSC 52 style bulk if trivial, else APC only.
- Bench with bench-core.ps1 / bench-paint.ps1 before and after; target:
  cut parser CPU share during 60fps rustel playback noticeably.

## 0b. ImageSlice refactor (lhecker's finding, architectural)
Problem: kitty pixels are composed INTO the buffer rows' ImageSlice, which
is Sixel's baked-in model. Wrong for kitty: placements float over the text.
Symptoms we patched around: InvalidateRow, _composedRows cache, generation
bumps, EraseCells killing both sixel and kitty, one flattened bitmap per
row per layer (no independent animation), sixel+kitty on one row unsupported.
Design:
- Buffer keeps ONLY the placement list (KittyImages) plus placeholder text.
- Delete ComposeRow's writes into ImageSlice; delete ImageSliceLayer usage
  for kitty. Sixel keeps ImageSlice untouched.
- Renderer side: AtlasEngine::PaintImageSlice is replaced by a kitty path
  that composes per visible row from placements at paint time (reuse the
  _Strip scaler, move strips cache into a renderer-owned helper or keep on
  KittyImages but call it from the renderer pass). Per-row bitmap slots in
  BackendD3D/D2D stay (they are the actual pixels cache); only the source
  changes from row->bitmap/bitmapBelow to a renderer row cache keyed by
  (rowY, generation).
- Text writes stop touching kitty entirely: remove InvalidateRow hook and
  the EraseCells interplay for kitty layers.
- renderer.cpp _PaintBufferOutput: instead of reading row slices, iterate
  visible rows, ask buffer.GetKittyImages() for composed pixels of that row
  (new method ComposeRowPixels(rowY, cellSize) returning a span + revision),
  paint below before text, above after, exactly like today.
- Tests to keep green: adapter kitty suite (they call ComposeRow then read
  row.GetImageSlice) -> migrate assertions to the new accessor. 465 tests.

## 1. Terminal-driven animation playback (DONE, c67afb461)

## 2. Payload sub-rectangle keys S / O
S = offset into the transmitted data, O = size. In _finishTransmission,
slice bytes by S before the expected size check; expected = O.w*O.h*bpp
(spec S/O are pixel counts, see kitty docs "size and offset"). Add to
Command, u32, CheckedCoord vs image dims. Tests: offset blit.

## 3. Indexed formats f=8 / f=16 (last)
Palette via the a=1 command first. Low value.

## 4. File / stdin transmission t=f / t=s
Out of scope (security, no client needs it).

Order: 0, 0b, 2, 3. All committed to origin feat/kitty-graphics as undy-aeon.
Suits to run: adapter (465), parser, host, TextBuffer.

