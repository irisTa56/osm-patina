# 0001. A regional extract kept on disk is the data source

- Status: Proposed
- Date: 2026-10-04

## Context

The map needs every feature in a rectangle up to 5 km on its long side, each with the `version` and `timestamp` of the OSM object it comes from.
Vertex-aware freshness, a later phase, dates a way by its newest node, so it needs the `timestamp` of untagged nodes as well.
A square of that size in a dense city centre holds between about 90,000 and 160,000 features ([phase 01 plan](../plan/phase-01-local-map.md#assumptions--risks), A003).

- Geofabrik's regional extracts keep both fields on every object and are rebuilt once a day.
  - Its [technical notes](https://download.geofabrik.de/technical.html) say the files "contain all data and metadata available in OSM for the region" except history, and that only "the user, uid and changeset fields are missing" (checked 2026-10-03).
  - `osmium fileinfo -e` on the Kanto extract printed "All objects have following metadata attributes: version+timestamp" (run 2026-10-04).
  - The Kanto extract is 516 MB (HEAD request, checked 2026-10-04).
  - Geofabrik asks users [not to fetch the same file again and again](https://blog.geofabrik.de/index.php/2025/09/10/download-responsibly/) (checked 2026-10-03).
- [SliceOSM](https://wiki.openstreetmap.org/wiki/SliceOSM) cuts a requested rectangle from minutely data through [an HTTP API](https://github.com/SliceOSM/sliceosm-api).
  - A request for a small box in central London returned in 0.36 seconds, with a timestamp on every way, relation, and tagged node, and none on its 2,383 untagged nodes (run 2026-10-04).
- The public Overpass API returns both fields with `out meta` ([output formats](https://dev.overpass-api.de/overpass-doc/en/targets/formats.html), checked 2026-10-03), and its data is minutes old.
  - Its main instance allows 1 GB a day for one-off use and a hundredth of that for regular use, and calls itself overloaded ([Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API), checked 2026-10-03).
  - A query for 11 buildings failed with HTTP 504 before it succeeded, at about 1.7 KB a way (run 2026-10-03), which puts the largest area at 150 to 270 MB a fetch before the nodes the ways are made of; a fetch of that size was not tried.
- The [ohsome API](https://docs.ohsome.org/ohsome-api/v2/migration_guide.html) requires an API key with every request, issued to an account (checked 2026-10-04).
- download.openstreetmap.fr serves daily regional extracts with all metadata, user names included, and splits Japan into the same eight parts as Geofabrik (its Monaco extract and directory listings, read 2026-10-04).

## Decision

The pipeline reads a regional extract in PBF format, downloaded from Geofabrik once and kept on disk, and cuts the chosen rectangle out of it locally.
Building the map for another rectangle inside the same extract makes no network request.

## Rejected alternatives

- **SliceOSM**: it would hand over the rectangle alone, minutes old, with no large download and nothing to cut, but its untagged nodes carry no timestamp, so Vertex-aware freshness would have to change source.
- **Overpass API with `out meta`**: a fetch of the largest area fits its one-off allowance but not its regular one, the instance is overloaded by its own account, and nothing shows that such a fetch completes.
  - Its fresher data does not outweigh that, since a map of features unedited for years loses little from being a day old.
- **ohsome API**: the map would work only on a machine that holds a key.
- **download.openstreetmap.fr**: it offers the same daily regional files as Geofabrik, with personal metadata the map has no use for.

## Consequences

- **Dependencies added**: none by this decision alone; reading the extract is [0002](0002-osmium-tool-selects-and-assembles.md).
- **Risks**:
  - The map lags OSM by a day or more, so a feature edited since then still shows its older date; the page states the date of its data so the lag is visible.
  - Geofabrik could drop more metadata, as it did in 2018, which `osmium fileinfo -e` would show as a metadata line without `version+timestamp`.
  - Where a country is not split into regions, the extract to download and to read on every build is that country's whole file.
