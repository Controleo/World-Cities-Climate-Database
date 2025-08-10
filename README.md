# World-Cities-Climate-Database
# 🌍 World Cities Climate Database Creation Project

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-13+-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![PostGIS](https://img.shields.io/badge/PostGIS-3.x-6E4C13?logo=postgis&logoColor=white)](https://postgis.net/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 📖 Overview
This project creates a **polygon-based spatial database** of major world cities enriched with multiple environmental, geological, and climatic attributes.  
Using **PostgreSQL + PostGIS** aggregation queries, it integrates datasets into a unified, queryable spatial dataset for geospatial analysis.

The end result is a polygon dataset containing:
- City location and metadata
- Climate classification
- Elevation statistics and landform classification
- Sunshine duration
- Precipitation and temperature ranges
- Geological, pedological, and lithological composition

---

## 📊 Dataset Overview

| Column              | Source                  | Description |
|---------------------|-------------------------|-------------|
| `id`                | World Cities            | Unique ID |
| `name`              | World Cities            | City name |
| `climate`           | Climate                 | Extract value according to dataset legend |
| `situation`         | DEM                     | Classify elevation as: `Plaine (0-200m)`, `Colline (200-800m)`, `Montagne (>800m)` |
| `sunshine`          | Diurnal Range           | Average radiation of polygon |
| `elevation_min`     | DEM                     | Minimum elevation |
| `elevation_moy`     | DEM                     | Mean elevation |
| `elevation_max`     | DEM                     | Maximum elevation |
| `precipitation_min` | Rainfall                 | Minimum precipitation |
| `precipitation_moy` | Rainfall                 | Mean precipitation |
| `precipitation_max` | Rainfall                 | Maximum precipitation |
| `temperature_min`   | Temperature min          | Minimum temperature |
| `temperature_moy`   | Temperature average      | Mean temperature |
| `temperature_max`   | Temperature max          | Maximum temperature |
| `geology`           | Geology                 | Top 3 geological types by area |
| `pedology`          | Pedology                | Top 3 soil types by area |
| `lithology`         | Lithology               | Top 3 lithologies by area |

---

## 🧹 Data Cleaning

Invalid geometries can cause spatial queries to fail.  
Check and remove invalid entries:

```sql
-- Check for validity
SELECT ST_IsValid(geom) FROM polygons_wld;

-- Delete invalid geometries
DELETE FROM polygons_wld
WHERE ST_IsValid(geom) = false;
``` 
## 🔍 Data Analysis Workflow
Large queries were split into smaller intermediate tables for performance:
Temperature statistics
Elevation classification
Precipitation aggregation
Sunshine & climate values
Pedology rankings
Geology rankings

### 1️⃣ Temperature Aggregation
``` sql
CREATE TABLE temperature AS
SELECT za.id, za.nom AS name,
       AVG(temp_min.dn) AS min_temperature,
       AVG(temp_moy.degres) AS moy_temperature,
       AVG(temp_max.dn) AS max_temperature,
       za.geom
FROM polygons_wld za
LEFT JOIN temp_minimales temp_min ON ST_Within(za.geom, temp_min.geom)
LEFT JOIN temp_moyennes temp_moy ON ST_Within(za.geom, temp_moy.geom)
LEFT JOIN temp_maximales temp_max ON ST_Within(za.geom, temp_max.geom)
GROUP BY za.id, za.nom, za.geom;
```
### 2️⃣ Elevation Classification
```sql
CREATE TABLE elevation AS
SELECT za.id, za.nom AS name,
       MIN(dem.zlevel) AS elevation_min,
       AVG(dem.zlevel) AS elevation_moy,
       MAX(dem.zlevel) AS elevation_max,
       CASE
           WHEN AVG(dem.zlevel) <= 200 THEN 'Plaine'
           WHEN AVG(dem.zlevel) > 200 AND AVG(dem.zlevel) <= 800 THEN 'Colline'
           ELSE 'Montagne'
       END AS situation,
       za.geom
FROM polygons_wld za
LEFT JOIN "merge 8" dem ON ST_Disjoint(za.geom, dem.geom)
GROUP BY za.id, za.nom, za.geom;
```
### 3. Precipitation Aggregation
```sql
CREATE TABLE precipitation AS
SELECT za.id, za.nom AS name,
       MIN(rainfall.mm) AS precipitation_min,
       AVG(rainfall.mm) AS precipitation_moy,
       MAX(rainfall.mm) AS precipitation_max,
       za.geom
FROM polygons_wld za
LEFT JOIN precipitation_moyennes rainfall ON ST_Within(za.geom, rainfall.geom)
GROUP BY za.id, za.nom, za.geom;
```
### 4. Sunshine & Climate Values
```sql
CREATE TABLE weatherclimate AS
SELECT za.id, za.nom AS name,
       AVG(diurnal.dn) AS sunshine,
       AVG(weather.zlevel) AS climat,
       za.geom
FROM polygons_wld za
LEFT JOIN diurnal_range diurnal ON ST_Within(za.geom, diurnal.geom)
LEFT JOIN weather ON ST_Within(za.geom, weather.geom)
GROUP BY za.id, za.nom, za.geom;
```
### 5. Pedology Ranking & Concatenation
Ranking:
```sql
CREATE TABLE pedologyrank AS
WITH ranked AS (
    SELECT name_fr, sqkm, country, geom,
           RANK() OVER (PARTITION BY country ORDER BY sqkm DESC) AS rnk
    FROM public.pedology
)
SELECT * FROM ranked WHERE rnk <= 3;
```
Concatenation:
```sql
CREATE TABLE pedologycolumn AS
WITH
M1 AS (SELECT name_fr, country, geom FROM pedologyrank WHERE rnk=1),
M2 AS (SELECT name_fr, country FROM pedologyrank WHERE rnk=2),
M3 AS (SELECT name_fr, country FROM pedologyrank WHERE rnk=3)
SELECT CONCAT(M1.name_fr, ', ', M2.name_fr, ', ', M3.name_fr) AS pedology, M1.geom
FROM M1
LEFT JOIN M2 ON M1.country = M2.country
LEFT JOIN M3 ON M1.country = M3.country;
```
### 6. Geology Ranking and Concatenation
```sql
CREATE TABLE geologyrank AS
WITH ranked AS (
    SELECT name_fr, perimeter, country, geom,
           RANK() OVER (PARTITION BY country ORDER BY perimeter DESC) AS rnk
    FROM public.geology
)
SELECT * FROM ranked WHERE rnk <= 3;

CREATE TABLE geologycolumn AS
WITH
M1 AS (SELECT name_fr, country, geom FROM geologyrank WHERE rnk=1),
M2 AS (SELECT name_fr, country FROM geologyrank WHERE rnk=2),
M3 AS (SELECT name_fr, country FROM geologyrank WHERE rnk=3)
SELECT CONCAT(M1.name_fr, ', ', M2.name_fr, ', ', M3.name_fr) AS geology, M1.geom
FROM M1
LEFT JOIN M2 ON M1.country = M2.country
LEFT JOIN M3 ON M1.country = M3.country;
```
### Final Table Compilation
```sql
SELECT za.id, za.nom AS name,
       climat, situation, sunshine,
       elevation_min, elevation_moy, elevation_max,
       precipitation_min, precipitation_moy, precipitation_max,
       min_temperature, moy_temperature, max_temperature,
       pedology, geology, za.geom
FROM polygons_wld za
LEFT JOIN temperature ON ST_Within(za.geom, temperature.geom)
LEFT JOIN elevation ON ST_Within(za.geom, elevation.geom)
LEFT JOIN precipitation ON ST_Within(za.geom, precipitation.geom)
LEFT JOIN weatherclimate ON ST_Within(za.geom, weatherclimate.geom)
LEFT JOIN pedologycolumn ON ST_Within(za.geom, pedologycolumn.geom)
LEFT JOIN geologycolumn ON ST_Within(za.geom, geologycolumn.geom);
```



