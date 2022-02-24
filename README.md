# POSTGIS Cheat Sheet <!-- omit in toc -->

- [Create table with geometry (geography) column](#create-table-with-geometry-geography-column)
- [Insert Point(LNG, LAT) geometry data](#insert-pointlng-lat-geometry-data)
- [Query geometry data and print out as text](#query-geometry-data-and-print-out-as-text)
- [Create spatial index](#create-spatial-index)
- [References](#references)
- [Insert and store geometry data (4326) into a different CRS](#insert-and-store-geometry-data-4326-into-a-different-crs)
- [Convert columns of latitude, longitude to geometry type](#convert-columns-of-latitude-longitude-to-geometry-type)
- [Find closest point (between two set of points)](#find-closest-point-between-two-set-of-points)
- [Find closest line to point](#find-closest-line-to-point)
- [Extract Lng, Lat from Point](#extract-lng-lat-from-point)
- [Find out current SRID](#find-out-current-srid)
- [Transform geometry into another SRID](#transform-geometry-into-another-srid)
- [Return GeoJSON FeatureCollection from table](#return-geojson-featurecollection-from-table)
- [Insert from GeoJSON feature](#insert-from-geojson-feature)
- [Setup PostGIS on AWS RDS Postgres](#setup-postgis-on-aws-rds-postgres)

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

## Create spatial index
```SQL
CREATE INDEX <index-name> ON <table> USING gist (geog);
```

## References
- http://postgis.net/workshops/postgis-intro/geometries.html
- http://postgis.net/workshops/postgis-intro/geography.html
- https://postgis.net/docs/using_postgis_dbmanagement.html#Geography_Basics

## Insert and store geometry data (4326) into a different CRS
These two inserts are equivalent in terms of result.
```SQL
CREATE TABLE test (
  geom geometry(Point, 3857)
);
INSERT INTO test VALUES
(ST_Transform('SRID=4326;POINT(-83.7 42.3)'::geometry, 3857)),
(ST_Transform(ST_SetSRID('POINT(-83.7 42.3)'::geometry, 4326), 3857));
```

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
```SQL
SELECT ST_AsText(geog_origin)
    ,B.stop_id
    ,B.stop_name AS pickup_location
FROM via_ride_requests_raw AS A
CROSS JOIN LATERAL (
	SELECT stop_id
		,stop_name
		,geog <-> A.geog_origin AS distance
	FROM ods_stops
	ORDER BY distance
	LIMIT 1
) AS B
```

## Find closest line to point
A is the point table and B is the linestring table. If tables were reversed, it would find the closest point to the line.
```SQL
SELECT A.edge
	,A.lat
	,A.lng
	,B.segment_id
	,B.st_name
	,B.manual_only
	,B.distance
FROM production_edges AS A
CROSS JOIN LATERAL (
	SELECT segment_id
		,st_name
		,manual_only
		,geog <-> A.geog AS distance 
	FROM ods_route
	ORDER BY distance
	LIMIT 1
) AS B
```

## Extract Lng, Lat from Point
```SQL
SELECT ST_X(geom)
    ST_Y(geom)
FROM nyc_streets
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

## Return GeoJSON FeatureCollection from table
```SQL
SELECT json_build_object(
    'type', 'FeatureCollection',
    'features', json_agg(
        json_build_object(
            'type', 'Feature',
            'geometry', ST_AsGeoJSON(geometry)::json,
            'properties', json_build_object(
                'stop_id', stop_id,
                'stop_name', stop_name
            )
        )
    )
)
FROM nyc_streets;
```

## Insert from GeoJSON feature
Point
```SQL
```

Linestring
```SQL
INSERT INTO ods_route 
VALUES ('GR','171','BRIDGE ST NW','false', ST_GeomFromGeoJSON('{"type": "LineString", "coordinates": [[-85.67919, 42.97059], [-85.67893, 42.97059]]}'))
```

Polygon
```SQL
```
## Setup PostGIS on AWS RDS Postgres
Reference: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.PostGIS.html

```SQL
SELECT CURRENT_USER;
CREATE EXTENSION postgis;
CREATE EXTENSION fuzzystrmatch;
CREATE EXTENSION postgis_tiger_geocoder;
CREATE EXTENSION postgis_topology;
ALTER SCHEMA tiger OWNER TO rds_superuser;
ALTER SCHEMA tiger_data OWNER TO rds_superuser; 
ALTER SCHEMA topology OWNER TO rds_superuser;
```

Verify ownership of the extensions
```SQL
SELECT n.nspname AS "Name"
    ,pg_catalog.pg_get_userbyid(n.nspowner) AS "Owner"
FROM pg_catalog.pg_namespace n
WHERE n.nspname !~ '^pg_'
AND n.nspname <> 'information_schema'
ORDER BY 1;
```

Transfer ownership of the PostGIS objects to the rds_superuser role
```SQL
CREATE FUNCTION exec(text) returns text language plpgsql volatile AS $f$ BEGIN EXECUTE $1; RETURN $1; END; $f$;
```

Alter permissions
```SQL
SELECT exec('ALTER TABLE ' || quote_ident(s.nspname) || '.' || quote_ident(s.relname) || ' OWNER TO rds_superuser;')
FROM (
    SELECT nspname
		    ,relname
    FROM pg_class c 
    JOIN pg_namespace n ON (c.relnamespace = n.oid) 
    WHERE nspname IN ('tiger','topology')
    AND relkind IN ('r','S','v')
    ORDER BY relkind = 'S'
) s;
```

Test the extensions
```SQL
-- add tiger to search path
SET search_path=public,tiger;
-- test tiger
SELECT na.address
    ,na.streetname
    ,na.streettypeabbrev
    ,na.zip
FROM normalize_address('915 E Washington St, Ann Arbor, MI 48109') AS na;
-- test topology
SELECT topology.createtopology('my_new_topo', 26986, 0.5);
```

List current postgis version
```SQL
SELECT PostGIS_Version();
SELECT PostGIS_Full_Version();
```

List available postgis versions
```SQL
SELECT *
FROM pg_available_extension_versions
WHERE name='postgis'
```
