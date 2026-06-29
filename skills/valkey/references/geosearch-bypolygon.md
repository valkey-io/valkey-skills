# GEOSEARCH BYPOLYGON

Use for Valkey 9.0 polygon geo queries, geometry restrictions, distance quirks, antimeridian behavior, and GEOSEARCHSTORE limits.

## GEOSEARCH BYPOLYGON (9.0+)

```
GEOSEARCH key
  BYPOLYGON <num-vertices> lon1 lat1 lon2 lat2 ... lonN latN
  [ASC | DESC]
  [COUNT count [ANY]]
  [WITHCOORD] [WITHDIST] [WITHHASH]
```

- `num-vertices >= 3`.
- Polygon is **auto-closed** - do NOT repeat the first vertex.
- Must be **simple** (non-self-intersecting). Winding (CW/CCW) doesn't matter.
- `FROMMEMBER` / `FROMLONLAT` are **rejected** with BYPOLYGON (syntax error); they're mandatory for BYRADIUS/BYBOX.

### WITHDIST quirk

With BYPOLYGON, `WITHDIST` returns distance from the **polygon's computed centroid**, not from any user-supplied point. For distance from a known location, compute client-side or issue a separate `GEODIST`.

### Coordinate storage

- Longitude: -180 to 180.
- Latitude: **-85.05112878 to 85.05112878** (Web Mercator limits).
- Internally a sorted set (score = geohash); sorted-set commands work on geo keys.

### Footguns

- **Antimeridian wrap**: bounding box is min/max lon/lat; a polygon spanning 170° to -170° through the Pacific produces a box covering the whole globe, over-matches. Split into two polygons (east/west of 180°), union client-side.
- **Self-intersecting** polygons produce even-odd fill. Keep simple.
- **Very small polygons** (< 100 m) hit geohash grid precision - verify with `GEODIST`.

### GEOSEARCHSTORE

Supports BYPOLYGON with same geometry, but **rejects `WITHCOORD`, `WITHDIST`, `WITHHASH`** with an error. Store-and-count XOR annotated-results.

### Complexity

`GEOSEARCH` is O(N + log M) where N = elements in the grid-aligned bounding box around the shape, M = matches actually inside. For large geo sets, always pass `COUNT`.
