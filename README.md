# Geographic Search / Geolocation with Neo4j + OpenStreetMap

This README shows a practical, end-to-end pattern for importing OpenStreetMap (OSM) data into Neo4j and running fast geographic / geospatial queries (nearest POIs, radius search, bounding-box filters, simple routing prep). It gives recommended schemas, ingestion options, example Cypher queries, performance tips, and helpful utilities.

Table of contents
- Overview
- Key concepts
- What you'll need
- Data extraction (from OSM)
  - Option A — Use osmnx (easy, Python)
  - Option B — Use pyosmium/osmium-tool (PBF → CSV)
  - Option C — Use an ETL like osm2neo4j / custom scripts
- Import into Neo4j
  - Neo4j schema and examples
  - LOAD CSV + APOC example
  - Bulk import notes (neo4j-admin import)
- Example geospatial queries (radius, nearest, bbox)
- Indexing & performance tips
- Ways (roads) & geometries
- Advanced: combining graph queries + spatial filters
- Roadmap / next steps
- References & credits
- License

Overview
--------
OpenStreetMap is an excellent source of POIs, roads, administrative boundaries and other geodata. Neo4j is a graph database that supports a native Point type and spatial queries (distance, indexing). Combining both allows you to perform geographic search plus graph traversals (e.g., find nearest bus stop and then walk along streets to a destination, or combine POI distance constraints with network queries).

Key concepts
------------
- OSM entities: nodes (points), ways (sequences of nodes = lines/polylines), relations (collections).
- Neo4j geo representation: use the native Point type (srid 4326 for GPS lat/lon).
- Index: create an index on the node property that stores the point for fast search.
- Query pattern: apply a cheap bounding-box filter first (lat/lon range) then compute exact distance().

What you'll need
----------------
- Neo4j 4.x or 5.x (supports Point type). APOC plugin recommended (for data conversions/import helpers).
- Python (optional): osmnx, pandas, pyosmium for extraction workflows.
- OSM PBF file for your area of interest (download from https://download.geofabrik.de/ or planet.osm).
- (Optional) neo4j-admin tool for bulk import.

Data extraction (from OSM)
--------------------------

Option A — osmnx (easy, great for POIs and road networks)
- Install:
  - pip install osmnx pandas
- Example: export POIs (amenity nodes) to CSV
  ```python
  import osmnx as ox
  import pandas as pd

  tags = {'amenity': True}  # or custom tags
  gdf = ox.geometries_from_place("San Francisco, California, USA", tags)
  # Keep only point geometries (nodes)
  points = gdf[gdf.geometry.type == 'Point'].copy()
  points['lon'] = points.geometry.x
  points['lat'] = points.geometry.y
  points.reset_index(inplace=True)
  points[['osmid','name','amenity','lat','lon']].to_csv('sf_pois.csv', index=False)
  ```
- Use the CSV to import into Neo4j.

Option B — pyosmium / osmium-tool (PBF → CSV)
- Use pyosmium for custom tags extraction or osmium tags-filter to extract nodes and ways, then convert to CSV/GeoJSON.
- Example (osmium):
  - osmium tags-filter input.osm.pbf n/amenity -o amenity_nodes.osm.pbf
  - osmium export amenity_nodes.osm.pbf -o amenity_nodes.geojson
- Or write a small pyosmium script to emit CSV rows with id,name,lat,lon,tags.

Option C — ETL / osm2neo4j
- There are community projects that import OSM into Neo4j directly (search for osm2neo4j, osm2graph). They can produce nodes/ways and relationships automatically. Use with caution and check the version compatibility.

Import into Neo4j
-----------------
Prepare the CSV with headers: id,name,lat,lon,tag1,tag2,...

Neo4j configuration
- Put CSV files in the import folder (or serve via http and use LOAD CSV with a URL).
- Install APOC (recommended) to help with WKT, JSON -> properties, etc.

Schema suggestions
- :Place {osm_id, name, tags..., location: Point}
- :Road {osm_id, name, way_id, nodes: [Point], geom_wkt}
- :AdminArea {name, boundary_wkt}
- Relationships
  - (p:Place)-[:LOCATED_IN]->(a:AdminArea)
  - (p:Place)-[:NEAR]->(r:Road) (optional, derived)
  - (n1:RoadNode)-[:NEXT]->(n2:RoadNode) (for routing graph)

Create index on location
```cypher
CREATE INDEX place_location_index FOR (p:Place) ON (p.location);
```

LOAD CSV + APOC example
```cypher
// CSV headers: osm_id,name,lat,lon,type
LOAD CSV WITH HEADERS FROM 'file:///sf_pois.csv' AS row
CREATE (p:Place {
  osm_id: row.osmid,
  name: row.name,
  amenity: row.amenity,
  location: point({srid:4326, latitude: toFloat(row.lat), longitude: toFloat(row.lon)})
});
```

If you want to convert tag JSON strings into properties:
```cypher
// If 'tags' column contains JSON like {"shop":"supermarket","name":"X"}
WITH row, apoc.convert.fromJsonMap(row.tags) AS tags
MERGE (p:Place {osm_id: row.osmid})
SET p += tags,
    p.location = point({srid:4326, latitude: toFloat(row.lat), longitude: toFloat(row.lon)})
```

Bulk import with neo4j-admin
- For millions of nodes, generate CSVs for nodes and relationships and use neo4j-admin import for fastest import:
  - neo4j-admin import --nodes=places.csv --nodes=roads.csv --relationships=road_edges.csv --database=neo4j

Example geospatial queries
--------------------------

Set a center point (e.g., user's location)
```cypher
WITH point({srid:4326, latitude: 37.7749, longitude: -122.4194}) AS center
```

Find all places within 1,000 meters (1 km)
```cypher
WITH point({srid:4326, latitude: 37.7749, longitude: -122.4194}) AS center
MATCH (p:Place)
WHERE distance(p.location, center) <= 1000
RETURN p.name, distance(p.location, center) AS dist
ORDER BY dist ASC
LIMIT 50;
```

Nearest neighbor (K nearest)
```cypher
WITH point({srid:4326, latitude: 37.7749, longitude: -122.4194}) AS center
MATCH (p:Place)
RETURN p.name, distance(p.location, center) AS dist
ORDER BY dist ASC
LIMIT 10;
```

Speed up with bounding box then exact distance
```cypher
WITH 37.7749 AS lat, -122.4194 AS lon, 0.01 AS latDelta, 0.015 AS lonDelta
WITH point({srid:4326, latitude: lat, longitude: lon}) AS center,
     lat - latDelta AS minLat, lat + latDelta AS maxLat,
     lon - lonDelta AS minLon, lon + lonDelta AS maxLon
MATCH (p:Place)
WHERE p.location.latitude >= minLat AND p.location.latitude <= maxLat
  AND p.location.longitude >= minLon AND p.location.longitude <= maxLon
WITH p, distance(p.location, center) AS dist
WHERE dist <= 1000
RETURN p.name, dist ORDER BY dist LIMIT 50;
```

Note: bounding box values depend on latitude for proper meters-to-degrees math; for production compute deltas using simple approximations or use a geodesic library.

Ways (roads) & geometries
-------------------------
- Ways can be stored as arrays of point coordinates or as WKT strings in a 'geom_wkt' property (e.g., LINESTRING).
- Neo4j core doesn't provide native geometry types beyond Point; if you need real geometry operations (intersection, buffering), consider:
  - Store WKT and use external spatial services or APOC spatial utilities if available.
  - Use the Neo4j Spatial plugin (community project) for advanced geometry operations.
- For routing, it's common to normalize OSM ways into a road network graph of RoadNode nodes and NEXT relationships with cost/length properties.

Example: create road nodes for routing (pseudo)
```cypher
// For each unique coordinate in roads CSV create RoadNode nodes
LOAD CSV WITH HEADERS FROM 'file:///road_nodes.csv' AS row
MERGE (n:RoadNode {coord: point({srid:4326, latitude: toFloat(row.lat), longitude: toFloat(row.lon)})})
SET n.osm_node_id = row.node_id;

// Create NEXT edges between consecutive nodes (ordered list)
LOAD CSV WITH HEADERS FROM 'file:///road_edges.csv' AS row
MATCH (a:RoadNode {coord: point({srid:4326, latitude: toFloat(row.lat1), longitude: toFloat(row.lon1)})})
MATCH (b:RoadNode {coord: point({srid:4326, latitude: toFloat(row.lat2), longitude: toFloat(row.lon2)})})
MERGE (a)-[:NEXT {length_m: toFloat(row.length_m)}]->(b);
```

Indexing & performance tips
---------------------------
- Create an index on the Point property used for geosearches (see earlier).
- Use bounding-box filtering before computing distance when dataset is large.
- Partition data spatially (tiles) if you need to scale horizontally or shard by region.
- Use neo4j-admin import for very large bulk loads instead of LOAD CSV.
- Avoid storing huge arrays of coordinates on single nodes; instead, normalize to nodes for routing.

Advanced: combining graph queries + spatial filters
--------------------------------------------------
- Example: find nearest POI that is reachable by walking along roads under 1km distance (graph + geo)
  - Find candidate POIs within a geometric radius, then run shortestPath/shortestWeightedPath between the user's nearest road node and each candidate POI's nearest road node. Filter by path length.
- Use KNN + ranking: combine distance to POI and graph-based travel time (speed * road lengths).

Roadmap / next steps
--------------------
- Add automated ETL scripts (pyosmium/osmnx) for your chosen region and tag mapping.
- Export sample datasets and unit tests for imports.
- Add a small example web UI that accepts a location and returns nearest POIs using these Cypher queries.
- Integrate routing (e.g., using Graph Data Science or custom Dijkstra on RoadNode graph).

References & useful links
-------------------------
- OpenStreetMap: https://www.openstreetmap.org
- Geofabrik OSM extracts: https://download.geofabrik.de/
- osmnx: https://github.com/gboeing/osmnx
- pyosmium: https://github.com/osmcode/pyosmium
- Neo4j spatial docs (Point type, distance): https://neo4j.com/docs/cypher-manual/current/functions/spatial/
- APOC plugin: https://neo4j.com/labs/apoc/

License
-------
MIT

Contributors
------------
- You (the implementer)
- Community libraries: osmnx, pyosmium, APOC, Neo4j
