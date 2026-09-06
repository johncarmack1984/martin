---
icon: material/grid
tags:
  - tile-sources
  - configuration
---

# Tile Grids

By default, Martin serves tiles on the Web Mercator grid (EPSG:3857).
That grid has one tile at zoom 0 and four times as many at every zoom level after.

Some data is published on other grids.
Examples are national projections like New Zealand's NZTM2000, polar grids or grids for other planets.
Martin can also serve a source on such a grid and clients like [MapLibre GL JS with `addProjection` as `tileMatrix: {origin, extentAtZoom0}`](https://github.com/maplibre/maplibre-gl-js/pull/8287) or [OpenLayers with a custom tile grid](https://gis.stackexchange.com/questions/394782/openlayers-custom-tilegrid-for-scanned-map-image-how-to-calculate-resolutions) can then render it.

We follow the "quad" family of the [OGC Two Dimensional Tile Matrix Set](https://docs.ogc.org/is/17-083r4/17-083r4.html) standard.
A tile grid in Martin is a square power-of-two grid and is defined by:
- a coordinate reference system (CRS),
- the top-left corner of the zoom-0 tile,
- the side length of that tile and
- how many tiles zoom 0 has.

Every zoom level splits each tile into four with columns growing east and rows growing south.

Two grids are built in:

- `WebMercatorQuad` is **the default** in martin and in all modern web maps (maplibre, google, mapbox, ...).
- `WorldCRS84Quad` is plain longitude and latitude with two tiles at zoom 0.

## Defining a tile grid

Grids are named under the top-level `tile_grids` key.
PostgreSQL tables and functions, MBTiles files and PMTiles files refer to a grid by name via `tile_grid`.
A PostgreSQL connection can set a default `tile_grid` for all of its sources.

```yaml hl_lines="1-5 15 26"
tile_grids:
  NZTM2000Quad:
    crs: EPSG:2193
    origin: [-3260586.7284, 10438190.1652]
    extent_at_zoom0: 10018754.1714

postgres:
  connection_string: postgres://postgres@localhost/db
  tables:
    nz_roads:
      schema: public
      table: roads
      srid: 2193
      geometry_column: geom
      tile_grid: NZTM2000Quad
    nz_roads_mercator:
      schema: public
      table: roads
      srid: 2193
      geometry_column: geom

mbtiles:
  sources:
    nz_basemap:
      path: /data/nz_basemap.mbtiles
      tile_grid: NZTM2000Quad
```

We support these settings:

| key               | default  | meaning                                                                                                                                                                                                                                                                                                   |
|-------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `crs`             | required | The CRS as `AUTHORITY:CODE`, or `simple` for planar units without a CRS (see [Non-geographic grids](#non-geographic-grids)). `EPSG` codes work out of the box. Other authorities must exist in the database's `spatial_ref_sys` table (see [Non-EPSG coordinate reference systems](#non-epsg-coordinate-reference-systems)). |
| `origin`          | required | Top-left corner `[x, y]` of the zoom-0 tile in CRS units.                                                                                                                                                                                                                                                 |
| `extent_at_zoom0` | required | Side length of one zoom-0 tile in CRS units.                                                                                                                                                                                                                                                              |
| `matrix_at_zoom0` | `[1, 1]` | Number of tile `[columns, rows]` at zoom 0.                                                                                                                                                                                                                                                               |

`WorldCRS84Quad` and most planetary geographic grids have `[2, 1]` tiles at zoom 0.
The OGC UTM quads however have `[1, 2]`.
A grid that is `[2, 2]` at zoom 0 is `[1, 1]` at zoom 1, so define it starting there.

Take `origin` and `extent_at_zoom0` from the tile matrix set document published with the data.
Do not derive them from the CRS's area of use.
Some documents list the origin northing-first, for example LINZ does for NZTM2000Quad.
Martin expects `[x, y]`, so you may need to switch `x` and `y`.

Some services count zoom levels from a 2x2 level.
NASA GIBS publishes its polar grids that way.
Their "level 0" is zoom 1 in Martin.
The zoom-0 tile is then the whole 8,388,608 m square with origin `[-4194304, 4194304]`.

Serving one table on two grids means two sources, as in the example above.
There is no per-request grid selection.
A source's URL and its cache entries never change with the grid.

## Serving a source on a grid

For a PostgreSQL table, Martin uses the grid's zoom-0 square as `bounds` for `ST_TileEnvelope` and encodes geometry in the grid's CRS.
A table stored in the grid's CRS needs no reprojection.
A table stored in another CRS is transformed by PostGIS.

PostgreSQL functions, MBTiles files and PMTiles files are served as they are.
Martin trusts them to produce tiles on the grid they declare and only advertises it.
Declaring a grid changes the `TileJSON` and the catalog, never the tile bytes.

Requests for tiles outside of the grid answer 404.
This includes a third column at zoom 0 of a two-wide grid.
This also holds for Web Mercator, where such tiles used to answer an empty 204.

The `TileJSON` of a source on a non-default grid has a `tileGrid` key.
It uses the same field names as MapLibre GL JS:

```json
"tileGrid": {
  "id": "NZTM2000Quad",
  "crs": "EPSG:2193",
  "origin": [-3260586.7284, 10438190.1652],
  "extentAtZoom0": 10018754.1714
}
```

The `/catalog` entry of such a source names the grid in `tile_grid`.
Sources on different grids cannot be combined into one composite source.

`martin-cp` copies a source on its own grid.
Without `--bbox` it copies the whole grid.
With `--bbox`, the bounds are `min_x,min_y,max_x,max_y` in the grid's CRS units, not longitude and latitude.
The `TileJSON` written into the archive contains the `tileGrid` key.
The archive can then be served back with the same `tile_grid` declared.

## Tables stored in another CRS

If a table is stored in a different CRS than its grid, Martin transforms each tile envelope into the table's CRS to query the spatial index.
The envelope is densified first.
This keeps edges that curve in the table's CRS inside the search area.

A pole inside the envelope cannot be recovered by any transform.
This happens on the zoom-0 tile of a polar grid.
Martin detects this at startup and warns for every affected table.
Store polar data in the grid's CRS to avoid the transform entirely.

The `bounds` a table advertises are computed in WGS84, as `TileJSON` requires.
If PostGIS cannot transform the table's CRS to WGS84, Martin skips this and logs why.
This is the case for planetary CRS.
Set `bounds` in the config if the `TileJSON` should have them.

## Non-geographic grids (floor plans and PC/mobile game maps) { #non-geographic-grids }

A grid with `crs: simple` has no geographic meaning.
Its units are whatever the data is in, for example pixels of a scanned plan or metres of a game level.
This is the server side of Leaflet's `CRS.Simple` and of MapLibre GL JS's `simple` projection.

```yaml
tile_grids:
  FloorPlan:
    crs: simple
    origin: [0, 1000]
    extent_at_zoom0: 1000

postgres:
  tables:
    rooms:
      schema: public
      table: rooms
      geometry_column: geom
      tile_grid: FloorPlan
```

The table stores its geometry with SRID 0.
PostGIS uses SRID 0 for coordinates without a CRS.
Nothing is ever transformed.
No `bounds` are computed, since there is no WGS84 to express them in.

## Non-EPSG coordinate reference systems

PostGIS knows a CRS by its row in `spatial_ref_sys`.
CRS that PostGIS does not ship can be added as rows.
An example are the CRS the International Astronomical Union defines for other planets.
A grid names them by `auth_name` and `auth_srid`:

```sql
INSERT INTO spatial_ref_sys (srid, auth_name, auth_srid, proj4text)
VALUES (949900, 'IAU_2015', 49900, '+proj=longlat +a=3396190 +b=3376200 +no_defs +type=crs');
```

```yaml
tile_grids:
  MarsGeographic:
    crs: IAU_2015:49900
    origin: [-180, 90]
    extent_at_zoom0: 360
```

Martin looks the SRID up once when it connects.
If the row is missing, Martin refuses to start.

## Sources that only support Web Mercator

COG, GeoJSON and DuckDB sources always produce Web Mercator tiles.
They cannot be declared to be on another grid.

- A COG is read as the Web Mercator pyramid it was written as.
- GeoJSON is tiled in Web Mercator on the way in.
- The DuckDB queries transform to EPSG:3857.

Martin never reprojects tiles.
Tiles are either generated on the grid or served as stored.
