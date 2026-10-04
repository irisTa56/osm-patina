# Phase 01: Local map for one area

- Status: In progress
- Roadmap: [Local map for one area](../ROADMAP.md#phases)

## Goal

On a developer's machine, one command turns a regional extract and a chosen rectangle into a page where every feature in scope is coloured by the date of its last edit, over a base map.
A mapper looking at the page sees which features have gone longest without one, and can open any of them on openstreetmap.org.

## Requirements & Constraints

- **R001**: Given a regional extract on disk, a centre point, and a width and height, one command builds the map of that rectangle and says how to open it.
  - A rectangle whose long side exceeds 5 km is refused, with a message that says so.
- **R002**: The page draws the features in the rectangle that the [roadmap's scope](../ROADMAP.md#scope) covers, each one once.
  - A feature is built from a tagged node, a tagged way, or a relation of type multipolygon or boundary; a relation of any other type is not a feature.
  - A feature is in the rectangle when one of its nodes is, and it is then drawn whole, past the rectangle's edge; one with every node outside is not drawn.
  - A feature outside that scope is not drawn, such as an administrative boundary or a feature tagged as indoor.
  - A feature osmium-tool does not produce for the rectangle is not drawn, which the maintainer accepted as a limit of this phase on 2026-10-04.
    - The cases known are a feature with no node inside the rectangle, a boundary relation that reaches past its edge, and a geometry that cannot be assembled (runs of osmium 1.19.1, 2026-10-03 and 2026-10-04).
- **R003**: Each feature is coloured by the date of its last edit, on one scale shared by points, lines, and areas, and a legend says which date each colour stands for.
  - The date is the `timestamp` of the OSM object the feature was built from.
- **R004**: Selecting a feature shows its type, id, version, and last-edit date, and a link to that object on openstreetmap.org.
- **R005**: The page states the date of the data it shows, since the data is a day or more behind OSM ([0001](../decisions/0001-regional-extract-as-data-source.md)).
- **R006**: The map credits OpenStreetMap in one of its corners with "© OpenStreetMap contributors", linked to `https://www.openstreetmap.org/copyright`, beside the base map's own credit, and the credits never collapse.
  - The [Attribution Guidelines](https://osmfoundation.org/wiki/Licence/Attribution_Guidelines) allow a credit to collapse on interaction (checked 2026-10-03), but one that stays needs no reading of those conditions.
- **R007**: Nothing the phase commits reveals where a developer works, as [CLAUDE.md](../../CLAUDE.md#keeping-a-developers-location-private) requires.
  - The centre point comes from outside the tracked files, and no tracked file gives it a default.
  - Everything the command writes goes where git ignores it, since it is cut from the developer's area.
- **R008**: At the largest rectangle in a dense city, on a developer's machine, the command and the page keep to these times:
  - the command finishes within 15 seconds from an extract of about 500 MB;
  - once the base map shows, the first kind of feature appears within 3 seconds and all of them within 5 seconds;
  - and the map answers panning and zooming while the features load.
- **R009**: The program is written in Rust ([0003](../decisions/0003-rust-for-the-program.md)), and its automated checks run in `mise run pre-commit` and in CI.
- **R010**: The features are drawn over a base map in muted colours, with its place and street names above them.
  - When the base map's service does not answer, the features are still drawn.
  - The base map's data has its own date, so a recent building can be on one layer and not the other.
- **R011**: The rules that bring a feature into scope or take it out are data in the repository, so that a rule is added or replaced without changing the program.

### Out of scope

- **Dating a feature by its newest part**: a way's `timestamp` does not change when only its nodes move, and a multipolygon's is its relation's own, so a reshaped feature shows an older date than its shape deserves.
  - Roland Olbricht's [Handling Timestamps in OpenStreetMap](https://dev.overpass-api.de/misc/timestamps.pdf) (2018) found that 20% to 30% of the ways changed in a period kept their version (checked 2026-10-03).
  - This goes to the Vertex-aware freshness phase.
- **Which attribute an edit touched**: freshness is per feature, so a shop whose opening hours changed last week shows as fresh; the roadmap has no phase for it.
- **Picking the point on the map or from the current location**: the Location phase.
- **Downloading or updating the extract**: the developer downloads it and passes its path, which keeps this phase to the map.
- **Assembling geometry in Rust**: a later piece of work ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)).
- **Vector tiles and a layout for phones**.

## Assumptions & Risks

- **A001**: A Geofabrik extract keeps every object's `version` and `timestamp`. Source: `osmium fileinfo -e` on the Kanto extract, run 2026-10-04 ([0001](../decisions/0001-regional-extract-as-data-source.md)).
  - Risk: features come out without a date, noticed by `osmium fileinfo -e` not printing `version+timestamp` for all objects.
- **A002**: `osmium export` writes each feature's type, id, version, and timestamp, the last as seconds since 1970. Source: a run of osmium 1.19.1 on a hand-written file, 2026-10-03.
  - Risk: the page has no number to colour by, noticed by the end-to-end test.
- **A003**: A dense 5 km square holds up to about 160,000 features before the scope narrows them. Source: the stand-in area held about 108,000 and a denser square in the same extract about 158,000 on 2026-10-04 (`osmium export` of each cut), a 5 km square in central London about 151,000 on 2026-10-03, and another city centre about 92,000 on 2026-09-28 (Overpass counts by key).
  - Risk: a denser area exceeds what A004 was measured on, noticed by its page missing R008's times.
- **A004**: MapLibre GL JS draws that many features from GeoJSON within R008 when they are split by kind. Source: the features of A003's denser square, 55 MB of GeoJSON in three files over the base map, MapLibre GL JS 6.11.2, Chromium on an Apple M3, two runs on 2026-10-04.
  - After the base map showed, the first kind appeared within 1.2 to 1.7 seconds and all three within 1.8 to 2.3 seconds; for the stand-in area's 41 MB they took 0.9 to 1.0 and 1.4 to 1.5 seconds.
  - The main thread's longest block was 0.6 seconds, while the base map loaded on the first run.
  - Risk: a slower machine, or the page's own work on top of this bare one, misses R008, noticed by the scale check; building vector tiles instead would be a change to this plan.
- **A005**: The extract's header states the date of its data. Source: the Kanto extract's header carries `osmosis_replication_timestamp=2026-10-03T20:20:50Z` (`osmium fileinfo`, run 2026-10-04).
  - Risk: another extract lacks it and R005 has nothing to show, noticed by `osmium fileinfo` printing no replication timestamp; the file's own date would be the fallback.
- **A006**: mise's conda backend installs osmium-tool on CI's Linux runner as it does on macOS arm64. Unverified on Linux; [conda-forge](https://anaconda.org/conda-forge/osmium-tool) lists a `linux-64` build (checked 2026-10-03).
  - Risk: the end-to-end test cannot run in CI, noticed by the job failing at install.
- **A007**: Cutting and exporting the largest rectangle takes about 5 seconds from an extract of about 500 MB. Source: [0002](../decisions/0002-osmium-tool-selects-and-assembles.md), run 2026-10-04.
  - The cut reads the whole extract, so the time grows with the extract and not with the rectangle; a larger extract has not been timed.
  - Risk: every build from an extract of several GB keeps a developer waiting, noticed by timing one; cutting a smaller extract once and building from that is the way out.
- **A008**: OpenFreeMap serves its `positron` style and tiles without a key or a limit on views. Source: [its site](https://openfreemap.org/) and a request for the style, 2026-10-04.
  - Its [terms](https://openfreemap.org/tos/) say the service may be discontinued at any time.
  - Risk: the base map disappears, noticed by features drawn on a blank page; another style URL replaces it.

## Decisions

- [0001. A regional extract kept on disk is the data source](../decisions/0001-regional-extract-as-data-source.md)
- [0002. osmium-tool selects the area and assembles geometry](../decisions/0002-osmium-tool-selects-and-assembles.md)
- [0003. The program is written in Rust](../decisions/0003-rust-for-the-program.md)
- GeoJSON in one file for each kind of geometry rather than vector tiles built with tippecanoe, because A004's measurement is within R008, and tiles would add tippecanoe and a server that answers range requests.
  - One file for each kind rather than one for all, because each kind then appears as soon as it is ready.
- OpenFreeMap's `positron` style as the base map rather than OpenStreetMap's standard tiles or a self-hosted Protomaps archive, because it is grey, so the colours of R003 keep their meaning over it, it needs no key and no server, and it is a vector style, so its labels can sit above the features.
  - The standard tiles are coloured by kind of feature and are images, so their names cannot sit above the features, and a Protomaps archive needs a server that answers range requests (checked 2026-10-03).
- An allow-list of tag keys rather than a deny-list, because `osmium export` turns a boundary relation into a polygon over everything inside it (run of 2026-10-03), and a key nobody thought to deny would do the same.
  - A feature the list admits is still left out when it is tagged as indoor, other than `indoor=no`, since a shop inside a building carries the same `shop` key as one on the street.
  - Which keys stand for each of the scope's kinds of feature is settled while building.
- Test fixtures are OSM files written by hand rather than cut from a real area, because data cut from OSM would bring its licence into the repository.
- The scale check and the screenshots use a stand-in area the maintainer picked for the purpose, a 5 km square centred on Tokyo Station, cut from Geofabrik's [Kanto extract](https://download.geofabrik.de/asia/japan/kanto.html), because they need real data and R007 rules out a developer's own point.

## Dependencies

- **osmium-tool 1.19**: selecting the rectangle and assembling geometry ([0002](../decisions/0002-osmium-tool-selects-and-assembles.md)), installed by `mise install` through the conda backend.
- **Rust toolchain**: building and testing the program, pinned in one place that `mise install` reads.
- **Rust crates**, chosen while building, for these purposes only:
  - parsing the command's arguments,
  - reading and writing JSON and the rules of R011,
  - and serving the page's files over HTTP on the local machine, if the page cannot fetch its data when opened from disk.
- **MapLibre GL JS 6.11** (BSD-3-Clause): drawing the map, pinned to an exact release.
- **OpenFreeMap**: the base map's style and tiles, requested by the page when it is viewed.

## Done when

- **Built from a fixture, the map's data holds each feature R002 says is drawn, once, with its type, id, version, and timestamp, and no other** — verifies R001, R002, A002.
  - Check: an end-to-end test runs the command on a hand-written extract and compares the result.
- **A change to the rules changes which features the same build of the program keeps** — verifies R011.
  - Check: a test runs the command on the fixture with a second set of rules, one added and one removed, and compares the result.
- **A rectangle with a long side over 5 km, or a command without a centre point, is refused** — verifies R001, R007.
  - Check: a test runs the command both ways and reads the message.
- **The page shows the stand-in area coloured by last-edit date with a legend, over the base map with its names above the features, and a selected feature shows its details and opens on openstreetmap.org** — verifies R003, R004, R006, R010, A008.
  - Check: build the stand-in area, open the page, select one point, one line, and one area, and follow each link; both credits are in a corner with their links, before and after panning; with requests to the base map's service blocked, the features are still drawn.
- **The stand-in area builds and shows within R008's times, with the date of the data on the page, and no feature is missing unaccounted for** — verifies R002, R005, R008, A001, A003, A004, A005, A007.
  - Check: `osmium fileinfo -e` on the Kanto extract prints `version+timestamp` for all objects and a replication timestamp; time the command, and on the page the first and the last kind of feature after the base map shows; pan while they load; record the machine and the number of features.
  - Check: count the tagged objects that go into the build and the features that come out, and record what accounts for the difference under R002 and the rules.
- **After a build, the working tree is clean** — verifies R007.
  - Check: `git status --short` prints nothing after building the stand-in area, and no tracked file holds a coordinate other than the fixture's and the stand-in area's.
- **CI passes with the Rust checks and the end-to-end test among its jobs** — verifies R009, A006.
  - Check: the pull request's CI run, on a runner that installed osmium-tool through `mise install`.

## Open questions

- Whether the colour scale is fixed dates or follows the dates found in the area: decided while building, looking at the stand-in area.
- Whether MapLibre GL JS is loaded from a CDN or kept in the repository.
- Whether the Rust toolchain is pinned in `mise.toml` or in `rust-toolchain.toml`.
