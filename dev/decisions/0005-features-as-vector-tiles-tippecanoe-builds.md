# 0005. The features reach the window as vector tiles that tippecanoe builds

- Status: Proposed
- Date: 2026-10-15

## Context

osmium-tool cuts the rectangle out of the extract and writes its features as GeoJSON ([0002](0002-osmium-tool-selects-and-assembles.md)), and walkers draws them as a layer of tiles ([0004](0004-map-drawn-in-rust-in-a-native-window.md)).
In 0004's trial, the program built that layer in memory from the GeoJSON.

- Every rectangle is cut by reading the whole extract, about 5 seconds for the Kanto extract ([phase 01's A007](../plan/phase-01-local-map.md#assumptions--risks)), and the roadmap's scope has the point later picked on the map or taken from the current location ([roadmap](../ROADMAP.md#scope)).
  - The maintainer judged a wait of several seconds each time the point is set or moved while the map is in use too long (said 2026-10-14).
  - Tiles built once are opened again without cutting, and can be built once for an area wider than one rectangle.
- [tippecanoe](https://github.com/felt/tippecanoe) builds vector tiles from GeoJSON, and since 2.17 writes them as one PMTiles archive (its README, read 2026-10-15).
- None of the tools found that build vector tiles in Rust takes this phase's features as they are (read 2026-10-15):
  - [`tylertoo`](https://github.com/geoparquet-io/tylertoo) reads GeoParquet alone, and its README calls it a work in progress.
  - [`freestiler`](https://github.com/walkerke/freestiler/) has its engine in Rust but is used as an R or Python package.
  - [`elivagar`](https://github.com/folknor/elivagar) builds a base map's layers from PBF in a schema of its own.
- A trial built the stand-in area's 108,074 features, 37 MB of GeoJSON, into archives with tippecanoe 2.79.0, which mise's conda backend installed on macOS arm64, on an Apple M3 (2026-10-15).
  - With its defaults and no simplification at the deepest zoom (`-pS`), 6 features were missing at zoom 14: four small areas and two lines of two points.
  - With small areas kept as they are (`-pt`) and coordinates 16 times finer within a tile (`-d16`), zoom 14 held all 108,074, built in 2.8 seconds into 2.9 MB, but two tiles over Tokyo Station had coarser coordinates, and each lacked a line that a tile beside it held.
  - With tippecanoe's limits on a tile's size and number of features lifted as well, every tile kept the finer coordinates, the largest at 599 KB, built in 2.2 seconds into 3.1 MB (2026-10-16).
  - With `-pt` and tippecanoe's default limit of 500 KB a tile, the zoom 13 tile over Tokyo Station could not be made small enough.
  - Built with those options, 40,000 small areas and short lines written into one zoom 14 tile came out with coarser coordinates and 14,688 lines missing, and tippecanoe ended without an error; with its limits on a tile's size and number of features lifted, all 40,000 were kept in a tile of 1.2 MB (2026-10-15 and 16, counted by `tippecanoe-decode --stats`).
- The program read the zoom 14 archive built with the default limits with the [`pmtiles`](https://docs.rs/pmtiles/0.24.0) crate 0.24.0 and [`mvt-reader`](https://docs.rs/mvt-reader/2.5.0) 2.5.0, which walkers 0.61.0 reads PMTiles and vector tiles with, and drew it as 0004's trial did, closer zooms as pieces of zoom 14 tiles, in two runs.
  - Its first frame came 127 to 377 ms after it started, and the 30 tiles it needed were read in 115 ms.
  - While the map zoomed and moved by itself, the 95th percentile between frames was 17.7 ms.
  - Picking at one point took 0.36 to 0.44 ms, and found another way than 0004's trial did at the same point; the trial judged the smallest area on each area as the tile holds it, cut at the tile's edge, which may be why, and which pick is right was not looked at.
- Read by the same program, the archive with the limits lifted gave a first frame 145 to 448 ms after start, its 30 tiles read in 116 to 119 ms, and a median of 3.3 to 4.0 ms of the window's own work in a frame (two runs, 2026-10-24).
  - The 95th percentile between frames was 31.3 to 31.8 ms, but not on an idle machine: other work on the machine that day slowed the default-limit archive more, to 46.9 ms.

## Decision

The rectangle is cut and exported with osmium-tool as 0002 says, the features the rules admit are kept, and tippecanoe builds them into a PMTiles archive, which the window reads.
The archive holds every feature at zoom 14, none left out, with no simplification at that zoom, small areas kept as they are, coordinates finer than tippecanoe's default, and no limit on a tile's size or number of features, since past a limit tippecanoe coarsens a tile and drops features without an error.
Each area carries its whole size, since a tile holds it cut at the tile's edge and the plan's R004 selects the smallest of several areas as a whole.
The map draws the features at zoom 14 and closer, and further out it draws none and says that zooming in shows them.

## Rejected alternatives

- **osmium-tool's export kept as it is, and read whole by the window, as 0004's trial read it**: it needs no tool beyond osmium-tool and also spares the cut on a second opening, but every opening reads every feature the export holds, which grows with the area, where the window reads from an archive only the tiles in view, 30 of them over the trial's whole tour; offered the tiles and the layer built from the export, the maintainer chose the tiles (said 2026-10-15), and a kept export was not among what they were offered.
  - How long reading the export whole takes was not measured.
- **Building the tiles in Rust**: it would keep the tools to Rust, but cutting at a tile's edge and encoding vector tiles would be the program's own work, which tippecanoe already does, and no Rust tool found takes the features as they are.
- **Features at every zoom, some left out further out**: the map would show features however far out it is, but not every feature; offered both, with zoom 14 and closer recommended since the whole rectangle shows at that zoom with every feature, the maintainer chose zoom 14 and closer (said 2026-10-15).
- **Tiles to zoom 16, with tippecanoe thinning zooms 14 and 15 as it does by default**: each tile stays small however dense the area, but zoom 14 held 74,381 of the stand-in area's 108,074 features and zoom 15 84,040, so every feature would show only at zoom 16, where a window of 1,280 points spans about 2.5 km (built in 3.0 seconds into 7.0 MB, 2026-10-16); the maintainer chose zoom 14 alone with the limits lifted (said 2026-10-16).
- **Every feature at zooms further out than 14**: tiles over Tokyo Station go over tippecanoe's default limit at zoom 13, and lifting the limit leaves tiles whose drawing was not tried, since the window trials drew zoom 14 and closer alone, while a 5 km rectangle fits a window of 1,280 pixels at zoom 14, which spans about 10 km at Tokyo's latitude.
- **One archive for an area wider than the rectangle**: it would spare every later rectangle in it the cut, but how long it takes and how large it is are not measured.

## Consequences

- **Dependencies added**: tippecanoe, installed by mise's conda backend and run as a separate program, and the `pmtiles` and `mvt-reader` crates.
- **Risks**:
  - conda-forge's build of tippecanoe on Linux has not been installed, which CI would show by failing at install.
  - In an area denser than the stand-in, a zoom 14 tile can be larger than any the window has drawn, which the map lagging there would show.
  - An archive is as old as the extract it was built from, which the window shows by the data's date.
