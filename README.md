# Vermont redistricting 2020
Data processing for the 2020 redistricting cycle in Vermont

## Block correspondence files

Usable in both Districtbuilder and Districtr (where [a good example file is provided](https://districtr.org/assets/import-export-examples/districting/assignment-ec0a7f81.csv)), block correspondence files are tables that show membership of each census block within a parent unit; in this case Legislative House and Senate districts for Vermont, as well as Wards and Districts for the city of Burlington.

### Data acquisition
VCGI has provided up-to-date boundaries for the legislative districts [here](https://drive.google.com/drive/folders/1TSMJ5Xv2MaqENmzMa_43pxnNA1ce6HGm) (enriched with the 2020 census data). And while the city of Burlington [provides excellent graphics](https://www.burlingtonvt.gov/sites/default/files/CT/ElectionMaps/burlington_vermont_city_wards_2015_24x36.pdf), the geodata behind them will have to be retrieved from [the participatory archives of 2013](https://www.dropbox.com/s/z21tn0nc0oe4iql/btv_ward_stats_2018.geojson?dl=0).

Meanwhile, the census provides TIGER data (boundary files) for census blocks of both the 2010 and 2020 vintages [here](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html). _Note that there are differences in block boundaries between the two vintages! We'll have to produce correspondence files for each to be comprehensive.

Lacking direct links to some of the above, we'll assume manual download and placement in a `data/` directory. Five datasets in all:

1. Census blocks 2010
2. Census blocks 2020
3. VT state Senate districts
4. VT state House districts
5. Burlington city Wards

### Database configuration
Let's PostGIS this situation.

```sh
# Make the DB
createdb vt_districts_2021
# Enable PostGIS
psql vt_districts_2021 -c "CREATE EXTENSION postgis"
```

### Data ingestion
Because some area calculations are going to be used, we'll standardize on the Vermont State Plane projection (EPSG:32145), and use the [OGR2OGR geospatial Swiss army knife](https://gdal.org/) to import the data.

```
ogr2ogr -t_srs "EPSG:32145" -f "PostgreSQL" PG:"host=localhost dbname=vt_districts_2021" ne_10m_admin_1_states_provinces.shp -nln vt_border
```
