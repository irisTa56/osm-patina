# 0004. The map is drawn in Rust, in a native window

- Status: Proposed
- Date: 2026-10-14

## Context

The phase was planned as a web page that draws the map with MapLibre GL JS, a library called from JavaScript ([phase 01 plan as it stood on 2026-10-13](https://github.com/irisTa56/osm-patina/blob/6436954/dev/plan/phase-01-local-map.md)).
That plan's A004 had MapLibre GL JS draw the densest 5 km square counted, 55 MB of GeoJSON, within 1.8 to 2.3 seconds of the base map showing, on an Apple M3 (2026-10-04).
The maintainer wants no JavaScript or TypeScript in the repository, in a state where it is plain that none is used rather than one where its absence has to be checked (said 2026-10-13).
They had pictured a JavaScript library with none of its JavaScript in the commits, and named drawing in Rust as closer to their ideal should that not be possible (said 2026-10-14).
A trial below showed that it is possible, and asked then to choose between the two, they chose drawing in Rust (said 2026-10-14).

- [walkers](https://crates.io/crates/walkers), the map widget for egui that the maintainer's other tool, qol-reweight, already draws with, draws the base map's tiles, then each layer added to the map, then the names it gathered from all of them, and only then what a plugin draws (`src/map.rs` of 0.60.0, read 2026-10-13).
  - A layer is anything that implements its public `Tiles` trait, and with its `mvt` feature a tile can be meshes with a colour on every vertex, and lines (`src/tiles.rs`, `src/lib.rs`, and `src/render/drawable.rs` of 0.60.0, read 2026-10-13).
- A trial drew the stand-in area's 108,074 features with walkers 0.61.0 and eframe 0.36.2 in a native window on an Apple M3, over OpenFreeMap's vector tiles, built in release, four runs on 2026-10-13 and 2026-10-14: one with walkers' bundled `openfreemap_bright` style, two with OpenFreeMap's `positron`, and one with `positron`'s expression for names rewritten.
  - The features were a layer of tiles the program built from osmium-tool's export, 30 of them in 66 to 68 ms.
  - While the map zoomed and moved by itself, frames came at the display's 60 a second, with the 95th percentile between frames at 17.3 to 17.9 ms.
  - The base map's names were drawn above the features.
- walkers 0.61.0 evaluates no `concat` (`src/expression.rs`, read 2026-10-14), which `positron` and walkers' own bundled `openfreemap_bright` use to label everything that has a name in a non-Latin script, so that gets no label.
  - With that expression replaced by the name in its own script and a font with Japanese glyphs added, the trial drew the names in Japanese, and frames kept to the same rate.
- MapLibre GL JS can be called from Rust compiled to WebAssembly with no JavaScript in the commits (a trial with wasm-bindgen 0.2.129 and trunk 0.21.14 in Chromium on macOS, 2026-10-13): the library's map was declared in Rust, and trunk wrote the script that loads the WebAssembly.
  - The page would then still run MapLibre GL JS's JavaScript, which the build would have to download, and the JavaScript that wasm-bindgen and trunk generate.
- Of the other ways found to draw a map in Rust, none was run (read 2026-10-13): [`galileo`](https://crates.io/crates/galileo) 0.2.1 was last released on 2025-07-11, [`maplibre`](https://crates.io/crates/maplibre) 0.0.3 draws neither names nor GeoJSON, and no way to put [`maplibre_native`](https://crates.io/crates/maplibre_native) 0.10.0 in a window was found in its documents.

## Decision

The program draws the map itself, in a native window built with eframe and egui, as qol-reweight does.
walkers draws the base map, and draws the features as a layer of tiles the program builds from osmium-tool's export, under the base map's names.
There is no page and no browser, and nothing the program runs is JavaScript.

## Rejected alternatives

- **A page drawn by MapLibre GL JS, with no JavaScript in the commits**: it keeps what the plan had measured of MapLibre GL JS, and is the route the maintainer had pictured, but offered both, with drawing in Rust recommended, they chose drawing in Rust; it would also add the WebAssembly target, trunk, and a download of the library to the build.
  - The maintainer left whether third-party and generated JavaScript is acceptable to how it is taken in (said 2026-10-14).
- **A page with JavaScript written by hand**: the fewest tools, but the maintainer chose not to write any.
- **The features drawn by a walkers plugin**: the simplest way to draw on a walkers map, but a plugin draws above the base map's names, which R010 has the other way round.
- **Another map renderer in Rust**: none of those found was run, and walkers is the one the maintainer's other tool already uses.

## Consequences

- **Dependencies added**: eframe with egui, and walkers with its `mvt` feature.
- **Risks**:
  - walkers is not yet at 1.0, and between 0.60.0 and 0.61.0 it changed two things the trial's code used, so an update may take changes to the program, which its build would show.
  - Whatever is named in a non-Latin script gets no label until the program rewrites the style's expression for names or walkers evaluates `concat`, which the eye check of the base map would show.
  - The window opens on the machine the command runs on, the one with osmium-tool and the extract.
  - What walkers draws on the GPU is not in the shapes a test of the window reads, so the features' colours are checked in what the program hands walkers and in an image a test renders, and the names above them by eye.
