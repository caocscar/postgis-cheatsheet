# POSTGIS Cheat Sheet <!-- omit in toc -->

- [Create table with geometry (geography) column](#create-table-with-geometry-geography-column)
- [Insert Point(LNG, LAT) geometry data](#insert-pointlng-lat-geometry-data)
- [Query geometry data and print out as text](#query-geometry-data-and-print-out-as-text)
- [References](#references)
- [Insert and store geometry data (4326) into a different CRS](#insert-and-store-geometry-data-4326-into-a-different-crs)
- [Convert columns of latitude, longitude to geometry type](#convert-columns-of-latitude-longitude-to-geometry-type)
- [Find closest point (between two set of points)](#find-closest-point-between-two-set-of-points)
- [Find closest line to point](#find-closest-line-to-point)
- [Find out current SRID](#find-out-current-srid)
- [Transform geometry into another SRID](#transform-geometry-into-another-srid)
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

## Find closest line to point
```SQL
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
