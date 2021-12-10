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
