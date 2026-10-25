# 0003. The program is written in Rust

- Status: Proposed
- Date: 2026-10-04

## Context

The program turns OSM data into what the map draws.
The language is chosen for three reasons: a preference for Rust; a preference for a fast finished pipeline; and an openness to writing in Rust, as a crate, what pyosmium, the Python library below, does today.

- [`osmpbf`](https://docs.rs/osmpbf/0.3.8/osmpbf/) 0.3.8 reads PBF in Rust and exposes `version` and `timestamp` on every element (checked 2026-10-03).
- No maintained Rust crate was found that offers the assembly of ways and multipolygon relations into geometry as a library ([0002](0002-osmium-tool-selects-and-assembles.md)).
- [pyosmium](https://docs.osmcode.org/pyosmium/latest/user_manual/03-Working-with-Geometries/) 4.3.1 reads PBF in Python with the same fields, assembles areas with `with_areas()`, and writes GeoJSON, all in one process (checked 2026-10-03).
- The one PBF parser for Elixir that searches of Hex for `pbf`, `osm`, `osmpbf`, and `openstreetmap` returned, [`pbf_parser`](https://hex.pm/packages/pbf_parser), was last updated on 2018-09-10 (searched 2026-10-11), and the two others found on GitHub in 2017 (checked 2026-10-04).

## Decision

The program is written in Rust.
Until a Rust crate assembles geometry, it leaves that step to osmium-tool ([0002](0002-osmium-tool-selects-and-assembles.md)).

## Rejected alternatives

- **Python with pyosmium**: it would be less code and one dependency fewer, since it reads and assembles in one process, but it gives up the preference for Rust and the openness to a Rust crate above.
- **A shell pipeline with no program for this phase**: `osmium extract`, `osmium tags-filter`, and `osmium export` into a static page would draw the map, and the program would be written when a later phase needs one; but what this phase has the program do, dropping features outside the rectangle and reading its rules from a file, would be written once in shell and again in Rust.
- **Elixir**: the three PBF parsers found for it have not been maintained since 2018.

## Consequences

- **Dependencies added**: the Rust toolchain.
- **Risks**:
  - Geometry assembly stays outside the program for as long as no crate does it, so osmium-tool has to be installed wherever the program runs.
  - Writing such a crate would be a project of its own, and until one exists the pipeline's speed is bounded by the two osmium-tool processes, which took about 5 seconds for the largest area ([0002](0002-osmium-tool-selects-and-assembles.md)).
