# Maze 95

A full-screen web recreation of the Windows 95 **3D Maze** screensaver — playable,
customizable, freshly generated on every load, and drivable by a browser agent
through [WebMCP](https://github.com/webmachinelearning/webmcp).

**Play it: https://cheneytsai.github.io/maze95/**

Or open `index.html` locally. That's the whole project: one file, no build step, no
dependencies. GitHub Pages serves the `gh-pages` branch; pushing to `main` copies the
site across.

## Why it isn't a port

The demo at [maze95.js.org](https://maze95.js.org/) is built on
[maze95/maze95-js](https://github.com/maze95/maze95-js) (three.js), which was archived in
November 2023 with **no license**, and its wall art comes from Microsoft's original
screensaver. [x86matthew/Playable3DMaze](https://github.com/x86matthew/Playable3DMaze) is a
Windows-only loader that DLL-injects and patches Microsoft's original `MAZE.SCR` at
hardcoded offsets — reverse engineering of a binary, with no web code and no license
either. None of it is redistributable, so nothing here is copied from any of them. The maze
generator, the WebGL renderer, and every texture in this page are written from scratch;
the textures are drawn procedurally into a canvas at load time, so the repository ships
no image assets at all.

## What's in it

**Renderer** — a ~200 line WebGL layer: one shader, interleaved position/UV/brightness
vertices, per-face shading, and squared distance fog that gives the corridors the
torch-lit falloff the original had. Walls, floor, ceiling, plaques and the exit portal
are separate batches; the drifting polyhedra are rebuilt into a dynamic buffer each frame.

**Maze** — recursive backtracker over an `W × H` grid, optionally *braided* (a share of
dead ends knocked through) so corridors form loops. Start and exit are the two furthest
apart cells, found with two breadth-first sweeps.

**Four ways to move**
- *Nobody, until asked* — the maze opens standing still. It never walks on its own, so an agent's first tool call starts from a known state.
- *Screensaver* — the left-hand wall follower, turning and gliding cell to cell like the 1995 original. Press the taskbar button to start it; it never starts itself.
- *Player* — `W A S D` / arrows, `Shift` to run, mouse look under pointer lock, `Esc` to hand it back. Touch gets a thumbstick and drag-to-look.
- *Agent* — tool calls walk the camera along a path and resolve when the walk finishes.
- Set an idle timeout and the screensaver takes over after a player stops moving, exactly like the real thing. It defaults to off, and it never interrupts an agent.

**Faithful by default** — it opens the way the 1995 screensaver looked: red brick with
grey mortar, mottled grey stone above and below, a perfect maze with no loops, and the
original's gentle light fade rather than a torch. **Classic 95** in the properties sheet
goes the whole way — no exit, no goal, no stopping, just the wall follower. `Tab` toggles
the overhead map, as in [x86matthew/Playable3DMaze](https://github.com/x86matthew/Playable3DMaze),
which patches the real `MAZE.SCR` binary to make it playable.

**Customizable** — grid size, loop density, five wall styles, seven floor styles, six
ceiling styles (including a roofless night sky), two lighting curves, light range, field
of view, wall height, render scale, walk speed, mouse sensitivity, scanlines, head bob,
map reveal, and the idle timeout. Settings persist in `localStorage`; the maze seed
deliberately does not, so every load is a new maze. `?seed=ABC123` reproduces one.

## Agent tools

The same ten tools are offered over three channels, because they are genuinely
different things and only one of them works in any given place:

1. **`navigator.modelContext`** (WebMCP) — a *browser* API, and still an early preview.
   In Chrome it needs 146 or newer with `chrome://flags/#enable-webmcp-testing` set to
   Enabled, the browser relaunched, and an HTTPS page; it is never exposed inside a framed
   viewer. Tools are registered with `registerTool()`, falling back to `provideContext()`
   on older drafts. Registration is retried for 20 seconds and on window focus, so an API
   that appears after load — a flag-gated context finishing initialisation, an extension
   injecting it — still gets the tools.
2. **The Claude artifact runtime.** Published as an artifact, the page declares the
   `sample` capability and hands Claude these same tools through `claude.use("sample")`.
   **Ask Claude to drive** in the Agent tools window is that channel: type an instruction,
   and Claude calls the tools from inside the page while you watch the log. This is what
   works on claude.ai, where `navigator.modelContext` does not exist.
3. **`window.maze95`** — always present. `window.maze95.call(name, args)` from a console,
   an extension or a driver script, or post `{type:"maze95:call", tool, args}` to the frame
   and read the `maze95:result` reply.

Every call, whichever channel it came from, is echoed in the on-screen log.

**If an agent can't see the tools**, the Agent tools window answers why: whether
`navigator.modelContext` is present and which registration API it offers, how many tools
registered, whether the page is in a secure context, whether it is framed, and whether the
Claude runtime is there. `window.maze95.diagnostics()` returns the same as JSON, and the
page logs it to the console on load. `window.maze95.register()` retries registration by
hand.

| Tool | Does |
| --- | --- |
| `maze_get_state` | Seed, grid size, current cell, heading, which sides are open, steps to the exit |
| `maze_look` | Corridor length ahead, side openings along it, whether the exit is in view |
| `maze_get_map` | ASCII floor plan — `@` player, `S` start, `X` exit, `?` unexplored |
| `maze_move` | Walk whole cells relative to the heading; stops at walls and says so |
| `maze_turn` | Quarter turns, or an absolute compass heading |
| `maze_goto` | Shortest route to any cell |
| `maze_solve` | Shortest route to the exit |
| `maze_new_maze` | Regenerate, optionally with a size or a seed |
| `maze_set_options` | Wall, floor and ceiling styles, lighting, light range, FOV, wall height, speed, loops, overlays |
| `maze_set_driver` | Hand the camera to the screensaver, the agent, or the player |

Because `maze_look` and `maze_get_map` return the layout as text, an agent can solve the
maze without ever looking at a pixel.

Four things make agent control predictable:

- **Nothing moves unless asked.** No autoplay, and the screensaver never starts on its own,
  so state between two tool calls only changes if a tool changed it.
- **Movement tools resolve when the walk lands.** Long routes animate proportionally faster
  so a call never takes more than a few seconds, and `instant: true` skips the animation.
- **The maze never regenerates under an agent**, even with "New maze at exit" on. It reports
  `solved: true` and waits for `maze_new_maze`.
- **Arguments are normalised.** Whether the host passes the arguments directly, wraps them
  in `arguments`/`input`/`params`, or hands over JSON text, the tool sees the same object.

## Keys

| | |
| --- | --- |
| `W A S D` / arrows | Move (arrows left/right turn) |
| Mouse | Look, after clicking to lock the pointer |
| `Shift` | Run |
| `Tab` | Overhead map |
| `N` | New maze |
| `Esc` | Back to the screensaver |
