# Rubik's Cube Solver - Legacy Svelte 5 UI + Rust/WebAssembly two-phase engine

> **Frozen snapshot.** This is the original Svelte 5 UI taken from commit `d528ef9` of the solver
> repository and kept for reference. Nothing here is deployed and there are no deployment
> workflows, so a push to `main` cannot publish anything. The current, Rust-only crate lives at
> <https://github.com/pattygcoding/Rubik-Cube-Solver>.

An interactive 3D Rubik's Cube you can *paint* (by clicking stickers on the cube or on an
unfolded net) and solve with a Herbert Kociemba style **two-phase search** compiled from Rust
to WebAssembly. The answer is played back move by move, with a move timeline you can scrub,
step through forwards and backwards, and re-play at any speed.

Source of this snapshot: <https://github.com/pattygcoding/Rubik-Cube-Solver-Legacy-Svelte>

```
┌──────────────────────┐   wasm-bindgen    ┌───────────────────────┐
│  Rust two-phase      │ ────────────────► │  Svelte 5 + three.js  │
│  solver (wasm)       │ ◄──────────────── │  painting + playback  │
└──────────────────────┘   facelet strings └───────────────────────┘
```

## Quick start

```bash
npm install          # SvelteKit, three.js and the wasm-pack binary
npm run wasm:build   # compile the Rust crate into src/lib/wasm-solver
npm run dev          # http://localhost:5173
```

The generated WebAssembly bindings are part of the repository, so `npm run dev` works right
after `npm install`. Re-run `npm run wasm:build` whenever `wasm-solver/src/lib.rs` changes.

### Building the WebAssembly module

The exact command (also available as `npm run wasm:build`) is:

```bash
wasm-pack build wasm-solver --target web --release --no-pack \
  --out-dir ../src/lib/wasm-solver --out-name rubix_wasm_solver
```

`--target web` produces an ES module whose default export is `init()`, which
`src/lib/solver.ts` awaits before touching any export:

```ts
import init, {
  solve_cube,
  js_apply_moves,
  warm_up
} from './wasm-solver/rubix_wasm_solver.js';

await init();                            // instantiate the .wasm module
warm_up();                               // build the lookup tables (~250 ms)
const moves = solve_cube(facelets, 22);  // "R U R' U' F2 ..."
```

### Requirements

| Tool | Version used |
| --- | --- |
| Node | 22 |
| Rust | 1.98 (`rustup target add wasm32-unknown-unknown`) |
| wasm-pack | 0.15 (a dev dependency, so `npx wasm-pack` works) |

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | SvelteKit dev server |
| `npm run build` | production build |
| `npm run check` | `svelte-check` (TypeScript + Svelte diagnostics) |
| `npm run wasm:build` | release build of the Rust crate into `src/lib/wasm-solver` |
| `npm run wasm:dev` | debug build of the Rust crate (compiles faster, runs slower) |
| `npm run wasm:test` | `cargo test` for the solver (15 tests, about a second) |

## The solver

The Rust crate is split by responsibility; `lib.rs` is only the crate root that
declares the modules and re-exports the public API.

| File | Contents |
| --- | --- |
| `wasm-solver/src/lib.rs` | crate docs, module declarations, public re-exports |
| `wasm-solver/src/geometry.rs` | faces, the integer grid, the 54 stickers and the 20 cubie slots |
| `wasm-solver/src/facelets.rs` | facelet string parsing, formatting, permutations |
| `wasm-solver/src/combinatorics.rs` | permutation and subset ranking, parity |
| `wasm-solver/src/cubie.rs` | the cubie level view of a cube (which cubie is where, twist/flip) |
| `wasm-solver/src/moves.rs` | move encoding, face turns, Singmaster notation, `apply_moves` |
| `wasm-solver/src/coordinates.rs` | phase coordinates and the canonical state each value stands for |
| `wasm-solver/src/tables.rs` | the move and pruning tables (the only heavy part) |
| `wasm-solver/src/search.rs` | the two-phase iterative deepening search |
| `wasm-solver/src/solver.rs` | validation and the solve entry points |
| `wasm-solver/src/wasm_api.rs` | the thin `#[wasm_bindgen]` wrappers |
| `wasm-solver/src/tests.rs` | the test suite for the crate as a whole |

The search itself:

* **Phase 1** drives the cube into the subgroup `G1 = <U, D, R2, L2, F2, B2>`: all corners
  untwisted, all edges unflipped and the four middle layer edges back in the slice. It is
  guided by the `twist (2187) x slice (495)` and `flip (2048) x slice (495)` pruning tables.
* **Phase 2** finishes the cube inside `G1` using `U, U2, U', D, D2, D', R2, L2, F2, B2`,
  guided by `corner permutation (40320) x slice permutation (24)` and
  `UD edge permutation (40320) x slice permutation (24)`.
* The phase-1 diameter is 12 and the phase-2 diameter is 18, so **every** solvable cube is
  solved within 30 moves and the search is complete. The engine first looks for an answer
  within the caller's `max_depth`, then falls back to the 30 move bound which always
  succeeds, so the UI always gets an answer.

Every lookup table is *derived* at runtime from the cube's geometry: the position and normal
of all 54 stickers, the eight corner slots and the twelve edge slots. Face turns are obtained
by rotating that geometry, and coordinate transitions come from small canonical states (twist
one corner in place, flip one edge in place, permute cubies). There is no hand written lookup
table anywhere, and the unit tests compare the derived tables against the classic unfolded net
layout, so a typo cannot produce a solver that disagrees with itself.

### API

| Export | Purpose |
| --- | --- |
| `solve_cube(facelets, max_depth)` | `"UUU…BBB"` to `"R U R' U' F2"`, or `""` when already solved |
| `solved_facelets()` | the reference facelet string |
| `js_apply_moves(facelets, moves)` | apply a move list to a facelet string |
| `js_move_indices(moves)` | `result[displayed] = original`, so painting works while a solution is shown |
| `js_is_solved(facelets)` | cheap "is this cube solved" check |
| `js_validation_error(facelets)` | the first thing that stops the cube from being solved, or `undefined` when it is solvable (no search, so it is cheap enough to call on every paint) |
| `move_index(token)` / `move_turn(index)` | Singmaster token to axis, layer and quarter turns, so the 3D view can never disagree with the solver about which way a face turns |
| `warm_up()` | build every table up front |
| `table_bytes()` | bytes of lookup tables currently held (about 5.5 MB) |

`solve_cube` and `js_validation_error` share one code path (`checked_state`), so the UI and
the solver can never disagree about whether a cube is usable. Both return an informative error
instead of panicking for: a facelet string that is not 54 characters, unknown characters,
wrong sticker counts, a displaced centre, an impossible cubie, a single twisted corner, a
single flipped edge, and mismatched permutation parity - and every message says what to do
about it.

## Invalid cubes

The app tells you what is wrong *while you paint*, rather than only when you press Solve.

`CubeState.validation` classifies the cube into four states, and the Solve panel shows the
verdict next to a matching icon:

| State | Meaning | Shown as |
| --- | --- | --- |
| `incomplete` | stickers are still unpainted | `… 31 of 54 stickers painted…` |
| `counts` | a colour does not appear exactly nine times | `… White 8 (too few), Red 10 (too many)…` |
| `impossible` | every sticker is there, but the cube could not exist (twisted corner, flipped edge, parity mismatch, duplicate cubie, moved centre) | `! Invalid cube: the corner twists add up to 2…` |
| `ok` | solvable | `✓ All 54 stickers painted and the cube is solvable.` |

The first two are worked out from the stickers alone, so they are known even before the engine
has loaded; `impossible` comes from the engine's validator. On top of the message:

* **Solve** is disabled whenever the cube is not solvable, and its tooltip is the reason.
* The palette shows a **sticker count per colour**; any colour that is not nine is highlighted.
* **Scramble** is the way out of a mistake. Random moves preserve whatever is already there, so
  an unsolvable cube would stay unsolvable forever - instead, an unsolvable cube is reset to a
  solved cube first and then scrambled, and the UI says so:

  > This cube could not be solved, so it was reset to a solved cube and then scrambled with 22 random moves.

  Scrambling a cube that *is* solvable keeps the existing behaviour and permutes it in place.

## The frontend

| File | Responsibility |
| --- | --- |
| `src/lib/solver.ts` | loads the WebAssembly module (browser only), adapts errors, converts between sticker colours and facelet strings |
| `src/lib/cube-geometry.ts` | sticker and cubie geometry shared by the 3D view and the net, plus "which cubie is in this slot and how is it turned" |
| `src/lib/CubeState.svelte.ts` | the reactive store (runes): painted colours, solution, cursor, playback and transitions |
| `src/lib/components/CubeView.svelte` | three.js scene, raycast click-to-paint, OrbitControls and the pivot animation |
| `src/lib/components/NetView.svelte` | collapsible 2D unfolded net for painting |
| `src/lib/components/Controls.svelte` | palette, solve, playback, speed slider and move timeline |
| `src/routes/+page.svelte` | page layout, the home mark back to pattygcoding.com and SCSS |

### The home mark

The chip beside the title is the same link every other pattygcoding.com project carries in the same
corner: the house glyph from the Feather icon set, pointed at `https://pattygcoding.com` in a new
tab, with the same treatment the Connect Four, Turing Machines, Scripting Library and Blazor sites
give it - a 32px square, a 6px radius, a hairline border, a muted icon, and the mint accent on
hover and focus. Only the token names differ, because those sites are Tailwind and this one is
plain SCSS: `ink-950`/`line`/`fg-muted`/`accent` here are `rgba(13, 16, 22, 0.75)`/`#1e2431`/
`#7c869a`/`#2dd4bf`. Keep the glyph, the destination and the four states together if it ever moves.

### How the 3D view stays in sync

The cube is 26 `BoxGeometry` meshes on the integer grid, each with six materials (three.js
order `+x, -x, +y, -y, +z, -z`). A mesh stands for one *slot*: its position and rotation come
from the displayed state, and its sticker colours come from the same state, so the two are
always applied together.

A face turn attaches the nine meshes of a layer to a temporary `THREE.Group` at the origin,
rotates that group by a multiple of 90 degrees, then detaches the meshes back into the scene,
rounds their positions and quaternions onto the grid (so hundreds of turns cannot accumulate
float drift) and re-applies the state. Because every mesh is an identical box, moving them
between slots is invisible - only the stickers matter.

Three constraints in that engine are easy to get wrong and are worth knowing about:

* A turn is animated along the **shortest path**. The solver encodes `F'` as three quarter
  turns, but playing that literally spins the layer 270 degrees, so it is normalised to -90.
* The black box that fills the middle cell of the grid is `1.04` wide, deliberately *smaller*
  than the 1.06 a sub cube would need before its inner faces touch: the sub cubes are 0.94 wide
  on a grid of spacing 1, so their inner faces are 0.53 from the centre. A core any larger pokes
  into them, slices through the layer that is turning and makes the cube look hollow. Just under
  it still blocks the 0.06 wide channels that run straight through the cube, so the page never
  shows through a gap.
* A face with no sticker at all (the inside of the cube) and a face whose sticker has not been
  painted yet are *not* the same thing. The first is dark, the second is an empty grey sticker,
  and `cubieFaceColors` reports them as `null` and `UNKNOWN_STICKER` so the two cannot be
  confused. Treating an unpainted sticker as "no sticker" paints it black and the cube looks
  full of holes.

Painting uses a `Raycaster`: the grid point of the hit sub cube plus the world normal of the
hit face identify the sticker, which is exactly how the facelet tables are keyed. A press only
paints when it was short and still, so orbiting never paints. While a solution is displayed,
clicks are mapped back through the engine's composed move permutation, so editing always
affects the sticker you clicked.

In development builds `window.__cubeView` exposes the scene, camera, meshes, current
placements and the pick function for debugging from the browser console.

### Keyboard

`1`-`6` select a palette colour, `0` (or `e`) selects the eraser. Click paints, right click or
shift click erases, drag orbits, scroll zooms.

## Transport controls

The transport follows the *painted* cube, not how far a solution has been played, so it always
reflects what the user can actually do next:

| Control | Enabled when | What it does |
| --- | --- | --- |
| **Play** / Pause | the painted cube is solvable and not already solved | plays the solution, solving the cube first when no solution has been found yet. A solution that has already been played to the end starts again |
| Step back / forward | a solution exists (and there is somewhere to step) | applies or undoes exactly one move, animated |
| Restart | a solution exists and is not already at the start | jumps back to the painted cube, keeping the solution |
| Solve cube | the painted cube is solvable | fills the move list without playing |
| Scramble | the engine is ready | 22 random moves; resets an unsolvable cube to a solved one first |
| Speed | always | 100-800 ms per move. The track reads slowest on the left to fastest on the right, so dragging right speeds the playback up; the readout shows the milliseconds per move |

So Play is available as soon as you have something to solve: scramble the cube (or paint it) and
press Play to watch it solve itself, with no need to press Solve first. While the cube is
unpainted, miscounted or physically impossible, Play is disabled and its tooltip is the reason.

## Tests

```bash
npm run wasm:test   # 17 Rust tests: geometry, tables, round trips, random scrambles, validation
npm run check       # Svelte and TypeScript diagnostics
```

The Rust suite solves batches of random scrambles and verifies every answer by replaying it,
and checks that `validation_error` and `solve_cube` always agree - including on 400 random
54-character strings built from `U R F D L B ?`.
`cargo test --release statistics -- --ignored --nocapture` prints table build time and solution
length statistics: median about 20 moves and roughly 10 ms per solve on a desktop CPU, with the
lookup tables taking about 250 ms to build in the browser.

## Deployment

There is none, deliberately. This snapshot ships without the original
`.github/workflows/deploy.yml` and without `static/CNAME` and `static/.nojekyll`, so a push to
`main` runs no workflow and publishes nothing, and Pages is not enabled on this repository.

The site is still a plain static build, so it runs entirely locally:

```bash
npm ci
npm run check
npm run build
npx vite preview          # or point any static host at build/
```

The wasm-bindgen output is committed under `src/lib/wasm-solver/`, so a build needs Node only - no Rust
toolchain and no wasm-pack. Re-run `npm run wasm:build` locally whenever the crate changes, and
commit the regenerated bindings with it.

`wasm-pack` drops a `.gitignore` holding a single `*` into its output folder, so the four binding
files are tracked by force-add - `git add -f src/lib/wasm-solver` - and that generated file is
deliberately not committed. Tracked files ignore `.gitignore`, so an ordinary `git add -A` keeps
updating them from then on.

`vite.config.ts` uses `@sveltejs/adapter-static` and `src/routes/+layout.ts` sets
`prerender = true`, so the build output is plain HTML plus assets. Everything that needs a browser -
the WebAssembly engine and the three.js scene - starts in `onMount`, which is what makes the page
prerenderable at all.

Bringing Pages back would be exactly the two things this snapshot dropped: the deleted
`.github/workflows/deploy.yml` job that uploads `build/` with `actions/upload-pages-artifact`, and
the one-time switch that makes a repository publish from GitHub Actions instead of from a branch
(`gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow`). The custom domain went with
it, so `static/CNAME` and `static/.nojekyll` are gone as well.

## Known limitations

* This is a two-phase search, not an optimal solver: answers are usually 20-22 moves.
* `max_depth` is honoured when possible. If nothing fits, the engine returns the shortest
  answer it can find under the 30 move bound and the UI says so.
* The net uses the classic Kociemba layout, where the top row of `U` is the back of the cube
  and the top row of `D` is the front, so the cross folds up without mirroring anything.
* `adapter-auto` cannot detect a deployment target here; choose a concrete adapter
  (`@sveltejs/adapter-node`, `-static`, …) before deploying.
