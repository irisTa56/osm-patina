# Phase 01: Local map for one area

- Status: In progress
- Roadmap: [Local map for one area](../ROADMAP.md#phases)

## Goal

On a developer's machine, one command builds the features of a chosen rectangle of a regional extract into vector tiles, and another opens them in a window where every feature in scope is coloured by the date of its last edit, over a base map.
A mapper looking at the window sees which features have gone longest without one, and can open any of them on openstreetmap.org.

## Requirements & Constraints

- **R001**: Given a regional extract on disk, a centre point, and a width and height, one command builds the map of that rectangle as an archive of vector tiles, and given that archive, another command opens a window over the rectangle at zoom 14 ([0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)).
  - A rectangle whose long side exceeds 5 km is refused, with a message that says so.
- **R002**: The map draws the features in the rectangle that the [roadmap's scope](../ROADMAP.md#scope) covers, each one once, at zoom 14 and closer ([0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)).
  - Further out than zoom 14, it draws no feature, and the window says that zooming in shows them.
  - A feature is built from a tagged node, a tagged way, or a relation of type multipolygon or boundary.
    - A relation of any other type is not a feature.
  - A feature is in the rectangle when one of its nodes is, and it is then drawn whole, past the rectangle's edge.
    - One with every node outside is not drawn.
  - A feature outside that scope is not drawn, such as an administrative boundary or a feature tagged as indoor.
  - A feature osmium-tool does not produce for the rectangle is not drawn, which the maintainer accepted as a limit of this phase on 2026-10-04.
    - The cases known are a feature with no node inside the rectangle, a boundary relation that reaches past its edge, and a geometry that cannot be assembled (runs of osmium 1.19.1, 2026-10-03 and 2026-10-04).
- **R003**: Each feature is coloured by the date of its last edit, on one scale shared by points, lines, and areas, and a legend says which date each colour stands for.
  - The date is the `timestamp` of the OSM object the feature was built from.
- **R004**: Selecting a feature shows its type, id, version, and last-edit date, and a link to that object on openstreetmap.org.
- **R005**: The window states the date of the data it shows, since the data is a day or more behind OSM ([0001](../decisions/0001-regional-extract-as-data-source.md)), and an archive is as old as the extract it was built from.
- **R006**: The map credits OpenStreetMap in one of its corners with "© OpenStreetMap contributors", linked to `https://www.openstreetmap.org/copyright`, beside the base map's own credit, and the credits never collapse.
  - The [Attribution Guidelines](https://osmfoundation.org/wiki/Licence/Attribution_Guidelines) allow a credit to collapse on interaction (checked 2026-10-03), but one that stays needs no reading of those conditions.
- **R007**: Nothing the phase commits reveals where a developer works, as [CLAUDE.md](../../CLAUDE.md#keeping-a-developers-location-private) requires.
  - The centre point comes from outside the tracked files, and no tracked file gives it a default.
  - Everything the commands write goes where git ignores it, since it is cut from the developer's area.
- **R008**: At the stand-in area, a 5 km square of about 108,000 features, on a developer's machine, the commands and the window keep to these times:
  - The archive is built within 12 seconds of its command starting, from an extract of about 500 MB.
  - Within 1 second of the other command starting, the window shows every feature in its opening view.
  - Once they show, the interval between frames while the map is panned and zoomed is within 20 ms at the 95th percentile.
- **R009**: The program is written in Rust ([0003](../decisions/0003-rust-for-the-program.md)) and draws the map itself, in no page or browser and with no JavaScript ([0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)), and its automated checks run in `mise run pre-commit` and in CI.
  - The tests that drive the window are meant to run in CI as well, and where that proves not to work, the maintainer accepts running them on a developer's machine alone (said 2026-10-11 of tests that drove a page, and carried over to the window on 2026-10-14).
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
- **Picking the point on the map or from the current location**: the Location phase.
- **Downloading or updating the extract**: the developer downloads it and passes its path, which keeps this phase to the map.
- **Assembling geometry in Rust**: not in this phase ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)).
- **Opening the map anywhere but a developer's machine**: the Publishing phase.
- **Building tiles for an area wider than the rectangle, once for every map opened in it**: how long it takes and how large the tiles are is not measured ([0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)).

## Assumptions & Risks

- **A001**: A Geofabrik extract keeps every object's `version` and `timestamp`. Source: `osmium fileinfo -e` on the Kanto extract, run 2026-10-04 ([0001](../decisions/0001-regional-extract-as-data-source.md)).
  - Risk: features come out without a date, noticed by `osmium fileinfo -e` not printing `version+timestamp` for all objects.
- **A002**: `osmium export` writes each feature's type, id, version, and timestamp. Source: a run of osmium 1.19.1 on a hand-written file, 2026-10-03.
  - Risk: the map has no number to colour by, noticed by the end-to-end test.
- **A003**: The densest of the four 5 km squares counted holds about 158,000 features before the scope narrows them, half as many again as the stand-in area, which R008 is stated for. Source: counts of the four squares.
  - The stand-in area held about 108,000 and a denser square in the same extract about 158,000 on 2026-10-04 (`osmium export` of each cut).
  - A 5 km square in central London held about 151,000 on 2026-10-03, and another city centre about 92,000 on 2026-09-28 (Overpass counts by key).
  - Risk: an area denser than the stand-in, such as the densest square counted, answers panning and zooming more slowly than R008's interval, which no check measures, since the maintainer chose not to hold such an area to R008 (said 2026-10-14); noticed by the map lagging when one is opened.
- **A004**: walkers draws the stand-in area's features within R008's frame interval. Source: the trials in [0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md) and [0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md), 108,074 features over OpenFreeMap's vector tiles while the map zoomed and moved by itself, walkers 0.61.0 and eframe 0.36.2 on an Apple M3, four runs from GeoJSON on 2026-10-13 and 2026-10-14 and two from the zoom 14 archive 0005 decides on 2026-10-15.
  - The 95th percentile between frames was 17.3 to 17.9 ms, the median 16.7 ms, which is the display's own rate.
  - The window's own work in a frame had a median of 0.15 ms, and of 4.2 to 4.6 ms once the base map's names were drawn in Japanese.
  - From the archive, the first frame came 127 to 377 ms after the program started, and the 30 tiles it needed were read in 115 ms.
  - Not measured: A003's denser square.
  - Risk: a slower machine, or the program's own work on top of the trial's, misses R008, noticed by the scale check.
- **A005**: The extract's header states the date of its data. Source: the Kanto extract's header carries `osmosis_replication_timestamp=2026-10-03T20:20:50Z` (`osmium fileinfo`, run 2026-10-04).
  - A file written by hand carries the same header once osmium 1.19.1 converts it to PBF with `--output-header` (run 2026-10-11), so a fixture can state a date as an extract does.
  - Risk: another extract lacks it and R005 has nothing to show, noticed by `osmium fileinfo` printing no replication timestamp.
    - The file's own date would be the fallback.
- **A006**: mise's conda backend installs osmium-tool on CI's Linux runner as it does on macOS arm64. Unverified on Linux; [conda-forge](https://anaconda.org/conda-forge/osmium-tool) lists a `linux-64` build (checked 2026-10-03).
  - Risk: the end-to-end test cannot run in CI, noticed by the job failing at install.
- **A007**: Cutting and exporting the largest rectangle takes about 5 seconds from an extract of about 500 MB. Source: [0002](../decisions/0002-osmium-tool-selects-and-assembles.md), run 2026-10-04.
  - The cut reads the whole extract, so the time grows with the extract and not with the rectangle.
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
- **A013**: tippecanoe builds the stand-in area's features into an archive that holds every one of them at zoom 14, within about 3 seconds. Source: the trial in [0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md), tippecanoe 2.79.0 with `-pS -pt -d16` on an Apple M3, 2026-10-15: 2.8 seconds and 2.9 MB for zoom 14, counted with `tippecanoe-decode`.
  - Risk: in a denser area a zoom 14 tile goes over tippecanoe's default limit of 500 KB, and the build fails, as the zoom 13 tile over Tokyo Station did, noticed only when such an area is built; lifting the limit is the way out.
- **A014**: mise's conda backend installs tippecanoe on CI's Linux runner as it does on macOS arm64. Source: it installed 2.79.0 on macOS arm64 (2026-10-15). Unverified on Linux.
  - Risk: the end-to-end test cannot run in CI, noticed by the job failing at install.
- **A015**: The feature under a point can be found from the tile under it, within a millisecond. Source: 0.36 to 0.44 ms at one point in the trial in [0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md).
  - A tile holds each area cut at its edge, so the smallest of several areas under a point is judged on the cut ones unless the archive carries each feature's whole size.
  - Risk: the larger of two overlapping features is selected, noticed by the test that selects within overlapping areas.
- **A016**: The command that builds the archive keeps to R008's time. Source: its parts as measured on an Apple M3, at most about 6 seconds to cut and export (A007) and 2.8 seconds for the tiles (A013).
  - Not measured: keeping the features the rules admit, writing what tippecanoe reads, and the whole command in one run.
  - Risk: those parts take more than the 3 seconds or so left, noticed by the scale check.

## Decisions

- [0001. A regional extract kept on disk is the data source](../decisions/0001-regional-extract-as-data-source.md)
- [0002. osmium-tool selects the area and assembles geometry](../decisions/0002-osmium-tool-selects-and-assembles.md)
- [0003. The program is written in Rust](../decisions/0003-rust-for-the-program.md)
- [0004. The map is drawn in Rust, in a native window](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)
- [0005. The features reach the window as vector tiles that tippecanoe builds](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)
- The Rust toolchain, osmium-tool, and tippecanoe at their latest releases rather than at fixed versions, because lychee and rumdl are declared that way, and CI installs the current release on each run, the weekly one included, so a release the checks fail on shows there.
  - The maintainer chose it for the Rust toolchain and osmium-tool (said 2026-10-13), and for tippecanoe (said 2026-10-15).
- One command that builds the archive and another that opens it, rather than one that does both, because an archive once built opens in under a second (A004) without cutting the extract again.
  - The maintainer chose it, as closer to what the project aims at (said 2026-10-15), over the one command they had chosen on 2026-10-14, when nothing stood between cutting and drawing that could be kept.
- OpenFreeMap's `positron` style as the base map rather than OpenStreetMap's standard tiles or a self-hosted Protomaps archive, because it is grey, so the colours of R003 keep their meaning over it, it needs no key and no server, and it is a vector style, so its labels can sit above the features.
  - The standard tiles are coloured by kind of feature and are images, so their names cannot sit above the features (checked 2026-10-03).
  - A Protomaps archive is a file the developer downloads, cut with `pmtiles extract` from a planet build of about 120 GB, and keeps up to date, where OpenFreeMap's tiles are requested as the window needs them ([its downloads page](https://docs.protomaps.com/basemaps/downloads), read 2026-10-15).
- Names in their own script rather than in Latin script alone, because a mapper in Japan reads names best in their local script, and a font for it is on the developer's macOS (A010).
  - The maintainer chose it (said 2026-10-14).
- An allow-list of tag keys rather than a deny-list, because `osmium export` turns a boundary relation into a polygon over everything inside it (run of 2026-10-03), and a key nobody thought to deny would do the same.
  - A feature the list admits is still left out when it is tagged as indoor, since a shop inside a building carries the same `shop` key as one on the street.
  - Which keys stand for each of the scope's kinds of feature is settled while building.
- What a test can check is checked that way rather than by eye, because the test goes on guarding the behaviour after the phase closes.
  - Three things are left to the eye:
    - what only an eye judges, that the base map is muted and its names can be read above the features;
    - what shows only while the base map's service answers, such as its own credit and its names, since a test that waited on that service would fail whenever it did not answer;
    - and what needs the stand-in area's data, which is not in the repository.
- Test fixtures are OSM files written by hand rather than cut from a real area, because data cut from OSM would bring its licence into the repository.
- The scale check and the screenshots use a stand-in area the maintainer picked for the purpose, a 5 km square centred on Tokyo Station, cut from Geofabrik's [Kanto extract](https://download.geofabrik.de/asia/japan/kanto.html), because they need real data and R007 rules out a developer's own point.

## Dependencies

- **osmium-tool**: selecting the rectangle and assembling geometry ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)), declared in `mise.toml` at its latest release, which was 1.19.1 on 2026-10-13, and installed by `mise install` through the conda backend.
- **tippecanoe**: building the rectangle's features into a PMTiles archive ([0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)), declared in `mise.toml` at its latest release, which was 2.79.0 on 2026-10-15, and installed by `mise install` through the conda backend.
- **Rust toolchain**: building and testing the program, declared in `mise.toml` at its latest release.
- **eframe 0.36 with egui, and walkers 0.61 with its `mvt` feature**: the window, and the map in it ([0004](../decisions/0004-map-drawn-in-rust-in-a-native-window.md)).
- **The `pmtiles` and `mvt-reader` crates, at the releases walkers 0.61 depends on**: reading the archive's tiles ([0005](../decisions/0005-features-as-vector-tiles-tippecanoe-builds.md)).
- **Other Rust crates** the program itself depends on, chosen while building, for these purposes only, with crates that only its tests use left to the building session:
  - parsing the commands' arguments,
  - reading osmium-tool's export and the rules of R011, and writing what tippecanoe reads,
  - finding which feature lies under a point, for the selection of R004,
  - and finding a font on the machine for the names of R010.
- **A software rasterizer on CI's runner**, for tests only: what the test that renders the window draws with there (A012), installed by the CI job, such as Mesa's Vulkan driver.
  - The maintainer chose to try it on CI first (said 2026-10-14).
- **OpenFreeMap**: the base map's style and tiles, requested by the window while it is open.

## Done when

- **The map's data holds the features R002 says are drawn, and no other** — verifies R001, R002, A002, A013.
  - Each such feature is there once in every zoom 14 tile it reaches, with its type, id, version, and timestamp, and once among the features the program reads from the archive, its pieces joined by type and id.
  - A feature R002 says is not drawn is absent.
  - Check: an end-to-end test runs the command that builds the archive on a hand-written extract, reads the archive as the window would, without opening one, and compares what it holds with what the extract was written to hold.
- **A change to the rules changes which features the same build of the program keeps** — verifies R011.
  - Check: the same test with a second set of rules, one added and one removed, compares the features that come out.
- **A rectangle with a long side over 5 km, or a command without a centre point, is refused** — verifies R001, R007.
  - Check: a test runs the command that builds the archive both ways and reads the message.
- **The window shows each feature by its last-edit date and leads from it to its object, with the data's date and OpenStreetMap's credit, and draws the features without the base map** — verifies R001, R002, R003, R004, R005, R006, R010, A005, A011, A012, A015.
  - The window opens over the rectangle at zoom 14.
  - Points, lines, and areas are coloured by last-edit date on one scale, and a legend says which date each colour stands for.
  - A selected point, line, and area each show their type, id, version, and last-edit date, and a link to that object on openstreetmap.org.
  - Selecting inside a small area that lies within a larger one, across a tile's edge, selects the small one.
  - Further out than zoom 14, no feature is drawn, and the window says that zooming in shows them.
  - The window states the date of the data.
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
  - Check: open the stand-in area's window and look.
    - Select one feature of each kind and follow its link.
- **The stand-in area builds, opens, and answers within R008's times, from an extract that dates its objects and itself, with no feature missing unaccounted for** — verifies R001, R002, R005, R008, A001, A004, A005, A007, A013, A016.
  - The Kanto extract has a `version` and a `timestamp` on every object, and its header states the date of its data.
  - The archive is built, and the window shows every feature in its opening view, each within R008's time of its command starting.
  - While the map is panned and zoomed, the interval between frames keeps within R008's.
  - The window states the date the extract's header gives.
  - R002 and the rules account for every tagged object that goes into the build and is not among the features that come out.
  - Check: on a developer's machine, with the machine and the number of features recorded.
    - `osmium fileinfo -e` on the Kanto extract prints `version+timestamp` for all objects and a replication timestamp.
    - Compare the date in the window with that replication timestamp.
    - Time the command that builds the archive, and the other until every feature in the opening view shows, and record the interval between frames while the program itself pans and zooms the map without a pause, as A004's trial did, since egui draws no frame while nothing moves and a pause by hand would count as a long one.
    - Count the tagged objects that go into the build and the features the archive's zoom 14 tiles hold, and record what accounts for the difference.
- **Building and opening a map leaves nothing in the repository that tells where a developer works** — verifies R007.
  - The working tree is clean after the stand-in area's archive has been built and opened.
  - No tracked file holds a coordinate other than the fixture's and the stand-in area's.
  - Check: `git status --short` prints nothing after building and opening the stand-in area, and the tracked files are searched for coordinates.
- **CI passes with the program's automated checks among its jobs** — verifies R009, A006, A011, A012, A014.
  - The Rust checks and the end-to-end test pass there.
  - The tests of the window pass there as well, or on a developer's machine alone where running them in CI proves not to work.
  - Check: the pull request's CI run, on a runner that installed osmium-tool and tippecanoe through `mise install`, and for tests left to a developer's machine, a run of them there.

## Open questions

- Whether the colour scale is fixed dates or follows the dates found in the area: decided while building, looking at the stand-in area.
