# Phase 01: Local map for one area

- Status: In progress
- Roadmap: [Local map for one area](../ROADMAP.md#phases)

## Goal

On a developer's machine, one command builds the features of chosen municipalities of a regional extract into vector tiles, and another opens them in a window where every feature in scope is coloured by the date of its last edit, over a base map.
A mapper looking at the window sees which features have gone longest without one, and can open any of them on openstreetmap.org.

## Requirements & Constraints

- **R001**: Given a regional extract on disk and a file listing one or more municipalities, each by name or by the id of its boundary relation in OSM, one command builds the map of their area as an archive of vector tiles, and given that archive, another command opens a window showing its whole area ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
  - A list with no municipality, or with one the extract does not hold, is refused, with a message that says so.
  - A name that matches several boundaries in the extract, such as a ward name several cities share, is refused, with the ids of the boundaries it matches.
  - Municipalities whose boundaries overlap, such as a city given with one of its own wards, are refused, with a message that says which.
  - The area is the union of the municipalities' boundaries, whether or not they touch.
  - Where the whole area does not fit the window at the furthest zoom the archive holds, the window opens at that zoom over the area's centre.
  - Adding a municipality is adding it to the file and building the archive again over all of them; the program keeps no list of its own.
- **R002**: The map draws the features in the area that the [roadmap's scope](../ROADMAP.md#scope) covers, each one once, every one of them at zoom 16 and closer but for any tylertoo drops there, which the build reports ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
  - A closed way is one feature, drawn as a line or as an area by its tags, not as both.
  - tylertoo drops at zoom 16 an area too small for a tile's grid to encode (A013); which options to build with is left to building.
  - The command that builds the archive compares the features it wrote with those zoom 16 holds, and reports the type and id of any that zoom 16 lacks.
  - Further out than zoom 16, tylertoo thins the features as 0005 describes, so one can be missing there until the map is zoomed in.
  - A feature drawn further out is never joined with another, so what is drawn there is selected and coloured as the one object it was built from.
  - Further out than the furthest zoom the archive holds, zoom 10, no feature is drawn, and the window says that zooming in shows them.
  - A feature is built from a tagged node, a tagged way, or a relation of type multipolygon or boundary.
    - A relation of any other type is not a feature.
  - A feature is in the area when one of its nodes is, and it is then drawn whole, past the area's edge.
    - One with every node outside is not drawn.
  - A feature outside that scope is not drawn, such as an administrative boundary or a feature tagged as indoor.
  - A feature osmium-tool does not produce for the area is not drawn, by a preference for drawing what osmium-tool produces over recovering in this phase what it leaves out.
    - The cases known are a feature with no node inside the area, a boundary relation that reaches past its edge, and a geometry that cannot be assembled (runs of osmium 1.19.1, 2026-10-03 and 2026-10-04).
    - The first two lie across the area's edge, and are left out on the judgment, not checked against any source, that features missing there change little of what a mapper looks for.
      - Completing boundary relations would also bring in every member of each, and [`osmium extract`'s manual](https://docs.osmcode.org/osmium/latest/osmium-extract.html) warns that boundary relations "can be huge" (read 2026-10-25).
    - The third has no valid shape for the map to draw: `osmium export` omits a closed way or a relation that cannot be assembled into a valid polygon ([its manual](https://docs.osmcode.org/osmium/latest/osmium-export.html), read 2026-10-25).
- **R003**: Each feature is coloured by the date of its last edit, on one scale shared by points, lines, and areas, and a legend says which date each colour stands for.
  - The date is the `timestamp` of the OSM object the feature was built from.
  - A feature whose object has no `timestamp`, as in an extract written without metadata, is drawn grey, which the legend gives as an unknown date.
- **R004**: Selecting a feature shows its type, id, version, and last-edit date, and a link to that object on openstreetmap.org.
  - A version or a date the object lacks, as in an extract written without metadata, is shown as unknown.
  - Where several features lie under the point selected, a point or a line is selected before any area, and of several areas the smallest as a whole, so that a building or a shop within a larger area can be reached.
  - Of several points or lines there, the nearest is selected, rather than whichever its tile happens to hold first, which differs by the tool that built the tiles (0005).
- **R005**: The window states the date of the data it shows, since the data is a day or more behind OSM ([0001](../decisions/0001-regional-extract-as-data-source.md)), and an archive is as old as the extract it was built from.
  - Where the extract's header gives no date, the window says that the data's date is unknown, rather than give the file's own date, which is when the file was written, usually near the day it was downloaded, and tells little of how old the data is.
- **R006**: The map credits OpenStreetMap in one of its corners with "© OpenStreetMap contributors", linked to `https://www.openstreetmap.org/copyright`, beside the base map's own credit, and the credits never collapse.
  - The [Attribution Guidelines](https://osmfoundation.org/wiki/Licence/Attribution_Guidelines) allow a credit to collapse on interaction (checked 2026-10-03), but one that stays needs no reading of those conditions.
- **R007**: Nothing the phase commits reveals where a developer works, as [CLAUDE.md](../../CLAUDE.md#keeping-a-developers-location-private) requires.
  - The file listing the municipalities sits outside the tracked files, and no tracked file gives them a default.
  - Everything the commands write goes where git ignores it, since it is cut from the developer's area.
- **R008**: At the stand-in area, on a developer's machine, with the window at 1,280 by 800 points, the commands and the window keep to these times:
  - The archive is built within 15 minutes of its command starting, from an extract of about 500 MB.
  - Within 1 second of the other command starting, the window shows the features of its opening view.
  - While the map is panned and zoomed, the interval between frames is within 20 ms at the 95th percentile, both over the opening view of the whole area and around Tokyo Station at zoom 14 and closer, where about 108,000 features lie within a 5 km square.
    - The opening view is held to it as well, since it is the first view the window shows and is drawn from the further zooms, whose tiles are the largest: in the archive of Tokyo's mainland, tiles at zooms 10 to 14 went over tylertoo's limit of 500 KB, and none at 15 or 16 did ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
- **R009**: The program is written in Rust ([0003](../decisions/0003-rust-for-the-program.md)) and draws the map itself, in no page or browser and with no JavaScript ([0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)), and its automated checks run in `mise run pre-commit` and in CI.
  - The tests that drive the window run on a developer's machine first, and in CI as far as its runner can run them, a preference that makes CI the best effort for these tests.
- **R010**: The features are drawn over a base map in muted colours, with its place and street names above them.
  - Each name is the place's own, in its own script, drawn with a font found on the machine.
    - Where no font for a script is found, the window still opens, and says that some names cannot be drawn.
  - When the base map's service does not answer, the features are still drawn.
  - The base map's data has its own date, so a recent building can be on one layer and not the other.
- **R011**: The rules that bring a feature into scope or take it out are data in the repository, so that a rule is added or replaced without changing the program.

### Out of scope

- **Counting a fixed shape as an edit**: a way's `timestamp` does not change when only its nodes move, and a multipolygon's is its relation's own, so a reshaped feature shows an older date than its shape deserves.
  - Roland Olbricht's [Handling Timestamps in OpenStreetMap](https://dev.overpass-api.de/misc/timestamps.pdf) (2018) found that 20% to 30% of the ways changed in a period kept their version (checked 2026-10-03).
  - This goes to the Vertex-aware freshness phase.
- **Which attribute an edit touched**: freshness is per feature, so a shop whose opening hours changed last week shows as fresh.
  - The roadmap has no phase for it.
- **Showing the user's current location and moving the map to it**: the Location phase.
- **Downloading or updating the extract**: the developer downloads it and passes its path, which keeps this phase to the map.
- **Assembling geometry in Rust**: not in this phase ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)).
- **Opening the map anywhere but a developer's machine**: the Publishing phase.

## Assumptions & Risks

- **A001**: A Geofabrik extract keeps every object's `version` and `timestamp`. Source: `osmium fileinfo -e` on the Kanto extract, run 2026-10-04 ([0001](../decisions/0001-regional-extract-as-data-source.md)).
  - Risk: features come out without a date and are drawn grey (R003), noticed by `osmium fileinfo -e` not printing `version+timestamp` for all objects.
- **A002**: `osmium export` writes each feature's type, id, version, and timestamp. Source: a run of osmium 1.19.1 on a hand-written file, 2026-10-03.
  - A node, a way, a closed way, or a multipolygon relation without a version or a timestamp comes out with 0 for it (a run on a hand-written file, 2026-10-25), and since OSM's versions start at 1 and its timestamps fall after 2004, the program takes 0 as unknown (R003, R004).
  - Risk: the map has no number to colour by, noticed by the end-to-end test.
- **A003**: The densest of the four 5 km squares counted holds about 158,000 features before the scope narrows them, half as many again as the square around Tokyo Station, which R008's interval between frames is stated for. Source: counts of the four squares.
  - The square around Tokyo Station held about 108,000 and a denser square in the same extract about 158,000 on 2026-10-04, as features of `osmium export` of each cut: the Tokyo Station square's given a list of area tags alone, which wrote each closed way once, and the denser square's given lists of line and area tags, which wrote 65 closed ways twice and left out closed ways whose tags were in neither list, so that it held 157,752 objects where the same cut exported with both settings `true` held 158,593.
  - A 5 km square in central London held about 151,000 on 2026-10-03, and another city centre about 92,000 on 2026-09-28 (Overpass counts by key).
  - Risk: an area denser than the square around Tokyo Station, such as the densest square counted, answers panning and zooming more slowly than R008's interval, which no check measures, since no trial has drawn so dense an area in the window (A004), so R008 is not stated for one; noticed by the map lagging when one is opened.
- **A004**: walkers draws the features around Tokyo Station within R008's frame interval. Source: the trials in [0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md) and [0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md), the 108,074 features of the 5 km square around it over OpenFreeMap's vector tiles while the map zoomed and moved by itself, walkers 0.61.0 and eframe 0.36.2 on an Apple M3, four runs from GeoJSON on 2026-10-13 and 2026-10-14 and two from a zoom 14 archive on 2026-10-15.
  - The 95th percentile between frames was 17.3 to 17.9 ms, the median 16.7 ms, which is the display's own rate.
  - The window's own work in a frame had a median of 0.15 ms, and of 4.2 to 4.6 ms once the base map's names were drawn in Japanese.
  - From the archive, the first frame came 127 to 377 ms after the program started, and the 30 tiles it needed were read in 115 ms.
  - Not measured: the window over the opening view of such an archive, at zoom 10; and A003's denser square.
  - Over archives of zooms 10 to 16 of the box around Tokyo's mainland, the 95th percentile around Tokyo Station was 18.4 and 18.6 ms over tylertoo's tiles, and 17.4 and 18.8 ms over tippecanoe's, on a machine with about three quarters of its CPU idle, 2026-10-25 ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
  - Risk: a slower machine, the program's own work on top of the trial's, or the tiles of zooms 10 to 14, which run up to 383 KB compressed and 616 KB decompressed where zoom 16's reach 149 KB and 311 KB, in tylertoo's archive (0005), miss R008, noticed by the scale check.
- **A005**: The extract's header states the date of its data. Source: the Kanto extract's header carries `osmosis_replication_timestamp=2026-10-03T20:20:50Z` (`osmium fileinfo`, run 2026-10-04).
  - A file written by hand carries the same header once osmium 1.19.1 converts it to PBF with `--output-header` (run 2026-10-11), so a fixture can state a date as an extract does.
  - Risk: another extract lacks it, and the window says the data's date is unknown (R005), noticed by `osmium fileinfo` printing no replication timestamp.
- **A006**: mise's conda backend installs osmium-tool on CI's Linux runner as it does on macOS arm64. Unverified on Linux; [conda-forge](https://anaconda.org/conda-forge/osmium-tool) lists a `linux-64` build (checked 2026-10-03).
  - Risk: the end-to-end test cannot run in CI, noticed by the job failing at install.
- **A007**: Cutting a prefecture's worth out of an extract of about 500 MB takes seconds, and exporting it under a minute. Source: a box around Tokyo's mainland cut out of the Kanto extract, 7.5 seconds to cut and 45 seconds to export 9,683,164 features with no scope rules applied, the 5,616,809 objects with each tagged closed way written twice, osmium on an Apple M3, 2026-10-24 ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
  - The cut reads the whole extract, so its time grows with the extract, and the export's with the area.
    - A larger extract has not been timed.
  - Risk: every build from an extract of several GB keeps a developer waiting, noticed by timing one.
    - Cutting a smaller extract once and building from that is the way out.
- **A008**: OpenFreeMap serves its `positron` style and tiles without a key or a limit on views. Source: [its site](https://openfreemap.org/) and a request for the style, 2026-10-04.
  - Its [terms](https://openfreemap.org/tos/) say the service may be discontinued at any time.
  - Risk: the base map disappears, noticed by features drawn on a blank window.
    - Another style URL replaces it.
- **A009**: walkers draws the names it gathers from every layer above those layers, so features handed to it as a layer of tiles sit under the base map's names. Source: walkers 0.60.0's `src/map.rs`, read 2026-10-13, and the screenshots of the trial in [0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md) at zoom 17.
  - Risk: a release of walkers draws them in another order, noticed by names under the features in the eye check.
- **A010**: walkers draws `positron`'s names once the style's expression for names is replaced by one it evaluates. Source: walkers 0.61.0's `src/expression.rs` has no `concat`, which that expression uses, and the trial drew names in Japanese with it replaced by the place's own name and a font with Japanese glyphs from macOS, 2026-10-14 ([0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)).
  - Risk: the style changes its expression for names, noticed by a base map without names in the eye check.
- **A011**: A test written in Rust can drive the window without showing it, and read what the checks of the window below need. Source: qol-reweight's window tests, which drive its egui window with egui_kittest 0.36.2, read the shapes a frame draws and the window's widgets through AccessKit, and send clicks and drags (its `src/window.rs`, read 2026-10-13). Unverified for this window, and on Linux, where qol-reweight's window tests do not run because they need a Japanese font from macOS.
  - Risk: something the phase must check cannot be reached that way, noticed by a check below that ends up done by hand.
  - Risk: the tests of the window cannot run on CI's runner, noticed by their failing in CI alone.
    - R009 then leaves them to a developer's machine.
- **A012**: A test can render the window to an image, what walkers draws on the GPU included. Unverified: egui_kittest 0.36.2's `wgpu` feature has `Harness::render`, which runs egui's wgpu renderer over a frame, paint callbacks included (its source, read 2026-10-14), but it has not been run with walkers, and it needs a wgpu adapter, which a runner with no GPU has only through a software rasterizer such as lavapipe, which the crate prefers (its `src/wgpu.rs`, read 2026-10-14).
  - The [`ubuntu-24.04` runner image](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md), which CI's `ubuntu-latest` names, lists no Mesa, Vulkan driver, or software rasterizer among its packages (image 20261004.327.1, read 2026-10-14), so CI has to install one.
  - Risk: walkers' drawing does not come out in the image, noticed by the check of drawn features ending up done by hand.
  - Risk: the test cannot render on CI's runner, noticed by its failing in CI alone, which R009 then leaves to a developer's machine.
- **A013**: tylertoo builds a prefecture's worth into an archive at zoom 10 to 16 within about five minutes, and zoom 16 holds every feature but areas too small for a tile's grid. Source: the trials in [0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md), 243 seconds for a box around Tokyo's mainland from an export of 9,683,164 features, each tagged closed way in it twice, whose zoom 16 lacked only the area copies of 15 closed ways, of 0.0002 to 0.02 m², and a square of about 1 cm on a side dropped at zoom 16 from a hand-written file.
  - Written once, as 0005 decides, such a closed way is lost from zoom 16 whole, which the build's report shows (R002).
  - Risk: zoom 16 lacks features a mapper would look for, noticed by the build command's report (R002) at the scale check; tylertoo's options are then looked at for one that keeps them.
- **A014**: mise's `github` backend installs tylertoo from the binaries its release attaches, on CI's Linux runner and on macOS arm64. Source: 0.8.0's release attaches binaries for `aarch64-apple-darwin`, `x86_64-unknown-linux-gnu`, and `x86_64-unknown-linux-musl` (its release on GitHub, read 2026-10-25). Unverified: the trial built it with `cargo install --locked` instead, which needs `protoc` (0005), and installed nothing through mise.
  - Risk: the end-to-end test cannot run in CI, noticed by the job failing at install.
- **A015**: The feature under a point can be found from the tile under it, within a millisecond. Source: 0.36 to 0.44 ms at one point in the trial in [0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md).
  - A tile holds each area cut at its edge, so the archive carries each area's whole size, which R004's rule ranks by.
  - Risk: selecting in a tile denser than those around Tokyo Station takes long enough to notice, noticed when selecting there.
- **A016**: The command that builds the archive keeps to R008's time. Source: its parts as measured on an Apple M3 for a box around Tokyo's mainland, 7.5 seconds to cut, 45 seconds to export, and 243 seconds for the tiles (A007, A013), with no scope rules applied and each tagged closed way written twice, either of which would leave fewer features.
  - DuckDB took 25.7 seconds to turn that export into GeoParquet (0005).
  - Not measured: taking the municipalities' boundaries out of the extract, keeping the features the rules admit, the program writing the GeoParquet tylertoo reads, reading zoom 16 back to compare it with what was written (R002), and the whole command in one run.
  - Risk: those parts take more than the 10 minutes or so left, noticed by the scale check.
- **A017**: The municipalities' boundaries can be taken from the extract and given to `osmium extract` as one area. Source: its manual says a file of several (multi)polygons is cut as one area if they do not overlap (read 2026-10-24), and distinct municipalities do not, R001 refusing those that do; and two wards of Tokyo that share an edge, cut from the Kanto extract by both boundaries alone, without their sub-boundaries, in one file with osmium 1.19.1, held exactly the objects of the two wards cut one at a time, those reaching across the edge once (2026-10-25).
  - Not tried: municipalities apart from each other.
  - Risk: features of a municipality apart from the others are lost, noticed by the end-to-end test.
  - Risk: a boundary reaches the cut with the boundaries inside it, which overlap it and leave part of its features out with no error (0005), noticed by the end-to-end test.
  - Risk: a municipality whose boundary reaches past the extract's edge comes out incomplete, noticed by the command refusing it or by features missing near that edge.
- **A018**: A Rust crate writes GeoParquet that tylertoo reads, with each feature's type, id, version, and timestamp. Unverified: the trial wrote it with DuckDB ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)), and no crate was tried.
  - Risk: none writes what tylertoo reads, noticed by the end-to-end test failing to build; DuckDB, run as another program as the trial ran it, is then the way out.

## Decisions

- [0001. A regional extract kept on disk is the data source](../decisions/0001-regional-extract-as-data-source.md)
- [0002. osmium-tool selects the area and assembles geometry](../decisions/0002-osmium-tool-selects-and-assembles.md)
- [0003. The program is written in Rust](../decisions/0003-rust-for-the-program.md)
- [0004. The map is drawn in Rust, in a native window](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)
- [0005. The features reach the window as vector tiles that tylertoo builds](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)
- The Rust toolchain, osmium-tool, and tylertoo at their latest releases rather than at fixed versions, because lychee and rumdl are declared that way, and CI installs the current release on each run, the weekly one included, so a release the checks fail on shows there.
- One command that builds the archive and another that opens it, rather than one that does both, because an archive once built opens in under a second (A004) without cutting the extract again.
- Municipalities, by name or by the id of their boundary relation, rather than a centre point and a size, because the map is meant to grow by the municipalities added, up to about a prefecture.
- The municipalities listed in a file the developer keeps outside the repository, rather than given as arguments each time, written into the archive, or kept by the program, because adding or removing one is then a line of that file, and the program keeps no state.
- OpenFreeMap's `positron` style as the base map rather than OpenStreetMap's standard tiles or a self-hosted Protomaps archive, because it is grey, so the colours of R003 keep their meaning over it, it needs no key and no server, and it is a vector style, so its labels can sit above the features.
  - The standard tiles are coloured by kind of feature and are images, so their names cannot sit above the features (checked 2026-10-03).
  - A Protomaps archive is a file the developer downloads, cut with `pmtiles extract` from a planet build of about 120 GB, and keeps up to date, where OpenFreeMap's tiles are requested as the window needs them ([its downloads page](https://docs.protomaps.com/basemaps/downloads), read 2026-10-15).
- Names in their own script rather than in Latin script alone, by a preference for names as they are written where the mapper is, which in Japan is Japanese, on the judgment, not checked against any source, that a mapper in Japan reads names most easily in Japanese, and a font for it is on the developer's macOS (A010).
- An allow-list of tag keys rather than a deny-list, because `osmium export` turns a boundary relation into a polygon over everything inside it (run of 2026-10-03), and a key nobody thought to deny would do the same.
  - A feature the list admits is still left out when it is tagged as indoor, since a shop inside a building carries the same `shop` key as one on the street.
  - Which keys stand for each of the scope's kinds of feature is settled while building.
- What a test can check is checked that way rather than by eye, because the test goes on guarding the behaviour after the phase closes.
  - Three things are left to the eye:
    - what only an eye judges, that the base map is muted and its names can be read above the features;
    - what shows only while the base map's service answers, such as its own credit and its names, since a test that waited on that service would fail whenever it did not answer;
    - and what needs the stand-in area's data, which is not in the repository.
- Test fixtures are OSM files written by hand rather than cut from a real area, because data cut from OSM would bring its licence into the repository.
- The scale check and the screenshots use a stand-in area, Tokyo's mainland, every municipality of Tokyo but its islands, cut from Geofabrik's [Kanto extract](https://download.geofabrik.de/asia/japan/kanto.html), and the view around Tokyo Station, a landmark well known enough that it says nothing of where a developer works, because they need real data and R007 rules out a developer's own area.

## Dependencies

- **osmium-tool**: taking the municipalities' boundaries, selecting the area, and assembling geometry ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)), declared in `mise.toml` at its latest release, which was 1.19.1 on 2026-10-13, and installed by `mise install` through the conda backend.
- **tylertoo**: building the area's features into a PMTiles archive ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)), declared in `mise.toml` at its latest release, which was 0.8.0 on 2026-10-25, and installed by `mise install` through the `github` backend, from the binaries its releases attach.
- **Rust toolchain**: building and testing the program, declared in `mise.toml` at its latest release.
- **eframe 0.36 with egui and eframe's wgpu backend, on by default in 0.36.2, and walkers 0.61 with its `mvt` feature**: the window, and the map in it ([0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)).
- **The `pmtiles` and `mvt-reader` crates, at the releases walkers 0.61 depends on**: reading the archive's tiles ([0005](../decisions/0005-features-as-vector-tiles-tylertoo-builds.md)).
- **Other Rust crates** the program itself depends on, chosen while building, for these purposes only, with crates that only its tests use left to the building session:
  - parsing the commands' arguments,
  - reading osmium-tool's export and the rules of R011, and writing the GeoParquet tylertoo reads,
  - finding which feature lies under a point, for the selection of R004,
  - and finding a font on the machine for the names of R010.
- **A software rasterizer on CI's runner**, for tests only: what the test that renders the window draws with there (A012), installed by the CI job, such as Mesa's Vulkan driver.
  - It is tried on CI, as R009's best effort there, since a test left to a developer's machine alone leaves CI no longer checking what the window draws, such as the features while the base map's service does not answer.
- **OpenFreeMap**: the base map's style and tiles, requested by the window while it is open.

## Done when

- **The map's data holds the features R002 says are drawn, and no other** — verifies R001, R002, A002, A013, A017, A018.
  - Each such feature is there once in every zoom 16 tile it reaches, with its type, id, version, and timestamp, and once among the features the program reads from the archive, its pieces joined by type and id.
  - A feature R002 says is not drawn is absent.
  - Further out than zoom 16, each feature a tile holds is drawn from its own object alone, even in a tile that went over the size limit.
  - Every feature written but missing from zoom 16 is reported by the command that builds the archive, and no other is.
  - Check: an end-to-end test runs the command that builds the archive on a hand-written extract and three of its municipalities, reads the archive as the window would, without opening one, and compares what it holds with what the extract was written to hold, 0 standing for a version or a timestamp the extract left out.
    - Two of the municipalities share an edge, and one lies apart from them.
    - For each rule of R001, R002, and A002 the test verifies, the extract puts features on each side of it, these among them:
      - a tagged object inside each municipality, the one apart included, and one with every node outside the area, in the gap between them;
      - a tagged way and a multipolygon relation with nodes in both municipalities that share the edge, each coming out once;
      - a tagged way with one node inside the area and one outside it, coming out whole;
      - a boundary relation S1, with ways and nodes of its own, that is a member of one municipality's relation and lies inside that municipality, and a boundary relation S2 that is a member of S1 and lies inside S1, with a tagged object inside S2, one inside S1 but outside S2, and one inside the municipality but outside S1;
      - a closed way tagged as a line, one tagged as an area, one with tags of both, such as a fence around a field, and one with tags of neither, such as a part of a building, each coming out once;
      - two lines of the same kind that touch end to end, long enough that joined they would reach more than a pixel past either at the zooms checked;
      - a tagged way with its two nodes at one position;
      - an area of about 1 cm on a side;
      - and a node, a way, and a multipolygon relation written without a version or a timestamp.
    - It also reads the command's report and compares it with the features written that zoom 16 lacks, among them the area of about 1 cm for as long as tylertoo drops it (0005), and fails where zoom 16 lacks none, since the report would then go unchecked.
    - At every zoom further out than 16, it finds each feature the touching lines' tiles hold carrying one object's id and lying within a pixel of that zoom of that object; and it builds the extract again with a size limit small enough that a tile further out goes over it, confirms from the build's report that one did, and finds the same of each feature that tile holds.
- **A change to the rules changes which features the same build of the program keeps** — verifies R011.
  - Check: the same test with a second set of rules, one added and one removed, compares the features that come out.
- **A list with no municipality, one the extract does not hold, a name several boundaries share, or municipalities that overlap, is refused** — verifies R001, R007.
  - Check: a test runs the command that builds the archive each of those ways on the hand-written extract and reads the message.
- **The window shows each feature by its last-edit date and leads from it to its object, with the data's date and OpenStreetMap's credit, and draws the features without the base map** — verifies R001, R002, R003, R004, R005, R006, R010, A005, A011, A012, A015.
  - The window opens showing the archive's whole area.
  - At zoom 16 and closer, every feature zoom 16 holds is drawn.
  - Further out than zoom 10, no feature is drawn, and the window says that zooming in shows them.
  - Built from municipalities too far apart to fit the window at zoom 10, the window opens at zoom 10 over the area's centre.
  - Points, lines, and areas are coloured by last-edit date on one scale, and a legend says which date each colour stands for.
  - A selected point, line, and area each show their type, id, version, and last-edit date, and a link to that object on openstreetmap.org.
  - Selecting inside a small area that lies within a larger one, across a tile's edge, selects the small one.
  - Selecting where two areas overlap, neither within the other, at a point where the tile holds less of the larger one, selects the one smaller as a whole.
  - Selecting where a point or a line lies over an area selects the point or the line.
  - Selecting near two lines, near two points, and near a point and a line, each pair at different distances and the nearer written after the farther in the extract, selects the nearer of each pair.
  - The window states the date of the data.
  - Built from an extract whose header gives no date, the window says that the data's date is unknown.
  - A feature whose object has no `timestamp` is drawn grey, and the legend gives grey as an unknown date.
  - Selecting a feature whose object has no `version` or `timestamp` shows each it lacks as unknown.
  - The credit to OpenStreetMap is in a corner with its link, before and after panning.
  - While the base map's service does not answer, the features are still drawn.
  - Given no font for Japanese, the window still opens, and says that some names cannot be drawn.
  - Check: automated tests on the fixture's map.
    - One compares the colour of every feature in what the program hands walkers with the colour the legend gives for that feature's date, which is the same for a point, a line, and an area of one date.
    - One renders the window to an image while the base map's service does not answer (A012), and finds each feature drawn in that colour.
    - The others drive the window without showing it (A011), and work out each date, detail, and link from what the fixture was written to hold.
    - They wait on no host other than the machine they run on.
- **To the eye, the stand-in area's window reads and answers over the base map** — verifies R003, R004, R006, R010, A008, A009, A010.
  - The stand-in area's features are coloured by last-edit date, with the legend, and those longest without an edit can be told from the rest.
  - The base map shows under the features in muted colours, with its place and street names above them, in Japanese.
  - The base map's own credit is beside OpenStreetMap's, before and after panning.
  - A selected point, line, and area each show their details there as well, and the link of each opens that object on openstreetmap.org.
  - Check: open the stand-in area's window and look, around Tokyo Station.
    - Select one feature of each kind and follow its link.
- **The stand-in area builds, opens, and answers within R008's times, from an extract that dates its objects and itself, with no feature missing unaccounted for** — verifies R001, R002, R005, R008, A001, A004, A005, A007, A013, A016, A017.
  - The Kanto extract has a `version` and a `timestamp` on every object, and its header states the date of its data.
  - The archive is built, and the window shows the features of its opening view, each within R008's time of its command starting.
  - Over the opening view and around Tokyo Station, while the map is panned and zoomed, the interval between frames keeps within R008's.
  - The window states the date the extract's header gives.
  - R002 and the rules account for every tagged object that goes into the build and is not among the features the archive's zoom 16 tiles hold.
  - The features the build reports missing from zoom 16 are counted by type and recorded by id (R002), and before the phase closes they are read, and either kept as a limit or met by looking at tylertoo's options for one that keeps them (A013).
  - Check: on a developer's machine, with the window at R008's size, and with the machine and the number of features recorded.
    - `osmium fileinfo -e` on the Kanto extract prints `version+timestamp` for all objects and a replication timestamp.
    - Compare the date in the window with that replication timestamp.
    - Time the command that builds the archive, and the other until the features of its opening view show, and record the interval between frames while the program itself pans and zooms the map without a pause, once over the opening view and once around Tokyo Station, as A004's trial did, since egui draws no frame while nothing moves and a pause by hand would count as a long one.
    - Count the tagged objects that go into the build and the features the archive's zoom 16 tiles hold, and record what accounts for the difference.
- **Building and opening a map leaves nothing in the repository that tells where a developer works** — verifies R007.
  - The working tree is clean after the stand-in area's archive has been built and opened.
  - No tracked file holds a coordinate other than the fixture's and the stand-in area's, or names a real municipality, the stand-in's included, whose list is given from outside the tracked files as any other area's is.
  - Check: `git status --short` prints nothing after building and opening the stand-in area, and the tracked files are searched for coordinates and for the names and boundary ids of the municipalities built.
- **CI passes with the program's automated checks among its jobs** — verifies R009, A006, A011, A012, A014.
  - The Rust checks and the end-to-end test pass there.
  - The tests of the window pass there as well, or on a developer's machine alone where running them in CI proves not to work.
  - Check: the pull request's CI run, on a runner that installed osmium-tool and tylertoo through `mise install`, and for tests left to a developer's machine, a run of them there.

## Open questions

- Whether the colour scale is fixed dates or follows the dates found in the area: decided while building, looking at the stand-in area.
