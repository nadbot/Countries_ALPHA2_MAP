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
2. Download the state maps from https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-1-states-provinces
3. Unzip the packages and update the input_files according to the shapefile.
4. Run the snippet
5. The output is in the downloaded folder


```python
import geopandas as gpd
import numpy as np
country_input_file = ...
states_file = ...

split_to_admin1 = {"US", "CA", "IN"}  # add more as needed (e.g., "AU", "BR", ...)
cols = ['NAME', 'NAME_LONG', 'ABBREV', 'NAME_CIAWF', 'ISO_A2_EH', 'ISO_A3_EH', 'CONTINENT', 'geometry']

a0 = gpd.read_file(country_input_file)   # ne_10m_admin_0_countries_*.shp
a1 = gpd.read_file(states_file)          # ne_10m_admin_1_states_provinces*.shp

# --- Admin-0 iso cols (keep your consistency) ---
iso2_a0 = "ISO_A2_EH" if "ISO_A2_EH" in a0.columns else "ISO_A2"
iso3_a0 = "ISO_A3_EH" if "ISO_A3_EH" in a0.columns else "ADM0_A3"

# --- Admin-1 country key (usually ADM0_A3) ---
adm0_a3_a1 = "ADM0_A3" if "ADM0_A3" in a1.columns else "adm0_a3"

# --- Build ISO2->ISO3 mapping from Admin-0 so we can filter Admin-1 by country ---
iso2_to_iso3 = (
    a0[[iso2_a0, iso3_a0]]
    .dropna()
    .drop_duplicates()
    .set_index(iso2_a0)[iso3_a0]
    .to_dict()
)
split_iso3 = {iso2_to_iso3[c] for c in split_to_admin1 if c in iso2_to_iso3}

# --- Filter the layers ---
a0_keep = a0[~a0[iso2_a0].isin(split_to_admin1)].copy()
a1_keep = a1[a1[adm0_a3_a1].isin(split_iso3)].copy()

# --- Helpers for Admin-1 region naming + ISO-3166-2 ---
def pick_first_col(df, candidates):
    return next((c for c in candidates if c in df.columns), None)

admin1_name_col = pick_first_col(a1_keep, ["name", "NAME", "name_en", "provname", "gn_name"])
admin1_iso3166_2_col = pick_first_col(a1_keep, ["iso_3166_2", "ISO_3166_2", "iso3166_2"])

# --- normalize to a schema that contains your Admin-0 columns (where applicable) ---
cols0_present = [c for c in cols if c in a0_keep.columns]
a0_keep = a0_keep.loc[:, cols0_present].copy()

# Admin-1 does not have all of those; we create them so concatenation is stable
for c in ["NAME", "NAME_LONG", "ABBREV", "NAME_CIAWF", "CONTINENT"]:
    if c not in a1_keep.columns:
        a1_keep[c] = None

# Attach ISO2/ISO3 for Admin-1 rows
a1_keep[iso3_a0] = a1_keep[adm0_a3_a1]  # store in same ISO3 col name as Admin-0 uses
iso3_to_iso2 = {v: k for k, v in iso2_to_iso3.items()}
a1_keep[iso2_a0] = a1_keep[iso3_a0].map(iso3_to_iso2)

# Put a region/state name into NAME (and NAME_LONG) for Admin-1 rows
if admin1_name_col:
    a1_keep["NAME"] = a1_keep[admin1_name_col]
    a1_keep["NAME_LONG"] = a1_keep[admin1_name_col]
else:
    a1_keep["NAME"] = None
    a1_keep["NAME_LONG"] = None

# For Admin-1, COUNTRY name should still come from parent country
# (use Admin-0 mapping; prefer NAME_CIAWF, else NAME)
a0_country_name_map = (
    a0.assign(_COUNTRY_NAME=a0.get("NAME_CIAWF", a0["NAME"]).fillna(a0["NAME"]))
      .set_index(iso2_a0)["_COUNTRY_NAME"]
      .to_dict()
)
a1_keep["NAME_CIAWF"] = a1_keep[iso2_a0].map(a0_country_name_map)

# Add an ISO-3166-2 style region code: e.g. "US-MI"
# Prefer Natural Earth's iso_3166_2 if present; otherwise best-effort fallback.
if admin1_iso3166_2_col:
    a1_keep["REGION_ISO"] = a1_keep[admin1_iso3166_2_col]
else:
    # fallback: if you have a postal abbrev, use it; otherwise slugify NAME
    postal_col = pick_first_col(a1_keep, ["postal", "POSTAL"])
    if postal_col:
        a1_keep["REGION_ISO"] = a1_keep[iso2_a0].fillna("") + "-" + a1_keep[postal_col].fillna("")
        a1_keep.loc[a1_keep["REGION_ISO"].str.endswith("-"), "REGION_ISO"] = None
    else:
        a1_keep["REGION_ISO"] = None

# Add LEVEL so you can tell country vs state
a0_keep["LEVEL"] = "country"
a1_keep["LEVEL"] = "admin1"

# Keep to the same column set (your cols + extras)
base_cols = ["NAME", "NAME_LONG", "ABBREV", "NAME_CIAWF", iso2_a0, iso3_a0, "CONTINENT", "LEVEL", "geometry"]
extra_cols = ["REGION_ISO"]  # new
for c in base_cols + extra_cols:
    if c not in a0_keep.columns:
        a0_keep[c] = None
    if c not in a1_keep.columns:
        a1_keep[c] = None

filtered_data = gpd.GeoDataFrame(
    pd.concat([a0_keep[base_cols + extra_cols], a1_keep[base_cols + extra_cols]], ignore_index=True),
    crs=a0.crs,
    geometry="geometry",
)

# --- Your existing logic, adapted to mixed admin levels ---

# Prefer NAME_CIAWF, fall back to NAME
filtered_data["COUNTRY_NAME"] = filtered_data["NAME_CIAWF"].fillna(filtered_data["NAME"])

# Compute lengths in meters for correct ordering
proj = filtered_data.estimate_utm_crs() if filtered_data.crs is not None else "EPSG:3857"
filtered_data["geom_length"] = filtered_data.to_crs(proj).geometry.length.fillna(0)

# Fix Åland label and ISO mapping
mask_ax = filtered_data[iso2_a0] == "AX"
filtered_data.loc[mask_ax, "COUNTRY_NAME"] = "Åland"
filtered_data.loc[mask_ax, iso2_a0] = "FI"

# Biggest per ISO2 among features (note: for split countries, biggest will be the biggest state)
g = filtered_data.groupby(iso2_a0, dropna=False)["geom_length"]
filtered_data["biggest"] = filtered_data["geom_length"].eq(g.transform("max"))

# Sort by ISO then length desc
filtered_data = (
    filtered_data
    .sort_values([iso2_a0, "geom_length"], ascending=[True, False], na_position="last")
    .reset_index(drop=True)
)
filtered_data["biggest"] = np.where(filtered_data['REGION_ISO'].notna(), True, filtered_data['biggest']) # for state level it should always be True for now (not really any split states
filtered_data['ISO_A2_EH'] = np.where(filtered_data['REGION_ISO'].notna(), filtered_data['REGION_ISO'].str.replace('-', ' - '), filtered_data['ISO_A2_EH'])
# Optional: ensure valid geometries
filtered_data["geometry"] = filtered_data.geometry.make_valid()
filtered_data.to_file(country_input_file.replace('shp', 'geojson'), driver='GeoJSON')


```
