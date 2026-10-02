# Attribution a page rendering OSM data must show

Checked on 2026-10-02 for the plan of the Local map for one area phase.
Both pages were fetched with `curl` and answered HTTP 200.

## Findings

- [openstreetmap.org/copyright](https://www.openstreetmap.org/copyright) states two duties under "How to credit OpenStreetMap": display the attribution notice, and make clear the data is available under the ODbL.
  - It does not give the wording of the notice, and sends the reader to the OSMF [Attribution Guidelines](https://osmfoundation.org/wiki/Licence/Attribution_Guidelines) for how the notice is displayed in each kind of use.
  - It says linking to the copyright page is the usual way to make the licence clear.
- The Attribution Guidelines, under "Requirements to fit within OSMF's safe harbour":
  - The attribution is to "OpenStreetMap".
  - The licence is made clear by making that text a link to `openstreetmap.org/copyright`.
  - "© OpenStreetMap contributors" and "© OpenStreetMap" are called historical forms and remain acceptable.
  - A viewer must see the attribution without interacting with the map.
  - The text has to be legible, with font, size, colour, contrast, position, and time on screen all counted.
- The Attribution Guidelines, under "Interactive maps":
  - The credit typically sits in a corner of the map; any corner is acceptable, the lower right being traditional.
  - Beside the map, or on a splash screen shown at start, is also allowed.
  - The attribution may be collapsed under certain conditions, and a collapsed one must still let the user find the licence information, for example from an "(i)" button.
  - Not read: the full list of conditions under which collapsing is allowed. Read them before relying on a collapsed control.
- The guidelines treat a static image as they treat an interactive map, so a screenshot of the map carries the attribution too.

## What this means for the plan

- The requirement is: the map shows "© OpenStreetMap contributors" in a corner, visible without interaction, linked to `https://www.openstreetmap.org/copyright`.
- Open: whether MapLibre GL JS's default attribution control stays visible without interaction at narrow widths. Check its documentation and the rendered page when the map page is built.
