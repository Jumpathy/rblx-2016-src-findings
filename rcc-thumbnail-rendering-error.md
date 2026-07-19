# RCC thumbnail corruption — root cause & fix (Mesa VAO index-buffer bug)

**TL;DR:** If your source-built RCCService renders avatar/headshot thumbnails as garbled,
partial, or warped geometry (detached limb fragments, missing chunks of the head with a
jagged diagonal edge, face/clothing decals smeared or rotated across the body, or renders
that come out nearly empty) while simple brick-only scenes render perfectly — the cause is
a bug in the Mesa software OpenGL driver (`OPENGL32.dll`, "Mesa Windows GDI Driver",
observed on Mesa 7.8.1) that RCC uses for headless rendering: **Vertex Array Objects do
not remember their element (index) buffer binding.** One 10-line, driver-gated workaround
in the engine fixes it completely.

---

## Symptoms

- Avatar / headshot renders come out different-but-deterministically-wrong depending on
  how many renders the RCC instance has already done: the first render on a fresh process
  looks different from the second, and renders 2..N are identical to each other but still
  wrong. A pooled RCC (multiple jobs per instance) therefore produces a mix of differently
  broken thumbnails.
- Typical looks: head with its front triangles missing (you see into the shell, jagged
  diagonal cut); the face decal stretched into a curved band across the head; clothing
  textures present but warped/rotated into wrong places; character reduced to a few
  floating slivers; sometimes nearly-empty output.
- **Plain-part scenes (bricks, no character) render flawlessly every time** — this is the
  tell that misdirects you away from the driver. Character scenes break because they are
  the first thing that puts *many* index buffers in flight at once.
- Nothing errors. The full pipeline (asset loads, texture compositor, render, PNG encode)
  reports success.

## Root cause

The engine's GL backend wraps every geometry in a VAO when the driver advertises
`GL_ARB_vertex_array_object` (`Rendering/GfxCore/GL/GeometryGL.cpp` — the ctor records
vertex attribute pointers *and the index buffer binding* into the VAO, and
`GeometryGL::draw()` just binds the VAO and calls `glDrawElements`).

Per the GL spec, the `GL_ELEMENT_ARRAY_BUFFER` binding is part of VAO state. **Mesa
7.8.1's implementation doesn't honor that: an indexed draw through a VAO uses whichever
index buffer was most recently bound globally, not the one recorded in the VAO.**

Consequence: with several live geometries (character mesh cluster batches, texture-
compositor quads, sky, adorns — all created in varying order), almost every indexed draw
runs with *another geometry's index buffer*. Triangles are assembled from wrong vertex
indices → detached/warped/missing geometry. Which index buffer happens to be bound when
each draw executes depends on the creation/bind history, which is why:

- render #1 on a fresh process ≠ render #2, but #2 == #3 == … (steady state), and
- a bricks-only scene is fine (essentially one geometry in play — the "wrong" index
  buffer is the right one).

This is the **same driver-bug class the engine already works around for Intel**:
`createDeviceCapsOld()` in `Rendering/GfxCore/GL/DeviceGL.cpp` has carried this for years:

```cpp
if (strstr(vendorString, "Intel"))
{
    // Intel drivers don't store IB state as part of VAO (http://stackoverflow.com/questions/8973690/vao-and-element-array-buffer-state)
    caps.extVertexArrayObject = false;
}
```

Mesa just needed the same treatment.

## Proof (so you don't have to take this on faith)

A standalone Win32 probe (link against `opengl32.lib`, run it from the RCC directory so
it binds the bundled Mesa `OPENGL32.dll`) shows the driver is healthy everywhere the
engine needs it — **except** VAO index-buffer state:

| Test | Result on Mesa 7.8.1 |
|---|---|
| Skinned-VS pattern: `WorldMatrixArray[int(attr) * 3]` dynamic uniform-array indexing | OK |
| `glUniform4fv` uploading a 216-element vec4 bone palette | OK |
| Non-normalized `GL_UNSIGNED_BYTE` vertex attributes (bone indices) | OK |
| `glMapBufferRange` / `glMapBuffer` upload paths | OK |
| **VAO recorded with index buffer A; buffer B bound while VAO 0 current; re-bind VAO and draw** | **BROKEN — draw uses B, must use A** |

Minimal repro of the broken case:

```c
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER, vbo);              // attribs...
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibGood);   // valid indices — recorded in VAO
glBindVertexArray(0);

glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ibJunk);   // junk indices, bound with VAO 0 current

glBindVertexArray(vao);
glDrawElements(GL_TRIANGLES, n, GL_UNSIGNED_SHORT, 0);
// spec: draws with ibGood.  Mesa 7.8.1: draws with ibJunk.
```

Two practical corollaries:

- **Recompiling Mesa 7.8.1 from official sources will not help** — it's a genuine
  implementation bug of that era (VAO support was brand new in 2010), not a bad build.
- Shader/GLSL theories (miscompiled skinning, uniform limits, etc.) are dead ends here;
  they all test clean.

## The fix

**File:** `Rendering/GfxCore/GL/DeviceGL.cpp`
**Function:** `createDeviceCapsOld()` (the caps path used when `FFlag::GraphicsGL3` is
false, which is the default), inside the existing `#ifdef _WIN32` block, immediately
after the Intel workaround (~line 201 in the unmodified file):

```cpp
    const char* versionString = reinterpret_cast<const char*>(glGetString(GL_VERSION));
    const char* rendererString = reinterpret_cast<const char*>(glGetString(GL_RENDERER));

    if ((versionString && strstr(versionString, "Mesa")) || (rendererString && strstr(rendererString, "Mesa")))
    {
        // Mesa's software GL (bundled with RCC for headless thumbnail rendering) has the
        // same defect as the Intel drivers above: the element-array-buffer binding is not
        // tracked per-VAO, so indexed draws through a VAO use whichever index buffer was
        // bound globally last instead of the VAO's own (verified on Mesa 7.8.1 "Mesa
        // Windows GDI Driver" with a standalone probe: a VAO recorded with index buffer A
        // drew using index buffer B after B was bound while VAO 0 was current). With many
        // live geometries this scrambles most indexed draws — avatar thumbnails rendered
        // as detached/warped fragments while single-geometry scenes looked fine.
        caps.extVertexArrayObject = false;
    }
```

With `caps.extVertexArrayObject == false`, `GeometryGL` skips VAO creation and instead
calls `bindArrays()` on every draw, which explicitly binds the vertex attribute pointers
*and the index buffer* each time — immune to the driver bug. The cost (a few extra
binds per draw call) is irrelevant on a software rasterizer.

**Blast radius:** none for Studio/WindowsClient — on Windows they render through D3D;
this code only runs for the GL device, and the check only matches drivers whose
`GL_VERSION`/`GL_RENDERER` string contains "Mesa". Mac GL (Apple drivers) never matches.
The `FFlag::GraphicsGL3 == true` caps path needs no change: it only enables VAOs on
GL 3.x contexts, and this Mesa is GL 2.1.

**Rebuild:** only `GfxCore` recompiles + relink of `RCCService.exe` (and anything else
you ship that can enter GL mode, if you want them consistent).

## How to verify after building

Render the same avatar **at least three times against the same RCC instance** (the pool
reuses instances, so single-render tests hide the bug). Before the fix: render 1 partial,
renders 2+ near-empty — all "successful". After the fix: all renders correct and
identical. A bricks-only scene passing means nothing here — it passes either way.

## Notes / things that are *not* the cause (verified while chasing this)

- Not the thumbnail scripts or camera setup: with a `ThumbnailCamera` under