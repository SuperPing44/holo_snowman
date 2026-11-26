# Rendering and warp: which stack draws Strokes and Stickers and warps each Surface?

Research for issue #3 (map: #1). Researched 2026-11-26 against library docs, MDN, the CSS Transforms 2 spec and the npm registry.

## Answer

**Drawing window: Konva + perfect-freehand, with an own command-based undo history. Projector window: one small raw WebGL2 renderer (no library) that samples the Design layer's canvas as a single texture and draws five clipped quads: an affine-mapped, ellipse-masked quad per Ball and a homography-mapped quad per Panel.**

CSS `matrix3d` on per-Panel canvases is a workable no-shader fallback for the projector side. PixiJS, three.js and Fabric.js are not needed.

## What the problem needs

| Need | Where it bites |
|---|---|
| Freehand Strokes with pen pressure, ~8 colours, 3 sizes, eraser | Drawing window |
| Stickers: place, move, resize, rotate with handles | Drawing window |
| Undo, clear all, autosave, export as image | Drawing window (document model) |
| Ball: template ellipse -> calibrated projector ellipse | Projector: an **affine** map (centre, two axes, rotation) plus an elliptical clip. Canvas 2D can already do this. |
| Panel: template rectangle -> four calibrated corners | Projector: a **projective** map (homography). Canvas 2D `setTransform` is affine only and cannot do it. |
| Live at 1080p, black background, clipped | Projector: re-render on every Design change |

So only the three Panels truly need perspective. Everything else is ordinary 2D.

## Options compared

### Drawing side

| Option | Pressure strokes | Sticker handles | Undo | Size / complexity | Notes |
|---|---|---|---|---|---|
| **Konva 10.x** | Via perfect-freehand outline -> `Konva.Path` (SVG path data) or a custom `Konva.Shape` | **Built in**: `Konva.Transformer`, `tr.nodes([...])`, resize by `scaleX/scaleY`, rotate handle, `boundBoxFunc` for limits | Not built in; docs recommend keeping your own history of state changes, not Konva serialization | npm unpacked ~2 MB, no deps, small API | Each `Konva.Layer` is its own `<canvas>`, so the Design layer can be the projector texture. Eraser via `globalCompositeOperation: 'destination-out'` (official free-drawing demo). |
| Fabric.js 7.x | Has `PencilBrush`; pressure is not documented on the intro page | Built in object controls | Not built in | npm unpacked ~22 MB, bigger object model | Fine too, but heavier; offers nothing Konva lacks here. |
| PixiJS 8.x | Must build yourself (Graphics) | None; build handles yourself | None | npm unpacked ~80 MB (tree-shakable), WebGL/WebGPU renderer | Would unify both windows in one library, but you hand-write sticker handles and hit-testing. |
| Plain Canvas 2D + own code | perfect-freehand + `Path2D` | Hand-written (hit-testing, rotate/scale handles: several days) | Own | Zero deps | Most control, most work. |
| **perfect-freehand 1.2.x** | `getStroke(points, { simulatePressure: false })` takes `[x, y, pressure]` and returns a closed outline polygon; fill it via `Path2D`/SVG path | – | – | ~110 KB unpacked, MIT | The standard for pressure-sensitive variable-width strokes. Pointer Events give `pressure` in 0..1 and `pointerType === "pen"`; Wacom eraser end shows as `buttons` 32. `getCoalescedEvents()` gives extra points for smoother lines (not Baseline, but works in Chromium). |

### Projector side (warp)

| Option | Perspective warp | Clipping | Performance at 1080p | Complexity |
|---|---|---|---|---|
| **Raw WebGL2, one shader** | Exact per pixel: fragment shader applies the inverse homography to get template UVs, no mesh subdivision | Ellipse test in shader (`discard`, `smoothstep` edge for anti-aliasing); quads clip themselves | One `texImage2D` of the Design canvas per change (HTMLCanvasElement is an accepted source) plus 5 quads: trivial for any GPU | ~150-250 lines incl. 4-point homography solver; no dependency |
| regl 2.1.1 | Same shader approach, nicer API | Same | Same | Last published Nov 2024, WebGL1 API; saves little over raw code for 5 quads |
| PixiJS 8 `PerspectiveMesh` | Yes: "a 2D mesh with a perspective projection applied", corners `x0..y3` clockwise from top-left, `setCorners(...)`, `verticesX/Y` grid (default 10x10) | Masks for ellipses | Good | Pulls in a whole engine for 5 quads; second library next to Konva |
| three.js | Possible (plane geometry + custom projection or shader) | Shader / stencil | Good | 3D scene graph and camera setup for a 2D job: overkill |
| **CSS `matrix3d` on per-Panel canvases** (fallback) | Yes: 4x4 homogeneous matrix, column-major; spec applies w-division so a homography embeds directly | Panel canvas holds only its own region, so it clips itself; `clip-path`/`overflow` on a 3D-transformed element force flattening, which is fine here | Browser compositor, GPU; very cheap | No shaders. Balls drawn on one Canvas 2D with `setTransform` (affine) + `ellipse()` clip + `drawImage` |

**Homography -> matrix3d.** Solve the 3x3 `H` mapping the four template corners to the four calibrated corners (8x8 linear system). Embed it as

```
| h11 h12 0 h13 |
| h21 h22 0 h23 |
|  0   0  1  0  |
| h31 h32 0 h33 |
```

and pass it to `matrix3d(...)` in **column-major** order, with `transform-origin: 0 0` and the canvas positioned at the template-space origin. The same `H` (inverted) is the uniform for the WebGL shader, so the maths is shared by both projector options.

## How drawing and projector share one scene

- The **Design model** is the single source of truth: `Stroke { points:[x,y,p][], colour, size, eraser }` and `Sticker { id, x, y, scale, rotation }` in template coordinates. Undo/redo is a command stack on this model (add Stroke, add/move/transform/delete Sticker, clear). Konva nodes are a view of it, not the store (as the Konva docs advise). Autosave and the "Done" export serialise the same model.
- Konva renders the Design on its own **Design layer** (no template outline, transparent background). That layer's `<canvas>` (`layer.getNativeCanvasElement()`) is the projector's single texture. The template outline sits on a separate layer and is never projected.
- The projector window is opened with `window.open` from the same origin. The plan: the main window keeps the single render loop and creates/drives the WebGL canvas inside the popup's document, uploading the Design canvas whenever it changes. **This cross-document `texImage2D` path is not confirmed by any primary source found; prove it in a day-one spike.** Fallbacks, in order: (a) `designCanvas.captureStream()` -> `<video>` in the projector window -> video texture (Baseline since 2020); (b) send the Design model over `BroadcastChannel` and re-render it with the same Konva code in the projector window.
- Template resolution: render the Design layer at least at the projector's on-Surface pixel density (set `pixelRatio` on the layer), so stretched Surfaces stay sharp.

## Why this stack

- Konva covers the two time sinks of the drawing side out of the box: sticker transform handles (`Transformer`) and layered canvases. perfect-freehand covers pressure strokes. What remains (undo) has to be custom in every option anyway.
- The warp is only five quads. A shader you wrote yourself is the smallest, most exact and most debuggable form of it, and it adds no second scene graph. The homography maths is the same as for the CSS fallback, so switching costs little.
- Learning curve for one developer: Konva's API is small; the WebGL part is a single fixed pipeline (one texture, one shader, five draw calls), learnable in a day or two.

## Main risk

**Getting the Design pixels into the projector window every frame.** Cross-window texture upload is unverified. Run a spike in week 1: popup window, Konva layer canvas -> WebGL texture, one homography quad, measure frame time at 1920x1080 while drawing. If it fails, use the `captureStream()` or `BroadcastChannel` fallback above. Secondary risks: the eraser's interaction via `destination-out` across Stickers (decide whether the eraser erases Stickers or only Strokes), and blurry Panels if the Design layer resolution is too low.

## Sources

- PixiJS mesh guide, `PerspectiveMesh`: https://pixijs.com/8.x/guides/components/scene-objects/mesh and https://pixijs.download/release/docs/scene.PerspectiveMesh.html
- MDN `matrix3d()`: https://developer.mozilla.org/en-US/docs/Web/CSS/transform-function/matrix3d
- CSS Transforms Module Level 2 (w-division, flattening by `clip-path`/`overflow`): https://www.w3.org/TR/css-transforms-2/
- Konva Transformer: https://konvajs.org/docs/select_and_transform/Basic_demo.html
- Konva undo/redo guidance: https://konvajs.org/docs/react/Undo-Redo.html
- Konva free drawing (eraser via `destination-out`): https://konvajs.org/docs/sandbox/Free_Drawing.html
- Konva performance tips: https://konvajs.org/docs/performance/All_Performance_Tips.html
- Konva custom shapes: https://konvajs.org/docs/shapes/Custom.html
- perfect-freehand README: https://github.com/steveruizok/perfect-freehand
- MDN Pointer Events: https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events and https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/getCoalescedEvents
- MDN `texImage2D` (accepted sources): https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D
- MDN `captureStream()`: https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/captureStream
- WebGL 1.0 spec (origin-clean rules, using a context as a source): https://registry.khronos.org/webgl/specs/latest/1.0/
- regl repository: https://github.com/regl-project/regl
- Versions and unpacked sizes: npm registry (`npm view <pkg> version dist.unpackedSize`) on 2026-11-26: konva 10.11.0, fabric 7.4.0, pixi.js 8.23.0, three 0.187.0, regl 2.1.1 (last published 2024-11), perfect-freehand 1.2.3.
