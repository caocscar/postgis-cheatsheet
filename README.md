# POSTGIS Cheat Sheet

## Create table with geometry (geography) column
```SQL
CREATE TABLE geometries (
  name varchar,
  geom geometry
)
```
## Insert Point(LNG, LAT) geometry data
```SQL
INSERT INTO geometries VALUES
  ('Point', 'POINT(0 0)'),
  ('Linestring', 'LINESTRING(0 0, 1 1, 2 1, 2 2)'),
  ('Polygon', 'POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))'),
  ('PolygonWithHole', 'POLYGON((0 0, 10 0, 10 10, 0 10, 0 0),(1 1, 1 2, 2 2, 2 1, 1 1))'),
  ('Collection', 'GEOMETRYCOLLECTION(POINT(2 0),POLYGON((0 0, 1 0, 1 1, 0 1, 0 0)))')
```

## Query geometry data and print out as text
```SQL
SELECT name
  ,ST_AsText(geom)
FROM geometries;
```

## References
- http://postgis.net/workshops/postgis-intro/geometries.html
- http://postgis.net/workshops/postgis-intro/geography.html
- https://postgis.net/docs/using_postgis_dbmanagement.html#Geography_Basics

## Convert columns of latitude, longitude to geometry type
add empty geography column
```SQL
ALTER TABLE your_table 
ADD COLUMN geom geography(Point, 4326);
```
then do an update by using `ST_MakePoint` and `ST_SetSRID`
```SQL
UPDATE your_table 
SET geom = ST_SetSRID(ST_MakePoint(longitude, latitude), 4326);
```

## Find closest point (between two set of points)
Use `ST_ClosestPoint` 
```SQL
SELECT ST_AsText(geog_origin)
    ,B.stop_id
    ,B.stop_name AS pickup_location
FROM via_ride_requests_raw AS A
CROSS JOIN LATERAL (
  SELECT stop_id
    ,stop_name
  FROM ods_stops
  ORDER BY ST_Transform(location, 4326) <-> A.geog_origin
  LIMIT  1
) AS B;
```
## Find out current SRID
```SQL
SELECT ST_SRID(geom) 
FROM nyc_streets
LIMIT 1;
```

## Transform geometry into another SRID
```SQL
SELECT ST_Transform(geom, SRID)
FROM nyc_streets
```
