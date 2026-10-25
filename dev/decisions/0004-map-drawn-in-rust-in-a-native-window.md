# 0004. The map is drawn in Rust, in a native window

- Status: Proposed
- Date: 2026-10-14

## Context

The phase was planned as a web page that draws the map with MapLibre GL JS, a library called from JavaScript ([phase 01 plan as it stood on 2026-10-13](https://github.com/irisTa56/osm-patina/blob/6436954/dev/plan/phase-01-local-map.md)).
That plan's A004 had MapLibre GL JS draw the densest 5 km square counted, 55 MB of GeoJSON, within 1.8 to 2.3 seconds of the base map showing, on an Apple M3 (2026-10-04).
The repository is to hold no JavaScript or TypeScript, in a state where it is plain that none is used rather than one where its absence has to be checked, a preference.
Whether third-party and generated JavaScript fits that preference depends on how it is taken in, so a JavaScript library whose JavaScript stays out of the commits can fit it too, and a trial below showed that such a page can be built.

- [walkers](https://crates.io/crates/walkers), the map widget for egui that [qol-reweight](https://github.com/irisTa56/qol-reweight) already draws with, draws the base map's tiles, then each layer added to the map, then the names it gathered from all of them, and only then what a plugin draws (`src/map.rs` of 0.60.0, read 2026-10-13).
  - A layer is anything that implements its public `Tiles` trait, and with its `mvt` feature a tile can be meshes with a colour on every vertex, and lines (`src/tiles.rs`, `src/lib.rs`, and `src/render/drawable.rs` of 0.60.0, read 2026-10-13).
- A trial drew the 108,074 features of a 5 km square around Tokyo Station with walkers 0.61.0 and eframe 0.36.2 in a native window on an Apple M3, over OpenFreeMap's vector tiles, built in release, four runs on 2026-10-13 and 2026-10-14: one with walkers' bundled `openfreemap_bright` style, two with OpenFreeMap's `positron`, and one with `positron`'s expression for names rewritten.
  - The features were a layer of tiles the program built from osmium-tool's export, 30 of them in 66 to 68 ms.
  - While the map zoomed and moved by itself, frames came at the display's 60 a second, with the 95th percentile between frames at 17.3 to 17.9 ms.
  - The base map's names were drawn above the features.
  - A second trial drew the same features, read from a PMTiles archive, as the same kind of layer, at the same rate ([0005](0005-features-as-vector-tiles-tylertoo-builds.md)).
- walkers 0.61.0 evaluates no `concat` (`src/expression.rs`, read 2026-10-14), which `positron` and walkers' own bundled `openfreemap_bright` use to label everything that has a name in a non-Latin script, so that gets no label.
  - With that expression replaced by the name in its own script and a font with Japanese glyphs added, the trial drew the names in Japanese, and frames kept to the same rate.
- MapLibre GL JS can be called from Rust compiled to WebAssembly with no JavaScript in the commits (a trial with wasm-bindgen 0.2.129 and trunk 0.21.14 in Chromium on macOS, 2026-10-13): the library's map was declared in Rust, and trunk wrote the script that loads the WebAssembly.
  - The page would then still run MapLibre GL JS's JavaScript, which the build would have to download, and the JavaScript that wasm-bindgen and trunk generate.
- Of the other ways found to draw a map in Rust, none was run (read 2026-10-13): [`galileo`](https://crates.io/crates/galileo) 0.2.1 was last released on 2025-07-11, [`maplibre`](https://crates.io/crates/maplibre) 0.0.3 draws neither names nor GeoJSON, and [`maplibre_native`](https://crates.io/crates/maplibre_native) 0.10.0 has a `wgpu` feature that renders into a texture shared with the host's wgpu device (`set_device_queue` and `take_texture` in `src/renderer/image_renderer.rs`, read 2026-10-25), though its README does not list the feature, and it takes wgpu 29 where eframe 0.36.2 takes wgpu 30, so the two do not share a device at these releases.

## Decision

The program draws the map itself, in a native window built with eframe and egui, as qol-reweight does.
walkers draws the base map, and draws the features as a layer of tiles the program builds from the archive of [0005](0005-features-as-vector-tiles-tylertoo-builds.md), under the base map's names.
The map is drawn in no page and no browser, and nothing the program runs is JavaScript; a link it shows opens in the user's own browser.

## Rejected alternatives

- **A page drawn by MapLibre GL JS, with no JavaScript in the commits**: it keeps what the plan had measured of MapLibre GL JS, but the page would still run MapLibre GL JS's JavaScript and what wasm-bindgen and trunk generate, so that no JavaScript is used would rest on how those are taken in, where a native window runs none; its tests would drive a headless browser, which had not been run on CI's Linux runner ([that plan's A009](https://github.com/irisTa56/osm-patina/blob/6436954/dev/plan/phase-01-local-map.md)), where the layer walkers draws hands a test meshes and lines it can check as Rust values; and it would add the WebAssembly target, trunk, and a download of the library to the build.
- **A page with JavaScript written by hand**: the fewest tools, but it puts JavaScript in the repository, against the preference above.
- **The features drawn by a walkers plugin**: the simplest way to draw on a walkers map, but a plugin draws above the base map's names, which R010 has the other way round.
- **The features drawn by walkers' own PMTiles source and a style**: it would need no layer of the program's own, but walkers 0.63.0 draws no circle layer, so no point (`src/mvt.rs`), and its PMTiles source reads no deeper than zoom 15 (`DEFAULT_MAX_ZOOM` in `src/pmtiles.rs`), where the archive of 0005 holds its features at zoom 16 (read 2026-10-15 and 24).
- **Another map renderer in Rust**: none of those found was run, and walkers is the one qol-reweight already draws with, whose window tests A011 of [phase 01's plan](../plan/phase-01-local-map.md#assumptions--risks) rests on.

## Consequences

- **Dependencies added**: eframe with egui and its wgpu backend, which walkers' renderer needs; walkers with its `mvt` feature; and a crate that finds a font on the machine for names in Japanese, as qol-reweight uses `fontdb` for.
- **Risks**:
  - walkers is not yet at 1.0, and between 0.60.0 and 0.61.0 it changed two things the trial's code used, so an update may take changes to the program, which its build would show.
  - Whatever is named in a non-Latin script gets no label until the program rewrites the style's expression for names or walkers evaluates `concat`, which the eye check of the base map would show.
  - What walkers draws on the GPU is not in the shapes a test of the window reads, so a feature lost in walkers' own drawing, or in a renderer never installed, is caught only by the test that renders an image, where one missing from the tiles the program hands walkers is caught from those tiles; where that test runs on a developer's machine alone, a change that stops walkers drawing the features passes CI, which the next run of it there would show.
  - The map is a native window, so it opens only on a machine the program is installed on.
