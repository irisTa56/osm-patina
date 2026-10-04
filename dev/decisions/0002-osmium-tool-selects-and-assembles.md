# 0002. osmium-tool selects the area and assembles geometry

- Status: Proposed
- Date: 2026-10-04

## Context

Between the extract of [0001](0001-regional-extract-as-data-source.md) and features a page can draw lie three steps: selecting the objects of the rectangle together with what they refer to, assembling ways and multipolygon relations into lines and polygons, and attaching each object's `version` and `timestamp`.
The program is written in Rust ([0003](0003-rust-for-the-program.md)).

- [osmium-tool](https://osmcode.org/osmium-tool/) does all three from the command line: [`osmium extract`](https://docs.osmcode.org/osmium/latest/osmium-extract.html) selects by bounding box, and [`osmium export`](https://docs.osmcode.org/osmium/latest/osmium-export.html) assembles geometry and writes GeoJSON with each object's type, id, version, and timestamp (manuals checked 2026-10-03; run on a hand-written file, 2026-10-03).
  - For the largest rectangle the two took about 5 seconds on the 516 MB Kanto extract, 4 to 5 to cut and under 1 to export between 100,000 and 160,000 features (osmium 1.19.1, Apple M3, runs of 2026-10-04).
  - It is under GPL-3.0-or-later and publishes no binary; [conda-forge](https://anaconda.org/conda-forge/osmium-tool) packages its current release, 1.19.1, which mise's conda backend installed on macOS arm64 (checked and run 2026-10-03).
  - What it selects and writes is not exactly the features in the rectangle: it leaves some out and brings some in from outside, without an error (runs of 2026-10-03 and 2026-10-04; the [plan](../plan/phase-01-local-map.md#requirements--constraints) lists the cases found under R002).
- No maintained Rust crate was found that offers the assembly of OSM geometry as a library (each repository or README, checked 2026-10-03 and 2026-10-04).
  - [`osm_boundaries_utils`](https://github.com/Qwant/osm_boundaries_utils_rs) is archived, [`libosmium`](https://github.com/gammelalf/libosmium), a binding to the C++ library, had its last commit on 2023-10-21, and [`osmdb-extract`](https://docs.rs/crate/osmdb-extract/0.2.2) lists "Polygon and multipolygon assembly" among what it has yet to do.
  - [`elivagar`](https://github.com/folknor/elivagar) 0.1.0, a tool that turns PBF into vector tiles, joins member ways into rings and pairs inner rings with outer ones in a module it keeps private (`src/multipolygon.rs`, Apache-2.0).
  - [`osmpbf`](https://docs.rs/osmpbf/0.3.8/osmpbf/) 0.3.8 reads PBF and exposes `version` and `timestamp` on every element, but stops at the elements.
- [pyosmium](https://docs.osmcode.org/pyosmium/latest/user_manual/03-Working-with-Geometries/) 4.3.1 wraps the same C++ library for Python, assembling areas with `with_areas()` and writing GeoJSON, in one process (checked 2026-10-03).

## Decision

The Rust program runs `osmium extract` and `osmium export` as separate processes and reads the GeoJSON they write.
It takes the features through one boundary, a stream of geometries each with its type, id, version, and timestamp, so that what produces the stream can be replaced.
The Rust code owns what comes after: which of those features the map keeps, and what freshness each one shows.
The options the two commands run with are settled while building.

## Rejected alternatives

- **Assembling geometry in Rust on `osmpbf` now**: joining rings, assigning inner rings to outer ones, and handling broken relations would be the largest piece of the program and the likeliest to be wrong, though `elivagar`'s module is there to read, and it would stand between this phase and a map to look at.
  - The maintainer means to write it as a crate later ([0003](0003-rust-for-the-program.md)); `osmium export`'s output on the same input is then what that crate is tested against.
- **The `libosmium` crate**: it reaches the same assembler without a second process, but through a binding nobody maintains and a C++ build.
- **pyosmium**: it would do all three steps in one process and add no command-line tool, but it means a program in Python, which [0003](0003-rust-for-the-program.md) turns down.

## Consequences

- **Dependencies added**: osmium-tool, installed by mise's [conda backend](https://mise.jdx.dev/dev-tools/backends/conda.html) and run as a separate program rather than linked.
- **Risks**:
  - It leaves features out and brings others in without an error, and not every such case is known, which would show as a difference between the objects that go in and the features that come out.
  - A new osmium-tool release could change its output, which the end-to-end test on the fixture would show as a failure.
  - conda-forge has no `linux-aarch64` build of osmium-tool, so `mise install` would fail on such a machine.
  - What `osmium export` writes carries each feature's own `timestamp` and not its nodes', so work that needs the nodes' has to read the cut PBF as well, which `osmpbf` allows.
