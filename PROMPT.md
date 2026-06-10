# The Prompt

> This is the one-shot prompt that specifies the build in this repository. It works
> because it pins down the six things a model otherwise guesses at: the *physics
> model*, the *interaction model*, the *rendering pipeline*, the *sensory inputs*,
> the *platform constraints*, and the *aesthetic target*. Copy everything below the
> line into a frontier coding model (Fable 5) to reproduce or remix this project.

---

Build a real-time, GPU-accelerated 2D fluid dynamics simulation as a **single
self-contained `index.html`** (no build step, no dependencies, no network requests)
that runs at 60fps on desktop browsers and iOS Safari.

## Physics — solve the incompressible Navier–Stokes equations on the GPU

Implement Jos Stam's "Stable Fluids" method entirely in fragment shaders with
ping-pong half-float framebuffers:

1. **Advection** — semi-Lagrangian backtrace for the velocity field; **MacCormack**
   for the dye field: advect forward, advect the result backward, add half the
   error (`forward + 0.5 × (source − backward)`), and clamp the result to the
   min/max of the four texels around the backtraced sample so it stays stable.
   This is the single biggest fidelity upgrade — filaments stay etched instead of
   blurring out. Provide a manual-bilinear fallback when linear filtering of
   half-float textures is unsupported (and drop back to plain advection there).
2. **Vorticity confinement** — compute the curl of the velocity field and inject a
   force along the curl gradient to restore the small swirling eddies that numerical
   diffusion destroys. Curl strength ≈ 30.
3. **Pressure projection** — compute divergence, solve the pressure Poisson equation
   with ~20 Jacobi iterations, then subtract the pressure gradient to make the
   velocity field divergence-free.
4. **Dissipation** — exponential decay on velocity (~0.25/s) and dye (~0.9/s) so the
   fluid slows and fades naturally instead of accumulating forever.
5. **Tilt gravity (iOS)** — read DeviceMotion (request permission from the first
   touch gesture, as iOS requires) and apply the gravity vector as a
   **density-weighted body force** — force ∝ dye brightness — so tilting the phone
   pours the dye while empty regions stay still. Rotate the device-frame vector by
   the screen orientation angle. (A uniform body force would be cancelled entirely
   by the pressure solve; weighting by density is what makes it pour.)

Simulate velocity at reduced resolution (~128–144px on the short axis) and dye at
high resolution (~1024px) — velocity fields are smooth and don't need the pixels;
dye is what you look at.

## Interaction — the pointer IS the force field

- On **pointer move** (mouse) and **touch move** (iOS, multi-touch — track every
  active finger independently by pointer id), inject:
  - a **velocity splat**: a Gaussian impulse centered at the pointer, with direction
    and magnitude taken from the pointer's frame-to-frame delta × a force scalar
    (~6000),
  - a **dye splat** of the current stroke color at the same point.
- **Velocity-reactive splats**: scale the splat radius (~0.5×–2×) and dye brightness
  (~0.5×–3×) with pointer speed, so fast flicks burst wide and bright while slow
  drags etch thin dim lines.
- **Double-tap / double-click**: emit a radial shockwave — a ring of ~14 splats
  bursting outward from the tap point, hue-graded around the ring.
- **Triple-tap or `c` key**: cycle color themes (see Rendering). Show the theme
  name briefly in a minimal toast overlay.
- Correct the splat's Gaussian for the canvas aspect ratio so splats are round.
- iOS specifics: `touch-action: none`, `preventDefault()` on touch events,
  `user-scalable=no` viewport, handle `devicePixelRatio` but cap the dye buffer so
  older iPhones stay at 60fps.

## Sound — the fluid hears the room

Behind an unobtrusive 🎙️ toggle button (and the `m` key) — never automatic, since
mic access needs a user gesture — open `getUserMedia` audio into an `AnalyserNode`
(fftSize 256). Each frame, average the low-frequency bins into a bass level with an
exponential moving average; when the instantaneous level jumps ~15% above the
average past a floor threshold (a beat), fire a splat whose force, radius, and
brightness scale with the bass level, rate-limited to ~5/sec. Music makes the
fluid dance.

## Rendering — make it cinematic, not clinical

- **Color themes**, cycled by triple-tap/`c`, each an HSV hue band + drift rate:
  - **Spectrum** (default): the full wheel.
  - **Nebula**: hue 0.72–0.95 — deep violet → electric purple → hot magenta.
  - **Ember**: hue 0.95–1.12 (wrapping) — crimson → orange → gold.
  - **Glacier**: hue 0.48–0.68 — teal → cyan → deep blue.
  Each stroke (each mouse-down, each finger) seeds its own hue at full saturation
  and **drifts slowly along the stroke** (~0.003 of the wheel per splat), so a long
  drag leaves a ribbon that melts between adjacent colors — iridescent, oil-slick
  gradients, never random confetti. Banded themes sweep as a triangle wave so the
  drift never jumps.
- **Background**: near-black indigo (#080011), never pure black — the fluid should
  feel lit from within, floating in dark space.
- **Bloom**: bright-pass threshold + iterative downsample/upsample Gaussian blur,
  added back at modest intensity so the hottest cores halo and glow.
- **Sunrays**: build a luminance mask from the dye, radial-blur it outward from the
  center (16 taps, decaying), blur the result, and multiply it over the scene —
  volumetric light shafts through the smoke.
- **ACES filmic tonemapping**: accumulate dye in linear HDR and tonemap at display
  time with the ACES fitted curve and an exposure control, so highlights roll off
  like film instead of clipping.
- **Soft shading**: derive a cheap normal from the dye field's gradient and apply a
  subtle diffuse term, giving the smoke its rolling, volumetric, silk-like depth.
- **Dithering**: add hash-based noise at display time to kill banding in smooth
  gradients (critical on OLED iPhones).

## Life — it should never look dead

- On load, fire 8–12 random splats so the screen opens mid-swirl, and show a brief
  hint toast ("Drag to stir · double-tap to burst · triple-tap for themes").
- When idle for a few seconds, emit occasional gentle ambient splats so the fluid
  keeps breathing until the user touches it.

## Engineering constraints

- WebGL2 preferred, WebGL1 + `OES_texture_half_float` fallback; probe
  `RGBA16F`/linear-filtering support and degrade gracefully (disable bloom,
  sunrays, shading, and MacCormack rather than break).
- All simulation state in half-float FBOs; ping-pong on every pass; never read back
  to the CPU.
- Pause the simulation when the tab is hidden; resize FBOs on orientation change.
- Zero console errors, zero external assets, works from `file://`.
