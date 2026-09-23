# Terminator — vision

A terminal emulator written in MetaScript. It gets its VT from **libghostty-vt**, draws its pixels
with **void2d**, and runs in an **Ion** window, on Windows, macOS and Linux with the same pixels
everywhere.

## Why it exists

Two goals, and the second is as important as the first.

1. **A real terminal**, the kind someone uses all day: correct VT, fast output, good text,
   IME, selection, scrollback, links.
2. **The first production client of void2d.** void2d aims to render application UI at the level of
   a modern code editor: a 13 px code font at 1× and 1.5× DPI that is crisp and scrolls smoothly.
   A terminal presses hard on exactly what that takes: glyph atlas pressure, font fallback (CJK,
   emoji, box drawing), thousands of coloured cells per frame, dirty-only redraw, scroll, and a
   truly idle frame.

## Principle: fix upstream, never work around

When Terminator hits a defect in void2d, Ion or the MetaScript compiler, the fix goes into that
project, and Terminator waits for it. It never works around the defect in its own code. A workaround
here would hide exactly what this project exists to exercise. Terminator builds against released
dependencies only.

## The stack

| project | role |
|---|---|
| **MetaScript** (`msc`) | the language and compiler Terminator is written in |
| **Void / void2d** | the renderer: a retained 2D node tree over sokol_gfx, one GLSL source for Metal, D3D11, GL/GLES3, WebGPU and WebGL2 |
| **Ion** | the desktop runtime: window, event loop, a native render surface, input |
| **libghostty-vt** | the VT emulation library extracted from Ghostty |
| **Neon** | the MetaScript UI framework; not used for the grid (see Open) |

Ion hands out a surface and input events and never draws. The glue between Ion's surface and void2d
belongs to Terminator.

## libghostty-vt — only the parts we need

libghostty-vt exposes a C API (`include/ghostty/vt.h`) and is built with `zig build`
(Zig 0.16). It is vendored as a git submodule pinned to a commit. Upstream marks the API as
unstable, so moving the pin is its own commit, with the build and tests re-run.

Take:
- `terminal.h`, `screen.h`: VT state, scrollback, reflow on resize.
- `render.h`: an incrementally updated render state with dirty-row iteration
  (`ghostty_render_state_row_iterator_next_dirty`). This is the seam to void2d's retained tree: a
  dirty row updates that row's node and nothing else.
- `key.h`, `mouse.h`, `focus.h`, `paste.h`: input encoding, the Kitty keyboard protocol included.
- `osc.h`, `sgr.h`, `selection.h`, `search.h`, `modes.h`, `color.h`.

Do not take: Ghostty's renderer, font stack and app runtime. Those are void2d's and Ion's jobs.
Kitty graphics is a later decision.

**libghostty-vt has no PTY.** Terminator owns ConPTY on Windows and `forkpty` on POSIX.

## How a frame is drawn

- One node per visible row, owning that row's cells. Only the rows the render state marks dirty
  are re-emitted.
- Glyphs sit on an **integer cell grid**: monospace cells with whole-pixel origins.
- Cell backgrounds, underline (straight and wavy), strikethrough, selection and cursor are
  instances of void2d's UI pipeline. The cursor and the selection are their own instances, so a
  blink touches no text.
- Box drawing, blocks, braille and powerline glyphs are drawn procedurally rather than taken from
  the font.
- Scroll is a snapped translation of the grid rather than a rebuild. When nothing changed, a frame
  draws nothing.

## References

- **Ghostty**: the VT, used as a library, and its renderer as a reference for the font stack,
  glyph atlas and frame discipline.
- **WezTerm**: Windows ConPTY handling, font fallback configuration and multiplexing.

## Platforms

- **Windows first**, then **Linux** and **macOS**.
- Every platform draws the same pixels, since void2d has no OS text system and no per-platform
  text path.
- A platform that has not been run is reported as not tested, never as passing.

## Open

- **Neon or void2d directly for the chrome.** The grid is drawn straight on void2d, since it needs
  no component model. Tabs, splits, a settings panel and a command palette could be Neon on Void,
  or Ion's webview.
- **Ion's surface on Windows**: a child window or a DirectComposition swapchain. This matters only
  if a webview ever sits above or below the grid.
- **Ion's input contract** for keys, text, IME, focus and DPI. Terminator is its first consumer.
- **A web build.** void2d already builds for WebGPU and WebGL2, and libghostty-vt has a wasm
  target. Not a goal yet.
