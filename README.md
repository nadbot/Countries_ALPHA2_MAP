# Countries_ALPHA2_MAP
Map containing the countries as geojson file including the name and Alpha code

# Data
The original input comes from naturalearthdata.com, the Admin 0 – Countries point-of-views, based on the POV of the Netherlands.
The data is then filtered to only contain the Name, Abbreviation, Iso/Alpha codes and the geometry.

To replicate the data, follow the script below.

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
filtered_data = gdf[['NAME', 'NAME_LONG', 'ABBREV', 'NAME_CIAWF', 'ISO_A2_EH', 'ISO_A3_EH', 'CONTINENT', 'geometry']]
filtered_data['COUNTRY_NAME'] = np.where(filtered_data['NAME_CIAWF'].notna(), filtered_data['NAME_CIAWF'], filtered_data['NAME'])
filtered_data.to_file(input_file.replace('shp', 'geojson'), driver='GeoJSON')

```
