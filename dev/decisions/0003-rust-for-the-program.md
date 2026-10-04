# 0003. The program is written in Rust

- Status: Proposed
- Date: 2026-10-04

## Context

The program turns OSM data into what the map draws.
The maintainer chose the language on 2026-09-30 and gave the reasons on 2026-10-04: a preference for Rust, the speed of the finished pipeline, and the intention to write in Rust, as a crate, what the Python library does today.

- [`osmpbf`](https://docs.rs/osmpbf/0.3.8/osmpbf/) 0.3.8 reads PBF in Rust and exposes `version` and `timestamp` on every element (checked 2026-10-03).
- No maintained Rust crate was found that offers the assembly of ways and multipolygon relations into geometry as a library ([0002](0002-osmium-tool-selects-and-assembles.md)).
- [pyosmium](https://docs.osmcode.org/pyosmium/latest/user_manual/03-Working-with-Geometries/) 4.3.1 reads PBF in Python with the same fields, assembles areas with `with_areas()`, and writes GeoJSON, all in one process (checked 2026-10-03).
- The only PBF parser for Elixir on Hex, [`pbf_parser`](https://hex.pm/packages/pbf_parser), was last updated on 2018-09-10, and the two others found on GitHub in 2017 (checked 2026-10-04).

## Decision

The program is written in Rust.
Until a Rust crate assembles geometry, it leaves that step to osmium-tool ([0002](0002-osmium-tool-selects-and-assembles.md)).

## Rejected alternatives

- **Python with pyosmium**: it would be less code and one dependency fewer, since it reads and assembles in one process, but it gives up the three reasons above.
- **A shell pipeline with no program for this phase**: `osmium extract`, `osmium tags-filter`, and `osmium export` into a static page would draw the map, and the program would be written when a later phase needs one; but what this phase has the program do, dropping features outside the rectangle and reading its rules from a file, would be written once in shell and again in Rust.
- **Elixir**: its PBF parsers have not been maintained since 2018.

## Consequences

- **Dependencies added**: the Rust toolchain.
- **Risks**:
  - Geometry assembly stays outside the program for as long as no crate does it, so osmium-tool has to be installed wherever the program runs.
  - Writing that crate is a project of its own, and until it exists the speed the choice was made for is bounded by the two osmium-tool processes, which took about 5 seconds for the largest area ([0002](0002-osmium-tool-selects-and-assembles.md)).
