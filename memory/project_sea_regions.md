---
name: Sea regions rendering
description: Sea regions should be processed same as land in Voronoi/rendering, only colored differently
type: project
---

Sea regions (`sea_*`) should participate in Voronoi cell building and rendering identically to land regions. They are valid seeds and should get cells. The only difference is color — sea cells should use a distinct sea color instead of the land/faction color.

**Why:** Sea regions are real game regions. Excluding them from the Voronoi means their area is incorrectly absorbed by neighboring land cells, distorting land cell shapes.

**How to apply:** When implementing:
- Remove the `!k.startsWith('sea_')` filter from `geoKeys` in `_buildGeo` (or keep for rendering exclusion — actually remove entirely, sea regions get cells too)
- In the draw/color logic, check `key.startsWith('sea_')` and apply `COL_OCEAN` or a dedicated sea color instead of faction/heat colors
