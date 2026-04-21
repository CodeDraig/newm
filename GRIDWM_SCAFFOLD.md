# gridwm — scaffolding document

A scaffold for a new Rust X11 window manager inspired by newm's infinite 2D
grid UX. The name `gridwm` is a placeholder — replace throughout.

This document is meant to be copied into a fresh repo (or used as a design
note alongside one) and fleshed out into real code. It does **not** aim to
be runnable itself.

---

## 1. Goals & non-goals

**Goals**
- Treat the desktop as an infinite 2D grid of cells; each managed window
  lives in exactly one cell.
- Smooth panning of the viewport across the grid via touchpad gestures
  (3-finger swipe) and keybinds.
- Discrete "overview mode" that shrinks all windows to thumbnails arranged
  in a mini-grid, toggled by a keybind / 4-finger gesture.
- Static Rust config, xmonad-style (recompile to reconfigure).

**Non-goals (Option A, per scoping discussion)**
- No true continuous zoom. Overview is a mode switch, not an interpolated
  scale. Implementing real zoom would require becoming a compositing WM
  (Composite + XRender/GL) and roughly 4× the work.
- No Wayland. X11 only.
- No built-in compositor. Users run `picom` separately if they want
  transparency/shadows.
- No IPC / scripting surface for v1. Keybinds and config are in Rust.

---

## 2. Repo layout

```
gridwm/
├── Cargo.toml
├── Cargo.lock
├── README.md
├── LICENSE
├── rustfmt.toml
├── .gitignore
├── src/
│   ├── main.rs              # entry point, wires everything together
│   ├── config.rs            # static config: cell size, keybinds, colors
│   ├── grid.rs              # Grid<T>: cell coordinate ↔ T map
│   ├── viewport.rs          # Viewport state + tween animation
│   ├── layout.rs            # penrose Layout impl: places windows per grid
│   ├── overview.rs          # overview mode: resize-to-thumbnail logic
│   ├── gestures.rs          # XInput2 gesture reader (separate x11rb conn)
│   ├── actions.rs           # user-facing commands (focus_cell, move_to, …)
│   ├── hooks.rs             # penrose hooks: map/unmap → grid assignment
│   └── util.rs
├── tests/
│   └── grid_math.rs
└── examples/
    └── minimal.rs           # smallest runnable WM, for smoke-testing
```

---

## 3. Cargo.toml

```toml
[package]
name = "gridwm"
version = "0.1.0"
edition = "2021"
rust-version = "1.74"

[dependencies]
# Core WM framework. Check crates.io for latest 0.4.x patch.
penrose = "0.4"

# Needed for XInput2 gestures — penrose does not expose XI2 events.
# Open a second X connection on a dedicated thread to read them.
x11rb = { version = "0.13", features = ["xinput"] }

# Config / logging
anyhow = "1"
thiserror = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
proptest = "1"

[profile.release]
lto = "thin"
codegen-units = 1
strip = true
```

**Verify before committing:**
- Exact penrose 0.4.x patch version and whether its `Layout` trait is named
  `Layout` or something else (0.3 → 0.4 reshuffled traits). Run
  `cargo doc --open -p penrose` after first build.
- Whether `x11rb` still needs `allow-unsafe-code` or similar for XI2 — has
  varied across versions.

---

## 4. Core types

### 4.1 `grid.rs`

```rust
use std::collections::HashMap;

#[derive(Copy, Clone, Eq, PartialEq, Hash, Debug)]
pub struct Cell {
    pub col: i32,
    pub row: i32,
}

impl Cell {
    pub const ORIGIN: Cell = Cell { col: 0, row: 0 };
    pub fn step(self, dx: i32, dy: i32) -> Cell {
        Cell { col: self.col + dx, row: self.row + dy }
    }
}

/// Bidirectional map between cells and window ids.
/// Keep it tight: one window per cell, one cell per window.
pub struct Grid<W: Copy + Eq + std::hash::Hash> {
    by_cell: HashMap<Cell, W>,
    by_win:  HashMap<W, Cell>,
}

impl<W: Copy + Eq + std::hash::Hash> Grid<W> {
    pub fn new() -> Self { /* … */ }
    pub fn insert(&mut self, cell: Cell, win: W) { /* … */ }
    pub fn remove_window(&mut self, win: W) -> Option<Cell> { /* … */ }
    pub fn window_at(&self, cell: Cell) -> Option<W> { /* … */ }
    pub fn cell_of(&self, win: W) -> Option<Cell> { /* … */ }
    pub fn next_free(&self, near: Cell) -> Cell { /* spiral search */ }
    pub fn iter(&self) -> impl Iterator<Item = (Cell, W)> + '_ { /* … */ }
}
```

### 4.2 `viewport.rs`

```rust
#[derive(Copy, Clone, Debug)]
pub struct Viewport {
    /// Pixel offset of the viewport's top-left within the grid.
    pub x: f32,
    pub y: f32,
}

pub struct ViewportAnim {
    pub from: Viewport,
    pub to:   Viewport,
    pub t0:   std::time::Instant,
    pub dur:  std::time::Duration,
}

impl ViewportAnim {
    pub fn sample(&self, now: std::time::Instant) -> (Viewport, bool) {
        let raw = (now - self.t0).as_secs_f32() / self.dur.as_secs_f32();
        let t = raw.clamp(0.0, 1.0);
        let eased = ease_out_cubic(t);
        let v = Viewport {
            x: lerp(self.from.x, self.to.x, eased),
            y: lerp(self.from.y, self.to.y, eased),
        };
        (v, t >= 1.0) // (current, done?)
    }
}
```

### 4.3 `config.rs`

```rust
pub struct Config {
    pub cell_w: u32,
    pub cell_h: u32,
    pub gap:    u32,
    pub anim_duration_ms: u64,
    pub overview_scale: f32,        // e.g. 0.2
    pub gesture_swipe_fingers: u32, // 3
    pub gesture_overview_fingers: u32, // 4
    pub border_focused:   u32, // hex color
    pub border_unfocused: u32,
}

pub const CONFIG: Config = Config {
    cell_w: 1920, cell_h: 1080, gap: 8,
    anim_duration_ms: 200,
    overview_scale: 0.2,
    gesture_swipe_fingers: 3,
    gesture_overview_fingers: 4,
    border_focused:   0x88c0d0,
    border_unfocused: 0x3b4252,
};
```

Keybinds live next to this (penrose has a `map!` / bindings DSL — use it
directly; don't reinvent).

---

## 5. Key algorithms

### 5.1 Window placement per viewport

For every managed window `w` in cell `(col, row)`, with viewport at
`(vx, vy)` and monitor size `(mw, mh)`:

```
screen_x = col * cell_w - vx
screen_y = row * cell_h - vy
```

Windows whose `screen_x + cell_w < 0` or `screen_x > mw` (and y equiv)
are offscreen. X11 will happily accept offscreen coordinates; no need to
unmap them. Still, consider unmapping for very-far-offscreen clients to
reduce redraw cost on animation frames — measure first.

### 5.2 Pan animation loop

Penrose is event-driven, not frame-driven. For animation you have two
options:

1. **Simple (recommended for v1).** When a gesture ends, compute the
   target viewport, then spawn a short-lived thread that wakes every
   ~16 ms, computes the eased viewport, and calls a
   `reposition_all_windows()` function that issues `ConfigureWindow`
   requests via penrose's X connection. Join when done.

2. **Integrated.** Set a repeating timer source in penrose's event loop.
   Cleaner, but depends on 0.4's event-loop API — verify it exposes
   timer/idle hooks.

Issue one batched `x11rb` request per frame if possible; avoid
`flush()`ing between every window move.

### 5.3 Overview mode

On toggle-on:
- Save current viewport + each window's cell.
- Compute `grid_bbox` — the bounding `(min_col, max_col, min_row, max_row)`
  of populated cells.
- Assign each populated cell a slot in a screen-filling mini-grid.
- `ConfigureWindow` each one to its mini-slot rectangle.

On toggle-off:
- Restore the saved viewport.
- Re-run the normal placement pass; windows snap back to full-cell size.

Clients will re-layout on resize — that's unavoidable without compositing.
Accept the flicker.

### 5.4 Gestures (XInput2)

Penrose owns the main X connection and its event loop. XInput2 events
need an `XISelectEvents` call and a reader. Cleanest approach:

- Open a **second** X connection via `x11rb::connect()` on a dedicated
  thread.
- `XISelectEvents` on the root window for
  `XI_GesturePinchBegin/Update/End` and
  `XI_GestureSwipeBegin/Update/End`.
- Translate events to a `GestureEvent` enum and send through a
  `crossbeam_channel` to the main thread.
- Main thread drains the channel on each penrose event tick (or via a
  custom `Poll` fd if 0.4 supports it) and updates viewport state.

This keeps penrose's ownership of the primary connection intact while
still getting multitouch input.

---

## 6. Module responsibilities (one-liners)

| Module | Responsibility |
|---|---|
| `main.rs` | parse args, init tracing, build penrose `WindowManager`, install hooks, run |
| `config.rs` | static `Config`, keybind table, theming constants |
| `grid.rs` | cell↔window bidirectional map; free-cell search |
| `viewport.rs` | viewport state, tween, easing |
| `layout.rs` | penrose `Layout` impl; consults grid+viewport; emits placements |
| `overview.rs` | mode flag, thumbnail slot computation, enter/exit transitions |
| `gestures.rs` | XI2 reader thread, `GestureEvent` channel |
| `actions.rs` | `focus_cell_dir`, `move_window_dir`, `toggle_overview`, `center_on_focused` |
| `hooks.rs` | `on_map`: assign cell; `on_unmap`: free cell; `on_focus`: recenter viewport |
| `util.rs` | ease fns, lerp, small helpers |

---

## 7. Config & keybind sketch

```rust
// in main.rs, using penrose's binding DSL (verify exact macro name vs 0.4)
let bindings = penrose::map! {
    "M-h" => actions::focus_cell_dir(Dir::Left),
    "M-l" => actions::focus_cell_dir(Dir::Right),
    "M-k" => actions::focus_cell_dir(Dir::Up),
    "M-j" => actions::focus_cell_dir(Dir::Down),
    "M-S-h" => actions::move_window_dir(Dir::Left),
    // …
    "M-Tab" => actions::toggle_overview(),
    "M-Return" => actions::spawn("alacritty"),
    "M-q" => actions::kill_focused(),
};
```

---

## 8. Testing

Pure logic is easy to test — keep it out of the X-dependent modules:

- `grid.rs`: property tests with `proptest` — insert/remove invariants,
  `next_free()` never collides, `cell_of(insert(c,w))` round-trips.
- `viewport.rs`: `sample()` hits `to` exactly at `t = dur`, monotonic
  in each axis when from/to differ on that axis.
- `overview.rs`: slot assignment for N windows fills the screen without
  overlap for N ∈ {1, 4, 9, 20, 100}.

The X-dependent modules (`layout.rs`, `gestures.rs`, `hooks.rs`) are
best exercised in a nested X server. Use `Xephyr`:

```
Xephyr -br -ac -noreset -screen 1600x900 :2 &
DISPLAY=:2 cargo run
```

Put this in `dev/run-xephyr.sh`.

---

## 9. Build & run

```
cargo build --release
# install
sudo install -m755 target/release/gridwm /usr/local/bin/

# minimal .xinitrc
cat > ~/.xinitrc <<'EOF'
exec gridwm
EOF
startx
```

---

## 10. Phased plan

**M0 — walking skeleton (weekend)**
- Crate compiles, runs under Xephyr, manages windows in a dumb single-row
  layout. Prove penrose integration works.

**M1 — grid placement (weekend)**
- `Grid` + `Layout` impl. New windows land in `grid.next_free()`. Keybinds
  move focus between cells. No animation yet — viewport snaps.

**M2 — viewport animation (few days)**
- Tween between viewport targets on focus change. Decide animation loop
  strategy (see §5.2) and commit.

**M3 — overview mode (few days)**
- Keybind toggles mode. Resize-to-thumbnail works. Exiting restores.

**M4 — gestures (week)**
- XI2 reader thread. 3-finger pan drives viewport directly. 4-finger
  triggers overview. Tune deadzones and snap thresholds.

**M5 — polish**
- Borders, gaps, multi-monitor (probably just "one grid per monitor" for
  v1 — shared grid across monitors is much harder).
- Crash logs via `tracing`.
- Document the config.

---

## 11. Known risks / open questions

1. **Penrose 0.4 API.** I've scaffolded against the *shape* of penrose,
   not the exact symbols. First task on M0 is reconciling against the
   actual 0.4 docs: `Layout` trait path, hook signatures, binding macro
   name, how to access the raw X connection from inside a `Layout`.

2. **Animation loop hookup.** Penrose is event-driven. If 0.4 has no
   timer source in its event loop, the background-thread approach is the
   fallback. Confirm before M2.

3. **Overview flicker.** Clients reflowing on resize is ugly. Mitigation
   options if it's bad: freeze backbuffers via Composite redirect just
   for the overview transition (this starts inching toward Option B —
   keep the scope honest).

4. **XI2 gesture portability.** Requires a fairly recent X server and
   libinput driver. Document the minimum versions in README.

5. **Multi-monitor.** Easiest model: each monitor has its own independent
   grid and viewport. A single shared grid spanning monitors is elegant
   but requires deciding how cells map across the monitor gap.

6. **Focus stealing / `_NET_WM_*` compliance.** Penrose handles much of
   EWMH. Spot-check what it doesn't and decide whether v1 cares.

---

## 12. First commit checklist

- [ ] `cargo new gridwm` (binary)
- [ ] Copy this doc in as `DESIGN.md`
- [ ] Paste the `Cargo.toml` above and `cargo build` to pin versions
- [ ] Stub every file listed in §2 with a one-line `//! module doc`
- [ ] Add `rustfmt.toml` (2021 defaults + `max_width = 100` if you like)
- [ ] Add `.github/workflows/ci.yml` with `cargo fmt --check`, `cargo
      clippy -- -D warnings`, `cargo test`
- [ ] MIT or Apache-2.0 `LICENSE`
- [ ] README pointing at DESIGN.md
