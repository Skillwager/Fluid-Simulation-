# The Prompt

> The prompt below is engineered to extract a maximum-quality result from a frontier
> coding model (Fable 5) in a single shot. It works because it specifies the *physics
> model*, the *rendering pipeline*, the *interaction model*, the *platform constraints*,
> and the *aesthetic target* — the five things a model will otherwise guess at.

---

Build a real-time, GPU-accelerated 2D fluid dynamics simulation as a **single
self-contained `index.html`** (no build step, no dependencies, no network requests)
that runs at 60fps on desktop browsers and iOS Safari.

## Physics — solve the incompressible Navier–Stokes equations on the GPU

Implement Jos Stam's "Stable Fluids" method entirely in fragment shaders with
ping-pong framebuffers:

1. **Advection** — semi-Lagrangian backtrace of both the velocity field and the dye
   field, with manual bilinear filtering as a fallback when linear filtering of
   half-float textures is unsupported.
2. **Vorticity confinement** — compute the curl of the velocity field and inject a
   force along the curl gradient to restore the small swirling eddies that numerical
   diffusion destroys. This is what makes fluid look *alive* instead of soupy.
   Curl strength ≈ 30.
3. **Pressure projection** — compute divergence, solve the pressure Poisson equation
   with ~20 Jacobi iterations, then subtract the pressure gradient to make the
   velocity field divergence-free.
4. **Dissipation** — exponential decay on velocity (~0.25/s) and dye (~0.9/s) so the
   fluid slows and fades naturally instead of accumulating forever.

Simulate velocity at reduced resolution (~128–144px on the short axis) and dye at
high resolution (~1024px) — velocity fields are smooth and don't need the pixels;
dye is what you look at.

## Interaction — the pointer IS the force field

- On **pointer move** (mouse) and **touch move** (iOS, multi-touch — track every
  active finger independently by pointer id), inject:
  - a **velocity splat**: a Gaussian impulse centered at the pointer, with direction
    and magnitude taken from the pointer's frame-to-frame delta × a force scalar
    (~6000), so fast flicks shear the fluid hard and slow drags gently fold it,
  - a **dye splat** of the current stroke color at the same point.
- Each new touch/click gets a fresh color sampled from the palette, so multi-finger
  gestures on iOS paint interleaving ribbons of different purples.
- Correct the splat's Gaussian for the canvas aspect ratio so splats are round, not
  stretched.
- iOS specifics: `touch-action: none`, `preventDefault()` on touch events,
  `user-scalable=no` viewport, no 300ms tap delay, handle `devicePixelRatio` but cap
  the dye buffer so older iPhones stay at 60fps.

## Rendering — make it cinematic, not clinical

- **Palette**: full-spectrum, but *organized*, never random-confetti. Each stroke
  (each mouse-down, each finger) seeds its own hue at full saturation, and that hue
  **drifts slowly along the stroke** (~0.003 of the color wheel per splat), so a
  single long drag leaves a ribbon that melts from cyan through green into gold —
  adjacent colors blend in the fluid into iridescent, oil-slick gradients instead
  of clashing. Multi-finger gestures interleave ribbons from different points on
  the wheel.
- **Background**: near-black indigo (#080011), never pure black — the fluid should
  feel lit from within, floating in dark space.
- **Bloom**: bright-pass threshold + iterative downsample/upsample Gaussian blur,
  added back at modest intensity so the hottest magenta cores halo and glow.
- **Soft shading**: derive a cheap normal from the dye field's gradient and apply a
  subtle diffuse term, giving the smoke its rolling, volumetric, silk-like depth.
- **Dithering**: add a tiny amount of noise at display time to kill banding in the
  smooth purple gradients (critical on OLED iPhones).

## Life — it should never look dead

- On load, fire 8–12 random splats so the screen opens mid-swirl.
- When idle for a few seconds, emit occasional gentle ambient splats so the fluid
  keeps breathing until the user touches it.

## Engineering constraints

- WebGL2 preferred, WebGL1 + `OES_texture_half_float` fallback; probe
  `RGBA16F`/linear-filtering support and degrade gracefully.
- All simulation state in half-float FBOs; ping-pong on every pass; never read back
  to the CPU.
- Pause the simulation when the tab is hidden; resize FBOs on orientation change.
- Zero console errors, zero external assets, works from `file://`.

---

## Enhancement levers — how to push this prompt further

Each line below is a self-contained clause you can append to the prompt to raise the
ceiling. They are ordered by visual impact per unit of complexity.

1. **MacCormack advection** — "Replace semi-Lagrangian advection with MacCormack
   (forward + backward advect, error-correct, clamp to neighborhood min/max)."
   Dramatically sharper filaments and curls; the single biggest fidelity upgrade.
2. **Sunrays / god-rays** — "Add a radial-blur light-shaft pass driven by the dye's
   luminance, composited under the bloom." Gives the smoke volumetric backlighting.
3. **Velocity-reactive splats** — "Scale splat radius and brightness with pointer
   speed, so flicks burst and slow drags etch thin lines."
4. **Device tilt as gravity** — "On iOS, read DeviceMotion and add the gravity
   vector as a uniform body force, so tilting the phone pours the fluid."
   (Requires the iOS permission prompt on first touch.)
5. **Double-tap shockwave** — "On double-tap/double-click, emit a ring of 8–16
   radial splats from the tap point."
6. **HDR tonemapping** — "Accumulate dye in linear HDR and apply ACES filmic
   tonemapping at display time," for highlights that roll off like film instead
   of clipping.
7. **Color themes** — "Expose named palettes (full-spectrum, nebula purple, ember,
   glacier) selected by a key press / triple-tap, each defined as an HSV band +
   drift rate." The purple band from the reference image is hue 0.72–0.95.
8. **Audio reactivity** — "With mic permission, map low-frequency energy to ambient
   splat force so the fluid pulses with music." (Needs a user gesture to start.)
