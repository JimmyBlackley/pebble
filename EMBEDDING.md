# Embedding pebble in a host application

pebble was built to own a window. This document is about the other case: pebble
as one component inside somebody else's web application, mounted into an element
it does not control, alongside a UI framework that will unmount and remount it
without warning.

It exists because that case drove a specific set of changes on `feat/embed`, and
because those changes now have a consumer — the Medsplat viewer, a Gaussian-splat
renderer shipped as an npm package and embedded in a React app. If the fork's
lineages are ever reconciled (see the last section, which is the real work), this
is the list of behaviour that must survive.

---

## What an embedded app needs, and why

### 1. Render into a canvas the host already owns

winit has supported this since 0.30 via `WindowAttributesExtWebSys::with_canvas`;
pebble simply never plumbed it, so the only option was `with_append(true)` —
winit creates its own canvas and appends it to `<body>`.

That is fine for a page that *is* the app and useless for one embedding it: the
canvas escapes the host's element tree and ignores its layout. The consumer's
workaround before this existed was a 100 ms `setInterval` that scraped the DOM
for the canvas and force-styled it `position: fixed; 100vw × 100dvh`.

```rust
WindowConfig { .. }.with_canvas(canvas)   // web only
```

**The non-obvious half:** when a host canvas is supplied, the requested inner
size must *not* be applied. winit writes it onto the element as inline pixel
styles, and inline styles beat the host's stylesheet — measured, a 640×380
container held a canvas that stayed stubbornly 1280×720 and overflowed it. With
the size left alone, `web_canvas_size` already measures the canvas every frame,
so CSS-driven resizes are picked up with no resize handling in the host at all.

`prevent_default` and `focusable` are exposed through the same config. winit
defaults both to `true`; an embedded app often wants `prevent_default` off so the
host page keeps its own scrolling and focus behaviour, and `focusable` must stay
on because keyboard events on the web are canvas-scoped.

### 2. Stop drawing when nothing is changing

A host application is not a game. A medical scan sits still while someone looks
at it, and re-presenting an identical frame at the display's rate is pure cost —
on a phone, it is battery spent on nothing.

```rust
WindowConfig { redraw_policy: RedrawPolicy::OnDemand, .. }
```

`Continuous` remains the default, so nothing that exists today changes.

Measured in the consumer: its own idle gate was already skipping the expensive
passes correctly — scene passes ran on **0 of every 120 ticks** — while the app
went on ticking at **~52/s**. The pipeline was never the idle cost. The frame
was: a full ECS tick, a blit, an overlay pass and a present, forever.

Three things about `OnDemand` are easy to get wrong, and all three were:

- **`Redraw::request` must wake a sleeping loop, not just set a flag.** Setting a
  flag is enough from inside a frame, where the end-of-frame check is still to
  come. A request arriving while the loop is asleep has nothing to read it — and
  waking from a DOM callback is the entire reason a caller is handed a `Redraw`.
  It therefore holds the window and calls `request_redraw`.
- **Buffered input events must wake the loop.** They are only drained by a
  redraw, so an arriving event has to ask for the frame that will consume it.
- **Readiness must be sampled *before* the tick.** The backend is acquired
  asynchronously, and until it is ready `App::update` runs only the internal GPU
  schedules and returns — so no system of the app's has run. Testing readiness
  afterwards sees `true` on the exact frame the backend arrives, which is the one
  frame where still nothing has run, and the loop sleeps one tick short of doing
  any work at all. The symptom is brutal to diagnose: a viewer that comes up with
  a live GPU device, a correctly sized canvas, and a blank image. `App::backend_ready`
  exists for this.

**The rule this implies for the whole engine:** under `OnDemand`, anything that
*polls* breaks, and breaks silently. The consumer had a scene picker that read a
`window.medsplatPending` global once per tick; the moment the loop could sleep,
clicking it did nothing and reported nothing. Polled state has to become pushed
calls — which is what `AppHandle` is for.

### 3. Be told things from outside the ECS

`AppHandle` (from `feat/stream-and-app-handle`) is the supported channel: get one
with `App::handle()` before `run()` consumes the app, then send events from a
browser callback. Events land at the start of the next tick in the same
double-buffered queues an `EventWriter` feeds.

Anything using it under `OnDemand` must also request a redraw, or the event lands
in a queue nothing will come back to drain.

### 4. Go away again, and come back

This is the requirement that is easiest to overlook and hardest to retrofit.

React unmounts and remounts constantly — StrictMode does it on every mount in
development. winit allows **one event loop per process** and hands it back only
once the old one is destroyed, which happens a frame or two after exit.

Two changes make this survivable:

- `EventLoop::new()` moved from `WindowPlugin::build` into the runner. Building it
  while merely *assembling* an App made construction the scarce operation, so a
  host that built an App it then chose not to run would poison the next attempt.
- **Acquiring the GPU is fallible, not a panic.** `request_adapter`/`request_device`
  were unwrapped, so a browser without WebGPU got a wasm trap rather than an
  error — unhandleable by the host, with nothing to show a user, and with the
  whole module dead afterwards. `init_gpu` returns `Result` and drops the
  fulfiller on failure, a path `poll_gpu` already treated as "no usable backend".

`panic = "abort"` makes this sharper than it looks: a panic does not fail an
operation, it ends the instance. An embedded app cannot recover from one, so
anything reachable from a host API call has to be an error instead.

### 5. Not drag in what it cannot use

`image` was a required dependency used in exactly one place — `decode_file`,
which decodes a texture from a filesystem path. A wasm app has no filesystem, and
an app that builds its textures from buffers never calls it anywhere. It is now
the `image-textures` feature, on by default so existing consumers notice nothing.

Most of it was already being dead-stripped by LTO; the real gain is that the
published package's TypeScript stopped mentioning JPEG chroma subsampling.

### 6. Hand over the canvas element

`Window::canvas()` returns the element winit is actually using, so a host can
attach its own DOM listeners without a `querySelector` guess. Needed for:

- a `ResizeObserver`, if per-frame canvas measurement is ever removed;
- a hidden `<input>` for text entry, which is the one thing in-canvas UI cannot
  do — winit gives key codes, not characters, and a touch device needs a real
  input element to raise its keyboard;
- pointer events winit does not surface. Stylus pressure and tilt are the live
  example: winit reads `pressure` and then discards it on the pen path, tilt has
  no representation in winit 0.30 at all, and `setPointerCapture` is called for
  `"mouse"` only, so a pen stroke leaving the canvas drops mid-draw.

---

## Constraints an embedded app has to live with

Not bugs; things to design around.

| Constraint | Consequence |
|---|---|
| **WebGPU only.** No WebGL2 feature, and the consumer's pipeline is compute-shader based anyway. | Hard browser requirement. Detect `navigator.gpu` and refuse with a message rather than mounting something that will never draw. |
| **One event loop per process.** | One pebble app per wasm module. Teardown is asynchronous, so a host that remounts must serialise against it. |
| **No text, no 2D drawing, no debug draw, no UI.** Deliberate — pebble gives primitives. | The consumer's transfer-function editor is ~1100 lines of hand-rolled WGSL because there was no other option. |
| **`build_material` hardcodes `TriangleList`,** and `PolygonMode::Line` needs a native-only feature. | No line-list pipeline on the web. Draw lines as SDF-shaded quads. |
| **Touch is mapped onto the primary mouse button** (`feat/timestamp-queries` only), discarding `touch.force` and `touch.id`. | Single-finger drag works everywhere for free; multi-touch and pinch do not exist, and pen/palm cannot be told apart. |

---

## Verifying it still works

The behaviours above are not unit-testable — they are about what a browser does
with a real module. The consumer's `web/embed-test.html` is the harness, driven
headlessly by Playwright, and it checks:

- the module instantiates and renders **nothing** until asked;
- **zero** stray canvases appended to `<body>`;
- the canvas obeys its container, including across a CSS resize (640×380 → 860×460,
  backing store included);
- a second mount is refused cleanly rather than trapping;
- destroy → remount works, and the remounted viewer loads and renders.

Anything that changes windowing, the run loop or GPU acquisition should be run
against it. A regression here looks like success from inside Rust.

---

## The actual blocker: two lineages

`fork/main` and `feat/embed` have diverged by **102 and 242 commits** from a
common base (`0b6f4fc`). A trial merge produces **30 conflicts, 20 of them in
`src/`**.

This is not a textual merge. The two lineages reorganised the tree differently —
what is `src/rendering/*` on `fork/main` is `src/graphics/*` on `feat/embed`, so
those appear as modify/delete pairs — and `fork/main` carries `feat!:` breaking
changes to the material and bind-group API, plus its own work that *overlaps what
`feat/embed` solves differently*:

- `fix: pace web frame loop to requestAnimationFrame instead of unthrottled Poll`
- `fix: sync canvas/window size to browser viewport on resize`
- `feat: add GPU->CPU readback helpers and clean up stale code`
- `feat!: explicit bind group and binding indices in materials/compute`

Each of those has a counterpart on `feat/embed`. Reconciliation means choosing a
winner per subsystem and knowing why, not resolving conflict markers.

**Nothing is blocked by this today.** The consumer pins rev `f4a836a` on
`feat/embed`, which builds from GitHub and is what its production deployment
runs. This is consolidation, and it should be done deliberately or not at all.

Whoever takes it on should know that the consumer is written against
`feat/embed`'s API and moves with whatever is chosen — so the two repositories
have to be stepped forward together, with the harness above as the gate.
