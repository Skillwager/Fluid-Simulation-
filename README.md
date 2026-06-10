# Nebula — Interactive Fluid Dynamics

A real-time GPU fluid simulation in a single `index.html`. Move your mouse (desktop)
or drag your fingers (iOS — multi-touch) to stir glowing full-spectrum dye through
an incompressible Navier–Stokes velocity field.

**Live:** https://fluid-simulation-six.vercel.app

## Run it

No build step, no dependencies:

- Open `index.html` directly in a browser, **or**
- `python3 -m http.server` and visit `http://localhost:8000`, **or**
- Enable GitHub Pages on this repo and open it on your iPhone.

## How it works

Jos Stam's *Stable Fluids* method, entirely in fragment shaders over ping-pong
half-float framebuffers (WebGL2, with a WebGL1 + `OES_texture_half_float` fallback):

| Pass | Purpose |
|---|---|
| Advection | Semi-Lagrangian transport of velocity and dye |
| Curl + vorticity confinement | Re-injects the small swirls numerics destroy |
| Divergence + 20× Jacobi pressure | Enforces incompressibility |
| Gradient subtract | Projects velocity to divergence-free |
| Bloom (prefilter → blur pyramid → composite) | Makes hot magenta cores glow |
| Display | Gradient shading, dark-indigo background, dithering |

Velocity is simulated at ~144px, dye at ~1024px. Pointer movement injects a
Gaussian velocity impulse (direction = pointer delta, force ≈ 6000) plus a dye
splat. Each stroke seeds its own hue and drifts slowly around the color wheel as
it moves, so long drags leave ribbons that melt between adjacent colors. Each
finger on iOS gets its own tracked pointer and hue.

Tuning lives in the `config` object at the top of the script in `index.html`.

See [PROMPT.md](PROMPT.md) for the prompt that specifies this build.
