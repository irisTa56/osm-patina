# Roadmap

## Purpose

A mapper looking for somewhere to improve OpenStreetMap needs to know which parts of the map have gone longest without attention.
OSM Patina shows them where features have gone longest without an edit, so stale areas stand out.
An old edit does not prove the data is wrong, so the map points at candidates, which the mapper then checks against aerial imagery or on site.

## Scope

- **Features**: anything that can be observed from outside without asking anyone or entering a building, such as:
  - buildings
  - land use and land cover
  - parks and natural features
  - roads and paths
  - street furniture, trees, and barriers
  - shops and other places visible from the street
- **Area**: a rectangle around a chosen point, adjustable in size up to a long side of about 5 km.
  - A point a developer picks is never hard-coded, in the repository or in anything published from it, because it can reveal where they live.
  - Later, the point may be the user's current location or one picked on the map.
- **Viewing**: on the local machine first, then published later.

## Non-goals

- **Editing OSM from this tool**: existing OSM editors already do it, and the map only points at where to look.
- **Features that can only be confirmed by asking or going inside**: indoor features and the like cannot be checked from outside, which is how the mapper checks a candidate.

## Phases

- **Local map for one area**: on a developer's machine, a mapper can see which features in one chosen area have gone longest without an edit.
- **Vertex-aware freshness**: a feature whose shape was fixed counts as edited, which Local map for one area does not yet cover.
- **Location**: the map centres on the current location or a picked point.
- **Publishing**: anyone can open the map for an area they choose, without a developer's machine.

Local map for one area comes first, and Publishing follows Location, because until Location the only map is around a point a developer picked.
The rest of the order is decided when Local map for one area ends.
