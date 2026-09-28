# Bloub → QML port

Everything needed to bring the x.ai blob avatar into Quickshell. Source of truth:
[`jeremy-prt/bloub`](https://github.com/jeremy-prt/bloub) (MIT, Vue 3 + Vite + TS +
Tailwind 4, created 2026-08-15). Live demo: <https://bloub.vercel.app> (that URL is the
repo's homepage, not a random deployment).

## 1. What bloub actually is

One filled shape (the body) that morphs between **14 states**, and two white shapes
(the eyes) that morph independently. Eyes are **holes in a mask**, not white shapes laid
on top. Body silhouettes come from **radial profiles r(θ) measured from the reference
video frame by frame** (64 samples per profile), so any two shapes have point-to-point
correspondence and a morph is linear interpolation of radii. No path-morphing library,
no animation library.

Key measured facts (from upstream `docs/measurements.md`):

| Assumption | Reality |
|---|---|
| Eyes lean `//` | They lean `\\`, ~26° off vertical |
| Body is a squircle | Perfect circle, radial deviation < 0.7 % |
| Transitions are springs | Exponential ease-outs; the body never overshoots |
| The comet crosses the screen | The dot stays put, the trail orbits it |
| Avatar floats at rest | It doesn't — life comes from gaze drift and blinking |

## 2. Upstream repo map (`src/bot/`, byte sizes)

| File | Size | Role | Port? |
|---|---|---|---|
| `engine.ts` | 22 227 | `BotEngine`: state machine, pose blending, morph history, look/gaze | **yes** |
| `states.ts` | 17 031 | 14 `StateDef`s + `POSES` durations + `SEQUENCE` | **yes** |
| `shape.ts` | 10 222 | radial-profile sampling, path building, `radiusAtAngle` | **yes** |
| `decor.ts` | 8 652 | dots/arcs render items (comet trail, burst particles) | **yes** |
| `expressions.ts` | 5 539 | blink/expression blending | **yes** |
| `face.ts` | 5 832 | eye poses, blink scale, liveliness/gaze drift | **yes** |
| `profiles.ts` | 2 077 | measured profiles, `PROFILE_SAMPLES = 64` | **yes** |
| `math.ts` | 1 566 | `clamp`, `lerp`, `r2`, `TAU`, easings | **yes** |
| `repere.ts` | 1 317 | shared reference frame helpers | **yes** (if imported transitively) |
| `eyefit.ts` | 18 589 | eye-offset solver table for *custom* body shapes | **stub → 0** |
| `skins.ts` | 4 785 | customiser shapes (analytic) | skip |
| `cycles.ts` | 8 924 | montage playback (stretch/cut blocks) | skip |
| `*.test.ts` | ~60 k | upstream tests | skip (port the intent) |

Dependency direction: `engine → {decor, expressions, eyefit, face, math, shape, states}`,
`shape → {math, profiles}`, `states → {face, math, shape}`.

**Do not port** the Vue app (`App.vue`, `components/`, `ui/`) — only `src/bot/` matters.
`BloubBot.vue` is still worth reading for how the SVG output is consumed (mask + paper
backing), see § 4.

Setup for reference runs:

```bash
git clone https://github.com/jeremy-prt/bloub && cd bloub
pnpm install && pnpm dev      # http://localhost:5190
pnpm test                     # upstream engine tests
```

Useful URLs: `#planche` (all 14 states side by side, frozen) and
`#etat=<id>&stop` (single state, paused). Both are the validation harness for the port.

## 3. Engine API surface

```ts
new BotEngine(R, state, shapeRadii, expression)   // R = radius in user units
engine.sample(now): BotFrame                       // now = seconds, monotonic
engine.setState(id, now)
engine.reset(state, now)
engine.setShape(radii, now)
engine.setExpression(expr, now)
engine.setLook(look | null, now, morph)            // pointer/gaze target
engine.state
```

```ts
interface BotFrame {
  bodyPath: string            // SVG path data, body silhouette
  bodyAlpha: number
  eyes: { d: string; matrix: string; alpha: number }[]
  dots: DotRender[]
  dotsBehind: boolean         // particles render behind the body
  arcs: ArcRender[]
  notif: { x: number; y: number; r: number } | null
  notch: { x: number; y: number; r: number } | null
}
```

`StateId` (14 in `SEQUENCE` + `swirl`, a UI transition state):

```
idle thinking wink wide alert notify exclaim sleep egg hexagon play orbit burst comet  (+ swirl)
```

Hold durations (`POSES`, seconds): idle 1, thinking 1.1, wink .8, wide .8, alert .75,
notify .9, exclaim .8, sleep .45, egg .8, hexagon .8, play .9, orbit 1.2, swirl .5,
burst .45, comet 1.15. Each `StateDef` also carries `morph` (entry fade, longest is
`orbit` at 0.6 s), `blinkIn`, `minDuration`, `baseBody`, `baseFace`.

### Invariants you must preserve

1. **`sample(t)` is a pure function of time.** No `Date.now()`, no clock, no mutation.
   `setState` keeps *one* slot of history; a state change inside a fade freezes the
   composite pose and blends from it. Purging the previous state during playback breaks
   replayability (upstream has a dedicated test for this).
2. **Profiles are sampled at the same 64 angles.** Never change `PROFILE_SAMPLES` in one
   place only, and route new shapes through radial profiles (or `profileFromPolygon`).
3. **Eyes live on a sphere of radius 1** and must be pro-rata placed with
   `radiusAtAngle` so they stay on the silhouette for non-circular shapes.
4. **Eyes are holes.** See § 4.
5. `eyefit` solves eye offsets *once at import* for custom shapes; solving it per frame
   caused real artefacts (active-set changes, chattering). Since bloub-island does not
   support custom body shapes in v1, replace `decalageDesYeux` with a constant zero and
   delete the table. Keep the function shape so the port stays diff-able against upstream.

## 4. Rendering in QML

Upstream DOM order (`BloubBot.vue`) is, bottom → top:

1. arcs/dots that belong *behind* the body (back half of rings, burst particles)
2. an opaque **paper-coloured body path** (the "backing") — exists so that the eye holes
   show paper instead of the rings behind, and so rings are occluded by the ball
3. a masked group: body filled in body colour, then front dots/arcs, notif pastille
   - mask = body path in white **minus eye paths** (holes)

### Option A — `Shape` + `PathSvg` with odd-even fill (target)

Three stacked `Shape` items mirror the three DOM layers:

```qml
Item {
    id: avatar
    property var frame // BotFrame
    property color bodyColor: Appearance.colors.colOnLayer0
    property color paperColor: Appearance.colors.colLayer0

    Shape { // layer 2: paper backing (also receives back dots below it)
        anchors.fill: parent
        ShapePath { fillColor: avatar.paperColor; PathSvg { path: avatar.frame.bodyPath } }
    }
    Shape { // layer 3: body with eye holes
        anchors.fill: parent
        ShapePath {
            fillColor: avatar.bodyColor
            fillRule: ShapePath.OddEvenFill
            PathSvg { path: avatar.frame.bodyPath }
            PathSvg { path: /* eye[0].d with matrix baked */ "" }
            PathSvg { path: /* eye[1].d with matrix baked */ "" }
        }
    }
}
```

- `RenderedEye.matrix` is an SVG transform (`matrix(a,b,c,d,e,f)`). `ShapePath` has no
  transform, so **bake the matrix into the path data in JS** before handing it to QML.
  (Same 2-D affine math the upstream DOM applies via the SVG `transform` attribute.)
- Eye holes reveal the paper layer → the classic bloub look, theme-safe because
  `paperColor` is the bar surface colour.
- Shortcut for v1: skip layer 2, fill eye paths with `paperColor` in layer 3 instead of
  making holes. Only differs when a *behind* dot/arc crosses the eye region. Acceptable.

### Option B — `Canvas` (fastest to prototype)

```qml
Canvas {
    onPaint: {
        const ctx = getContext("2d");
        ctx.reset();
        ctx.setTransform(s, 0, 0, s, cx, cy);      // s = pixel radius, origin at centre
        ctx.fillStyle = bodyColor;
        ctx.fill(parsePath(frame.bodyPath));
        ctx.globalCompositeOperation = "destination-out";
        for (const e of frame.eyes) {
            ctx.save(); ctx.transform(...parseMatrix(e.matrix));
            ctx.fill(parsePath(e.d)); ctx.restore();
        }
    }
}
```

- Simplest because `ctx.setTransform` handles the eye matrices natively.
- Needs a tiny SVG path parser (`M/L/C/Z`, maybe `Q`) because `Path2D`/`fill(path)` support
  in Qt's Canvas must be verified on this machine — **verify first**, otherwise parse
  manually.
- Raster, but at 32–80 px this is irrelevant.
- Layer nuance: punching holes reveals the pill background (which *is* paper in the bar),
  so Option A's layer 2 is unnecessary here.

### Option C — GIF fallback (demo only)

Export each state as a GIF from the web app (`#planche` → Export) and `AnimatedImage` it.
20-minute path to something pretty; not state-addressable, no gaze, banding at 32 px.
Keep as a fallback if the port stalls, not as the plan.

### Drive loop

```qml
Timer {
    interval: 1000 / 30          // 30 fps is plenty at pill size
    repeat: true
    running: avatar.visible
    onTriggered: { frame = engine.sample(clock / 1000); }
}
```

`clock` is a monotonic elapsed counter in ms — never `Date.now()` deltas; upstream numbers
are seconds.

## 5. Theming

- Body = `Appearance.colors.colOnLayer0` (light in dark theme, dark in light theme).
- Eyes/paper = surface behind the widget: pill `colLayer0`, or `colPrimary` for an accent
  mascot. Never hardcode black/white.
- Optional: `colError` tint for the `exclaim`/error phase badge (not the mascot body).

## 6. Size guidance

- Pill height is 32 px → mascot ≈ 22–24 px, leaving room for a label.
- At that size, `comet`/`orbit`/`notch` detail is lost. Use this **pill subset**:
  `idle`, `thinking`, `play`, `alert`, `burst`, `sleep`, `exclaim`.
- Hover-expanded view (Phase 3) can render at 64–96 px where the full set is legible.

## 7. Validation

1. Run upstream, open `#planche`, screenshot the frozen states.
2. In the QML preview harness, freeze the port at the same state and timestamp
   (mirror of `frozenAt`; do this by calling `engine.reset(id, t0)` then `sample(t0 + t)`),
   screenshot, diff.
3. Port `engine.test.ts` intent for the critical invariants: replayability, no overshoot,
   fade continuity (`idle → wide → idle` at 100 ms), pure `sample`.
4. Accept only antialiasing differences; geometry must match.

## 8. License

Upstream bloub is MIT (`LICENSE` in that repo). The port is a derivative work:

- Keep the MIT license text in `modules/common/bloub/LICENSE`.
- Add a header to every ported file: *"Ported from jeremy-prt/bloub (MIT).
  https://github.com/jeremy-prt/bloub — original © Jeremy Prt."*
- Mention bloub in the end4-pC README/about screen.
- Do not relicense the port under anything stricter than MIT.
