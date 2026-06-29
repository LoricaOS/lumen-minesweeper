# lumen-minesweeper

The Minesweeper game for **AspisOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

minesweeper is the classic mine-hunting game: a 9x9 covered grid hiding 10
mines, left-click to reveal, right-click to flag, with flood-fill on empty
cells and first-click safety. It is a leaf component of the Lumen desktop,
distributed as a [herald](https://github.com/AspisOS/AspisOS) package, and runs
as an **external client** of the [lumen](https://github.com/AspisOS/lumen)
compositor — it connects to `/run/lumen.sock` over the Lumen window protocol and
is handed a shared-memory buffer to draw into, rather than being an in-process
compositor built-in.

## Where minesweeper fits

AspisOS is decomposed into independent repositories. minesweeper sits at the
leaf of the graphical stack:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel: capability model, `AF_UNIX` sockets, `memfd`, the syscalls the desktop runs on. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit minesweeper links against: the software renderer (`draw_*`, `font_*`), theme/accent values, and the client side of the Lumen protocol (`lumen_client.h`). |
| `AspisOS/lumen-minesweeper` | **This repo.** The minesweeper app. |

## What it does

Grounded in `src/main.c`:

- Opens a fixed **352x412** window titled "Minesweeper" via
  `lumen_window_create` (`WIN_W`/`WIN_H` = a `9x9` grid of `36`px cells plus a
  `60`px header and `14`px margins) and draws the board and header into the
  shared surface.
- Pure userspace, integer-only logic — a **9x9 board (`COLS`/`ROWS`) with 10
  mines (`MINES`)**, stored as two `uint8_t[ROWS][COLS]` arrays (a `F_*` bitmask
  per cell plus a precomputed adjacent-mine count). No new kernel support, no
  file I/O, no syscalls beyond the compositor socket.
- **First-click safety:** the mine field is generated lazily on the first
  reveal (`place_mines` runs from `reveal_cell` only when `g.placed` is unset),
  and the clicked cell and its 8 neighbours are kept mine-free so the first
  click always opens a region.
- **Left-click reveals, right-click flags** (`LUMEN_MOUSE_DOWN`,
  dispatched on `ev.mouse.buttons`): `reveal_cell` uncovers a cell; stepping on
  a mine calls `lose`, which marks the `F_BOOM` cell and reveals every mine.
  `toggle_flag` plants or clears a flag on a covered cell.
- **Flood-fill of empty cells:** revealing a cell with zero adjacent mines runs
  `flood_reveal`, an iterative flood-fill over an explicit `NCELLS` stack (no
  recursion) that opens the contiguous empty region and its numbered border.
- **Win/lose detection:** `check_win` declares a win once every non-mine cell is
  revealed (`revealed_count >= NCELLS - MINES`) and auto-flags the remaining
  mines; the smiley header face (`draw_face`) tracks state, and a banner across
  the grid reads "You win!" or "Game Over".
- **Restart** by clicking the smiley face (`hit_face`) or pressing `R`/`r`
  (`new_game`); Esc or the window close request quits.
- Rendering is classic-Minesweeper styling drawn with glyph primitives —
  bevelled covered tiles (`bevel`), per-number colours (`s_num_color`, blue
  through grey for 1..8), hand-drawn mine and flag glyphs, and two header
  counters (mines-remaining and revealed-cell progress). Text uses the TTF font
  when available and falls back to the bitmap font (`text_sz`/`text_w`).

## Capabilities

AspisOS grants a process no ambient authority; it can touch the system only
through capabilities declared for it at exec time. minesweeper's policy
(`pkg/etc/aegis/caps.d/minesweeper`) is the baseline:

```
service
```

The `service` profile and **no** elevated capabilities — minesweeper is pure
integer compute over a window surface and touches nothing beyond the compositor
socket.

## Status

minesweeper is intentionally small: a single, fixed Beginner board (9x9, 10
mines). It is early-stage and could grow (selectable difficulty / board sizes, a
game timer, chord-clicking) as AspisOS matures. What ships today is complete and
honest about its scope rather than feature-padded.

## Building

minesweeper builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/AspisOS/glyph) toolkit artifact (the GUI libraries it
links), then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `make` runs `tools/fetch-glyph.sh $(GLYPH_VERSION)` to download and unpack the
  pinned toolkit into `toolkit/`, compiles `src/*.c` against it, then packs.
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA-P256 key that signs the `.hpkg`.
- `GLYPH_VERSION` pins the toolkit release; `VERSION` is this app's own version.

Output: `lumen-minesweeper.hpkg` (a `class=system` herald package) +
`lumen-minesweeper.hpkg.sig`.

## Package payload

`lumen-minesweeper.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-minesweeper`) deliberately differs from
the bundle/exec name (`minesweeper`), and it installs across two trees — which is
exactly why it is `class=system` (first-party, signature-trusted, installed
verbatim) rather than an ordinary single-prefix package:

```
/apps/minesweeper/minesweeper   the app binary
/apps/minesweeper/app.ini       the bundle descriptor (name=Minesweeper, exec=minesweeper)
/etc/aegis/caps.d/minesweeper   its capability policy
```

## Repository layout

```
src/        minesweeper source (main.c)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — minesweeper is an external client of the compositor, so
installing it pulls [lumen](https://github.com/AspisOS/lumen). lumen also ships
the desktop fonts (Inter, JetBrains Mono), so minesweeper inherits them
transitively; there is no separate font package.
