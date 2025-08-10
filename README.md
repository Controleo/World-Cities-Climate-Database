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
