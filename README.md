# Countries_ALPHA2_MAP
Map containing the countries as geojson file including the name and Alpha code

# Data
The original input comes from naturalearthdata.com, the Admin 0 – Countries point-of-views, based on the POV of the Netherlands.
The data is then filtered to only contain the Name, Abbreviation, Iso/Alpha codes and the geometry.

To replicate the data, follow the script below.
# Note
This data contains only specific columns that are relevant for me, renames Åland to have the alpha 2 code FI (to correspond to Finland) and sorts the entries by 
alpha 2 code, such that the biggest country is always first per alpha 2 code. It also adds a column "biggest" which defines which country is biggest per alpha 2 code.

# Script
1. Download the country POV from https://www.naturalearthdata.com/downloads/10m-cultural-vectors/
2. Unzip the package and update the input_file according to the shapefile.
3. Run the snippet
4. The output is in the downloaded folder


```python
import geopandas as gpd
import numpy as np
input_file = ...

gdf = gpd.read_file(input_file, driver='ESRI Shapefile')
cols = ['NAME', 'NAME_LONG', 'ABBREV', 'NAME_CIAWF', 'ISO_A2_EH', 'ISO_A3_EH', 'CONTINENT', 'geometry']
filtered_data = gdf.loc[:, cols].copy()

# Prefer NAME_CIAWF, fall back to NAME
filtered_data['COUNTRY_NAME'] = filtered_data['NAME_CIAWF'].fillna(filtered_data['NAME'])

# Compute lengths in meters for correct ordering
proj = filtered_data.estimate_utm_crs() if filtered_data.crs is not None else "EPSG:3857"
filtered_data['geom_length'] = filtered_data.to_crs(proj).geometry.length.fillna(0)

# Fix Åland label and ISO mapping without chained assignment
mask_ax = filtered_data['ISO_A2_EH'] == 'AX'
filtered_data.loc[mask_ax, 'COUNTRY_NAME'] = 'Åland'
filtered_data.loc[mask_ax, 'ISO_A2_EH'] = 'FI'

# Create column whether country is biggest with that alpha2 code
g = filtered_data.groupby('ISO_A2_EH', dropna=False)['geom_length']
filtered_data['biggest'] = filtered_data['geom_length'].eq(g.transform('max'))
# Sort by ISO then length (descending); reset index
filtered_data = (
    filtered_data
    .sort_values(['ISO_A2_EH', 'geom_length'], ascending=[True, False], na_position='last')
    .reset_index(drop=True)
)
filtered_data.to_file(input_file.replace('shp', 'geojson'), driver='GeoJSON')


```
