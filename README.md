# Vermont redistricting 2020
Data processing for the 2020 redistricting cycle in Vermont

## Block correspondence files

Usable in both [Districtbuilder](https://www.districtbuilder.org/) and [Districtr](https://districtr.org/) (where [a good example file is provided](https://districtr.org/assets/import-export-examples/districting/assignment-ec0a7f81.csv)), block correspondence files are tables that show membership of each census block within a parent unit; in this case Legislative House and Senate districts for Vermont, as well as Wards and Districts for the city of Burlington.

### Data acquisition
[VCGI](https://vcgi.vermont.gov/) has provided up-to-date boundaries for the legislative districts [here](https://drive.google.com/drive/folders/1TSMJ5Xv2MaqENmzMa_43pxnNA1ce6HGm) (enriched with the 2020 census data). And while the city of Burlington [provides excellent graphics](https://www.burlingtonvt.gov/sites/default/files/CT/ElectionMaps/burlington_vermont_city_wards_2015_24x36.pdf), the geodata behind them will have to be retrieved from [the participatory archives of 2013](https://www.dropbox.com/s/z21tn0nc0oe4iql/btv_ward_stats_2018.geojson?dl=0).

Meanwhile, the census provides TIGER data (boundary files) for census blocks of both the 2010 and 2020 vintages [here](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html). _Note that there are differences in block boundaries between the two vintages! We'll have to produce correspondence files for each to be comprehensive.

Lacking direct links to some of the above, we'll assume manual download and placement in a `data/` directory. Five datasets in all:

1. Census blocks 2010
2. Census blocks 2020 (packaged in the same geodatabase as item 1)
3. VT state Senate districts
4. VT state House districts
5. Burlington city Wards

### Database configuration
Let's PostGIS this situation.

```sh
# Make the DB
createdb vt_districts_2020
# Enable PostGIS
psql vt_districts_2020 -c "CREATE EXTENSION postgis"
```

### Data ingestion
Because some area calculations are going to be used, we'll standardize on the Vermont State Plane projection (EPSG:32145), and use the [OGR2OGR geospatial Swiss army knife](https://gdal.org/) to import the data.

```
# Census blocks 2010
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2020" data/tlgdb_2020_a_50_vt.gdb/ Block10 -nln block10 -nlt PROMOTE_TO_MULTI
# Census blocks 2020
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2020" data/tlgdb_2020_a_50_vt.gdb/ Block20 -nln block20 -nlt PROMOTE_TO_MULTI
# VT state Senate districts
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2020" -sql "SELECT sldust20,geoid20,namelsad20,members,ideal_valu,POP100,deviation,per_dev FROM vt_senate_2020_enriched" data/vt_senate_2020_enriched/vt_senate_2020_enriched.shp -nln vt_senate_2020 -nlt PROMOTE_TO_MULTI
# VT state House districts
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2020" -sql "SELECT sldlst20,geoid20,namelsad20,members,ideal_valu,POP100,deviation,perc_dev AS per_dev FROM vt_house_2020_enriched" data/vt_house_2020_enriched/vt_house_2020_enriched.shp -nln vt_house_2020 -nlt PROMOTE_TO_MULTI
# Burlington wards
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2020" data/btv_ward_stats_2018.geojson -nln btv_wards_2018 -nlt PROMOTE_TO_MULTI
```

### Correspondence file generation
There are two vintages and three sets of districts, so that'll be 6 correspondence files. let's tackle them by vintage.

#### 2010

```sh
# Senate
psql vt_districts_2020 -c "\\copy (
  SELECT
    block10.geoid AS block_id,
    vt_senate_2020.ogc_fid AS assignment
  FROM vt_senate_2020
  JOIN block10 ON ST_Intersects(ST_Centroid(block10.shape), vt_senate_2020.wkb_geometry)
) TO STDOUT CSV HEADER" > correspondence/vt_senate_2010.csv

# House
psql vt_districts_2020 -c "\\copy (
  SELECT
    block10.geoid AS block_id,
    vt_house_2020.ogc_fid AS assignment
  FROM vt_house_2020
  JOIN block10 ON ST_Intersects(ST_Centroid(block10.shape), vt_house_2020.wkb_geometry)
) TO STDOUT CSV HEADER" > correspondence/vt_house_2010.csv

# City
psql vt_districts_2020 -c "\\copy (
  SELECT
    block10.geoid AS block_id,
    btv_wards_2018.ogc_fid AS assignment
  FROM btv_wards_2018
  JOIN block10 ON ST_Intersects(ST_Centroid(block10.shape), btv_wards_2018.wkb_geometry)
    AND ST_Area(ST_Intersection(block10.shape, btv_wards_2018.wkb_geometry)) > (ST_Area(block10.shape) / 2)
) TO STDOUT CSV HEADER" > correspondence/btv_wards_2010.csv
```

#### 2020
```sh
# Senate
psql vt_districts_2020 -c "\\copy (
  SELECT
    block20.geoid AS block_id,
    vt_senate_2020.ogc_fid AS assignment
  FROM vt_senate_2020
  JOIN block20 ON ST_Intersects(ST_Centroid(block20.shape), vt_senate_2020.wkb_geometry)
) TO STDOUT CSV HEADER" > correspondence/vt_senate_2020.csv

# House
psql vt_districts_2020 -c "\\copy (
  SELECT
    block20.geoid AS block_id,
    vt_house_2020.ogc_fid AS assignment
  FROM vt_house_2020
  JOIN block20 ON ST_Intersects(ST_Centroid(block20.shape), vt_house_2020.wkb_geometry)
) TO STDOUT CSV HEADER" > correspondence/vt_house_2020.csv

# City
psql vt_districts_2020 -c "\\copy (
  SELECT
    block20.geoid AS block_id,
    btv_wards_2018.ogc_fid AS assignment
  FROM btv_wards_2018
  JOIN block20 ON ST_Intersects(ST_Centroid(block20.shape), btv_wards_2018.wkb_geometry)
    AND ST_Area(ST_Intersection(block20.shape, btv_wards_2018.wkb_geometry)) > (ST_Area(block20.shape) / 2)
) TO STDOUT CSV HEADER" > correspondence/btv_wards_2020.csv
```

### Play around!
Once uploaded to one of the online tools, [the resulting plans can be shared and evaluated](https://app.districtbuilder.org/projects/4eda3498-b405-44d5-8e37-c37c36563ddc).

![editing](img/editing.png)
