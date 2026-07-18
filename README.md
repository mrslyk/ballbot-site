# Ballbot

**The robot that clears the court.**

A solar-assisted autonomous robot that finds tennis balls on a court, sweeps them into a hopper, and returns to its dock. Version 1 concept by Tim Parsa & Kevin Eisenbacher.

## Site

- [`index.html`](index.html) — the Ballbot marketing site: how it works, the pickup mechanism, corner handling, the vision brain, electronics & compute, specs, and roadmap.
- [`design.html`](design.html) — the Version 1 design document: scoping draft with schematics, the compute split, bill of materials, hopper sizing, and open decisions.

Both pages are self-contained static HTML (inline CSS + SVG, no build step, no dependencies). Open `index.html` in a browser, or serve the folder with any static file server:

```sh
python3 -m http.server
```
