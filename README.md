# TAZ generation from OSM

<img width="1790" height="845" alt="TAZ" src="https://github.com/user-attachments/assets/6189b101-5fc4-4019-a5d2-885115619de3" />

`TAZ_generation.ipynb` builds **Traffic Analysis Zones** for a circular study
area by carving its bounding box with linear barriers (roads, railways, rivers)
sourced from OpenStreetMap, then merging tiny slivers.

## Inputs

Two parameters at the top of the notebook:

| Parameter  | Example                | Meaning                            |
|------------|------------------------|------------------------------------|
| `center`   | `(50.01640, 20.01117)` | AOI centre, `(lat, lon)` in WGS84  |
| `radius_m` | `10_000`               | AOI radius in metres               |

Everything downstream is derived from these two values.

## Pipeline

1. **Download from OSM** (via `osmnx`):
   - **Roads** — `motorway`, `trunk`, `primary`, `secondary`, `tertiary`
     plus their `_link` (junction) variants.
   - **Railways** — `railway=rail` with no `service=*` tag (i.e. main / branch
     lines only, no sidings, yards, spurs or crossovers).
   - **Rivers** — `waterway` of type `river`, `canal`, `stream`, `drain`.
   - **Forests** — polygons tagged `landuse=forest` or `natural=wood`.

2. **Per-class parallel-way merge** — for each highway / railway / waterway
   class, buffer every way by a class-specific width, dissolve overlapping
   buffers, then extract the centerline of each resulting polygon with the
   `centerline` library. Dual carriageways and double-track railways collapse
   to a single feature; isolated ways pass through unchanged.

3. **Unified barrier centerline** — stack the three line layers, buffer them by
   `UNIFIED_BUFFER_M`, dissolve everything together, run `Centerline` on each
   polygon blob, then **Chaikin smoothing** (2 iterations).

4. **Dead-end pruning** — build a graph from the unified centerline and
   iteratively drop leaf branches shorter than `MIN_BRANCH_M`. A branch is the
   path from a degree-1 vertex back to the nearest junction (degree ≥ 3);
   removing one can expose a new dead end, so the loop runs to fixpoint.

5. **Zone generation** — build the AOI bounding box (in EPSG:2180), clip the
   pruned barrier network to it, then `shapely.ops.polygonize` on the union of
   the box boundary and the barriers to extract every face.

6. **Sliver merging** — repeatedly find the smallest zone below
   `MIN_ZONE_AREA_M2` and merge it into the neighbour with which it shares the
   **longest common border**, until every zone meets the threshold.

## Outputs

A single GeoPackage `osm_barriers.gpkg` with six layers:

| Layer      | Geometry       | Description                                 |
|------------|----------------|---------------------------------------------|
| `roads`    | LineString     | Per-class merged road centerlines           |
| `railways` | LineString     | Per-class merged rail centerlines           |
| `rivers`   | LineString     | Per-class merged waterway centerlines       |
| `forests`  | Polygon        | Dissolved forest / wood patches             |
| `barriers` | (Multi)LineString | Unified, smoothed, dead-end-pruned barriers |
| `zones`    | Polygon        | Final TAZ polygons                          |

## Tunable parameters

| Symbol               | Default                     | Effect                                           |
|----------------------|-----------------------------|--------------------------------------------------|
| `ROAD_BUFFER_M`      | per-class table (40–55 m)   | Half the spacing between parallel carriageways   |
| `RAIL_BUFFER_M`      | `{"rail": 50}`              | Buffer for double-track collapse                 |
| `RIVER_BUFFER_M`     | per-class table (35–45 m)   | Buffer for braided / dual-channel waterways      |
| `UNIFIED_BUFFER_M`   | `50`                        | Buffer when fusing roads + rails + rivers        |
| `MIN_BRANCH_M`       | `3 * UNIFIED_BUFFER_M`      | Max branch length pruned as a Voronoi spur       |
| `MIN_ZONE_AREA_M2`   | `250_000` (0.25 km²)        | Slivers below this get absorbed into neighbours  |

## Dependencies

```
pip install osmnx geopandas shapely centerline networkx matplotlib
```

CRS used for length / area operations needs to be metric (i.e. 27700 for UK - Replace if working elsewhere.).

## Quick usage

1. Edit `center` and `radius_m` in the settings code cell.
2. Run all cells top-to-bottom.
3. The final cell renders a two-panel preview; the GPKG is written next to the
   notebook as `osm_barriers.gpkg`.

## Caveats

- **OSM completeness.** Outputs only reflect what is mapped in OSM at run time.
  This is generally good for trunk and primary roads and 
  railway lines, less so for small streams and farm tracks.
- **Coastal / boundary AOIs.** The dead-end pruner needs barriers to terminate
  on either the AOI rectangle or a junction; barriers that fade out inside the
  AOI may leave un-split regions. Increase `radius_m` slightly to push more
  barriers across the rectangle's edge.
- **`centerline` on near-1D polygons.** Buffers that didn't actually fuse two
  parallel ways can produce noisy Voronoi skeletons; the pruning step
  (`MIN_BRANCH_M`) is what keeps the output usable.
- **Sliver merging is greedy.** Smallest-first with longest-shared-border is a
  good heuristic but not optimal; results can differ slightly if the input
  zone order changes.
