# 0005. The features reach the window as vector tiles that tylertoo builds

- Status: Proposed
- Date: 2026-10-15

## Context

osmium-tool cuts the area out of the extract and writes its features as GeoJSON ([0002](0002-osmium-tool-selects-and-assembles.md)), and walkers draws them as a layer of tiles ([0004](0004-map-drawn-in-rust-in-a-native-window.md)).
In 0004's trial, the program built that layer in memory from the GeoJSON.

- Every area is cut by reading the whole extract and exported, about 50 seconds for a prefecture's worth of the Kanto extract ([phase 01's A007](../plan/phase-01-local-map.md#assumptions--risks)).
  - The maintainer judged a wait of several seconds too long each time the map is moved to another place while in use (said 2026-10-14).
  - Tiles built once are opened again without cutting, and the window reads only the tiles in view, reading more as the map moves.
- The maintainer looks at about 5 km in one sitting, zooming in for the smaller features, and wants the map to hold up to about a prefecture, growing by the municipalities added (said 2026-10-24).
- The program reads osmium-tool's export and keeps the features the rules admit before the tiles are built ([phase 01's Dependencies](../plan/phase-01-local-map.md#dependencies)), so it writes them in whatever form the tool that builds the tiles reads.
- Two tools build such tiles into one PMTiles archive:
  - [tippecanoe](https://github.com/felt/tippecanoe) reads GeoJSON, and has written PMTiles since 2.17 (its [CHANGELOG](https://github.com/felt/tippecanoe/blob/main/CHANGELOG.md), read 2026-10-25).
  - [tylertoo](https://github.com/geoparquet-io/tylertoo) reads GeoParquet alone and builds with a Rust engine, which its README claims 8.6 to 21.3 times faster than tippecanoe on its benchmarks; 0.8.0, released on 2026-10-06, is its latest release, and its first, 0.1.0, came out on 2026-02-24 (its [CHANGELOG](https://github.com/geoparquet-io/tylertoo/blob/main/CHANGELOG.md), read 2026-10-25).
    - `cargo install` builds it, with `protoc`, and each release attaches built binaries for Linux, macOS, and Windows (its CHANGELOG, 0.7.0 and 0.7.1).
- Of the other tools found that build vector tiles in Rust, none takes this phase's features as they are (read 2026-10-15):
  - [`freestiler`](https://github.com/walkerke/freestiler/) has its engine in Rust but is used as an R or Python package.
  - [`elivagar`](https://github.com/folknor/elivagar) builds a base map's layers from PBF in the Shortbread schema.
- [`osmium export`](https://docs.osmcode.org/osmium/latest/osmium-export.html) with no config writes each tagged closed way twice, as a line and as an area, unless it is tagged `area=yes` or `area=no` (its manual, read 2026-10-25, and a run of osmium 1.19.1 on a hand-written file, 2026-10-25).
  - Given a list of tags for lines alone, it wrote each closed way once: a line where a tag is in the list, an area otherwise, `area=yes` and `area=no` deciding first.
  - Given lists for both, it wrote a closed way matching both twice and one matching neither, such as one tagged `building:part`, not at all.
  - The box around Tokyo's mainland below, exported with no config, gave 9,683,164 features for 5,616,809 objects, 4,066,355 closed ways among them written twice, and the trials below built from that export.
- Trials with tippecanoe 2.79.0, which mise's conda backend installed on macOS arm64, on an Apple M3:
  - From the 108,074 features of the 5 km square around Tokyo Station, 37 MB of GeoJSON, zooms 14 to 16 with no simplification at the deepest zoom (`-pS`) built in 3.0 seconds into 7.0 MB; zoom 16 held all 108,074, zoom 15 84,040, and zoom 14 74,381 (2026-10-16).
    - Built with zoom 14 as the deepest, 6 of them were missing there, four small areas and two lines of two points; turning tiny-polygon reduction off (`-pt`) brought the areas back, and a detail of 16 (`-d16`) the lines; tippecanoe's README says it reduces tiny polygons at every zoom, the deepest included, unless told not to (read 2026-10-25), and lines finer than a tile's grid collapse (2026-10-15).
  - At zoom 16 alone, squares of 1 cm to 1 m on a side near Tokyo Station: the defaults and `-pS` kept those of 0.5 m and more; adding no tiny-polygon reduction (`-pt`) kept 0.2 m and more; adding a detail of 16 as well (`-d16`, a tile of 65,536 units rather than 4,096) kept every one (2026-10-25).
    - A tagged way of two nodes at one position, and a closed one of four, which `osmium export` writes as lines, came out of no build with `-pS -pt -d16` (2026-10-25).
  - From a box around Tokyo's mainland cut out of the Kanto extract, every tagged object with no scope rules applied, in 3.5 GB of GeoJSON, zooms 10 to 16 with `-pS` failed after 232 seconds, the further zooms' tiles being too large; with `--drop-densest-as-needed` added, which thins a whole zoom where any of its tiles goes over tippecanoe's limit of 500 KB, it built in 608 to 612 seconds into 640 MB (2026-10-24).
    - The archive's metadata records features dropped for size at zooms 10 to 14 and at neither 15 nor 16, and tiny polygons reduced at every zoom, 3,390 times at zoom 16.
    - Its zoom 16 tiles, decoded and counted by type, id, and whether a line or an area, lacked 50 areas and 8 lines, of 57 objects; 10 objects lacked every copy, 8 lines of 5 to 18 cm and areas of 0.04 m² or less, and the rest were closed ways whose line copy it held (2026-10-25).
    - Cutting the box took 7.5 seconds and exporting it 45 seconds.
- Trials with tylertoo 0.8.0, which `cargo install --locked` built on the same Apple M3 (2026-10-25):
  - DuckDB turned the export into GeoParquet with each feature's type, id, version, and timestamp, in 0.7 seconds for the 5 km square and 25.7 seconds into 545 MB for the box around Tokyo's mainland; writing GeoParquet from Rust, with the `geoparquet` crate, was not tried.
  - Zooms 14 to 16 of the 5 km square built in 0.39 seconds, zoom 16 holding all 108,074 features and zoom 14 39,689, where tippecanoe's held 74,381.
  - Zooms 10 to 16 of the box, with lines not joined (`--no-coalesce-lines`), built in 243 seconds into 625 MB, at a peak of 2.2 GB of memory.
    - 192 tiles at zooms 10 to 14 went over its limit of 500 KB and lost features until they fit; none at 15 or 16 did, and the largest zoom 16 tile was 149 KB (`tylertoo stats`).
    - Its tiles came to 63 MB at zooms 10 to 13 against tippecanoe's 28 MB, 2.0 MB against 0.7 MB at zoom 10 alone, 259 MB at zooms 14 and 15 against 345 MB, and 304 MB at zoom 16 against 268 MB (`tylertoo stats` on both archives).
    - Its zoom 16 tiles, decoded and counted the same way, held every one of the 5,616,809 objects, but lacked the area copies of 15 closed ways, of 0.0002 to 0.02 m², whose line copies they held (2026-10-25).
  - From a hand-written file at zoom 16 alone, it kept a way of two nodes at one position and a closed one of four, both as lines, and dropped a square of about 1 cm on a side, which its report counted as collapsed when encoded rather than as dropped.
  - Its help for 0.8.0 says a tile over the limit drops features (`tylertoo tiles --help`), where [its documentation](https://geoparquet-io.github.io/tylertoo/OVERVIEW_TUNING/), which describes changes after 0.8.0, says the default merges the densest features of such a tile into a neighbour, which keeps its own id and attributes, and `--size-valve drop` drops them instead (read 2026-10-25).
- [`osmium extract`](https://docs.osmcode.org/osmium/latest/osmium-extract.html) cuts by polygons given in an OSM file, merging several (multi)polygons into one area as long as they do not overlap (read 2026-10-24), which distinct municipalities' boundaries do not, though a city and one of its own wards do.
  - Cut by two wards of Tokyo that share an edge, the two boundary relations with only their own ways and nodes in one file, the Kanto extract gave no error and held exactly the objects of the two wards cut one at a time, 310,857, those reaching across the edge, 3,694, once (osmium 1.19.1, 2026-10-25).
  - A file that `osmium getid -r` had filled with the wards' sub-boundaries as well, which overlap them, cut out about a fifth fewer objects for one ward, so the boundaries are given alone.
  - Municipalities apart from each other were not tried.
- The program read a zoom 14 archive of that square, built with tippecanoe's default limits, with the [`pmtiles`](https://docs.rs/pmtiles/0.24.1) crate 0.24.1 and [`mvt-reader`](https://docs.rs/mvt-reader/2.5.0) 2.5.0, which walkers 0.61.0 reads PMTiles and vector tiles with, at the releases the trial resolved, and drew it as 0004's trial did, closer zooms as pieces of zoom 14 tiles, in two runs on an idle machine (2026-10-15).
  - Its first frame came 127 to 377 ms after it started, and the 30 tiles it needed were read in 115 ms.
  - While the map zoomed and moved by itself, the 95th percentile between frames was 17.7 ms.
  - Picking at one point took 0.36 to 0.44 ms, and found another way than 0004's trial did at the same point; the trial judged the smallest area on each area as the tile holds it, cut at the tile's edge, which may be why, and which pick is right was not looked at.
- The program then read each tool's archive of the box at zooms 10 to 16 over the same tour around Tokyo Station, zooms 13.6 to 17, two runs each, on a machine with about three quarters of its CPU idle (2026-10-25).
  - The 95th percentile between frames was 18.4 and 18.6 ms over tylertoo's tiles, and 17.4 and 18.8 ms over tippecanoe's.
  - The program built the 90 tiles the tour needed in 240 to 242 ms from tylertoo's and 346 to 348 ms from tippecanoe's, whose zoom 14 tiles hold more features.
  - Picking at the centre found a different way in each archive, two lines 2.3 and 2.1 m from the point, both within the trial's reach of 3.3 m; the trial picks the first line its tile holds, and tylertoo keeps the order of its input, which put the way 0004's trial found first, where tippecanoe put the other first (the decoded tiles, 2026-10-25).
  - The opening view of the whole box, at zoom 10, was not run.

## Decision

The area is the municipalities given, taken as the union of their boundaries in the extract, each without the boundaries inside it; osmium-tool cuts and exports it as 0002 says, writing each closed way once, as a line or an area by its tags; the program writes the features the rules admit as GeoParquet, and tylertoo builds them into one PMTiles archive in one run, which the window reads.
The archive holds zooms 10 to 16; at zoom 10 a window of 1,280 points spans about 160 km at Tokyo's latitude, so a prefecture's worth nearly fits one view.
Zoom 16 is the closest zoom it holds, the closest the trials over the box around Tokyo's mainland built: no tile there went over the size limit in either tool's build, where tiles at zooms 10 to 14 did; none at zoom 15 did either, so 15 might serve as the closest as well, which no build has tried.
A closer zoom would hold the same features in about three times as many tiles for each zoom, as zoom 16 held of zoom 15's (21,990 against 6,922), where the window draws closer views by enlarging zoom 16's tiles, as 0004's trial did with zoom 14's; none was tried.
Zoom 16 holds every feature, with no simplification there, but for any too small for a tile's grid to encode, which the report below shows; a closed way written once as such an area is then missing there whole.
Further out, tylertoo thins the features as its help for 0.8.0 describes, in these ways among others:

- it keeps one feature in each cell of a grid sized by the zoom, for points, lines, and areas apart;
- it drops a line or an area whose bounding box is under about half a pixel of a 256-pixel tile across at that zoom;
- it keeps fewer features for each zoom out, protecting sparse places, so dense places thin first;
- and it drops features from a tile over 500 KB until the tile fits.

Where features compete for a cell or for a zoom's count, the one whose last edit is oldest is kept first, so that the features longest without an edit stay in view further out, where tylertoo would otherwise keep the largest ([its documentation](https://geoparquet-io.github.io/tylertoo/OVERVIEW_TUNING/), read 2026-10-25); tylertoo ranks them by a numeric column given as `--sort-key`, and which end of it wins is left to building.
A tile over the size limit still keeps its largest features first, whatever that ranking, as the same documentation says.

Each feature drawn at any zoom is one object, coloured by its own date: lines are not joined (`--no-coalesce-lines`), which would make touching lines one stroke further out, and a tile over the size limit loses features rather than merging them into a neighbour, as 0.8.0 does and as `--size-valve drop` keeps for a release that merges by default.
The command that builds the archive compares the features it wrote with those zoom 16 holds and reports any that zoom 16 lacks, since tylertoo's report counts the features that collapse when encoded without naming them.
Each area carries its whole size, since a tile holds it cut at the tile's edge and the plan's R004 selects the smallest of several areas as a whole.
Adding municipalities builds the archive again over all of them, rather than building the new ones alone and joining their tiles to the old.

## Rejected alternatives

- **Every feature at zoom 14, with the tools' limits lifted**: the whole of a 5 km area would show every feature at once, but it holds every feature at the furthest zoom it draws, where tiles grow with the area's density, and thinning further out suits a map that is zoomed in on for the smaller features.
- **A furthest zoom closer than 10, such as 12**: the archive would build faster and smaller, and its opening view would be lighter, but a prefecture's worth would no longer fit one view; offered both, with zoom 10 recommended, the maintainer chose zoom 10 (said 2026-10-24).
- **Thinning sparse and dense places alike**, as tippecanoe's `--drop-fraction-as-needed` or tylertoo's `--drop-gamma 1` does: a sparse place would lose features it has few of, where thinning dense places first takes them from where many remain.
- **Keeping the largest of the features that compete for a cell or a zoom's count, tylertoo's default**: the map zoomed out would keep its roads and larger areas, but an older feature could be hidden there behind a newer, larger one, which is what the map is for showing.
- **Every feature kept at zoom 16**: a mapper zoomed in would never miss one, but tylertoo 0.8.0 offers no finer grid for a tile (`tylertoo tiles --help` names none), so it would take a closer zoom and about three times the tiles for each zoom added, for areas of 0.02 m² or less, under a pixel at zoom 16, where a pixel of a 256-pixel tile spans about 1.9 m at Tokyo's latitude, though walkers 0.63.0 lets the map zoom in to 26 (`src/zoom.rs`); the report names each such feature instead.
- **Building the tiles with tippecanoe**: it reads the GeoJSON osmium-tool writes, so nothing would write GeoParquet, but it built the box around Tokyo's mainland in 608 to 612 seconds against tylertoo's 243, and its zoom 16 lacked 50 areas and 8 lines where tylertoo's lacked 15 areas, which only options that make its tiles larger kept, short of a way at one position.
- **Building each municipality's archive alone and joining them**, with tippecanoe's `tile-join`, since tylertoo's `merge` joins only archives that hold disjoint tiles (`tylertoo merge --help`, 0.8.0, read 2026-10-25): adding a municipality would build only that one, but a feature crossing a boundary is in both cuts, so each feature would need a rule for which municipality owns it; the thinning further out would differ on either side of a boundary; and a newer extract rebuilds every municipality anyway to bring the dates up.
- **A rectangle of up to 5 km around a chosen point, built and opened one at a time**: each build is quick, but the map holds one rectangle, and growing it brings back the overlap of the previous alternative.
- **osmium-tool's export kept as it is, and read whole by the window, as 0004's trial read it**: it needs no tool beyond osmium-tool, but every opening reads every feature the export holds, which grows with the area, where the window reads from an archive only the tiles in view; offered the tiles and the layer built from the export, the maintainer chose the tiles (said 2026-10-15).
- **Tiling in the program itself**: cutting at a tile's edge, thinning, and encoding vector tiles would be the program's own work, which tylertoo already does; calling tylertoo's engine as a library from the program was not tried.

## Consequences

- **Dependencies added**: tylertoo, run as a separate program; a crate that writes GeoParquet; and the `pmtiles` and `mvt-reader` crates.
- **Risks**:
  - tylertoo is young, and a release can change what it drops or merges, as its documentation describes for one after 0.8.0; the report and the end-to-end test show a change at zoom 16, and the end-to-end test one that merges features further out, but a change in how much is thinned further out shows nowhere.
  - tylertoo has not been installed on CI's Linux runner, which CI would show by failing at install.
  - No crate has been tried writing GeoParquet that tylertoo reads, where the trial wrote it with DuckDB; without one, DuckDB would run as another program.
  - At about zoom 14, where a 5 km view sits, tylertoo's tiles held fewer features than tippecanoe's, 39,689 of the 5 km square's 108,074 against 74,381, so more of them show only once the map is zoomed in.
  - Keeping the oldest first has not been built or looked at, and further out it can keep a small old feature in place of a road or a larger area.
  - A tile over the size limit, as 192 tiles at zooms 10 to 14 of the box around Tokyo's mainland were, keeps its largest features rather than its oldest, so in the densest places an old, small feature can be missing further out.
  - Building a prefecture's worth takes about five minutes, and every municipality added or every newer extract takes that again; writing GeoParquet and reading zoom 16 back to compare it with what was written add to that.
  - A denser area, or one with finer features, can lose more at zoom 16 than the trials did, which the report shows.
  - The opening view at zoom 10 has not been drawn in a window, where tylertoo's tiles hold nearly three times the bytes tippecanoe's did, so whether it opens and moves within the plan's R008 is unknown, as is how full it looks.
  - An archive is as old as the extract it was built from, which the window shows by the data's date.
