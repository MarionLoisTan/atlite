# Changes for Damage Feature in PyPSA-Eur

## 1. What was added and where

All changes live in a fork of atlite on the `damage-features` branch.

### `atlite/datasets/era5.py`

Three new ERA5 retrieval functions, each registered in the module-level
`features` dict so they are available via `cutout.prepare()` without any
runtime patching:

| Function | Feature key | Output variable |
|---|---|---|
| `get_data_wind_gust` | `"wind_gust"` | `wnd_gust10m` |
| `get_data_lake_s_temperature` | `"lake_s_temperature"` | `lake_s_temp` |
| `get_data_lake_t_temperature` | `"lake_t_temperature"` | `lake_t_temp` |

`open_with_grib_conventions` was also updated with a mixed-dataType fallback.
Some ERA5 variables are stored with both `dataType='an'` (analysis) and
`dataType='fc'` (forecast) in the same GRIB file, which causes cfgrib to raise
`DatasetBuildError`. The function now catches this and retries by filtering
each type separately before concatenating — this applies to all variables
automatically, not just wind gust.

### `atlite/cutout.py`

Two utility methods added to the `Cutout` class:

- **`copy(dest)`** — copies the cutout's NetCDF file to `dest`, creates
  parent directories if needed, and returns a new `Cutout` pointing to the
  copy. This is the entry point for creating a derived cutout before calling
  `prepare()`.
- **`open_dataset()`** — opens the cutout as an `xarray.Dataset` with chunks
  aligned to the on-disk `chunksize_time` attribute, avoiding Dask performance
  issues from misaligned chunk sizes.

---

## 2. Setting up the forked atlite in PyPSA-Eur

The fork is installed as an editable PyPI dependency so changes to the source
are picked up immediately.

In `pixi.toml`, remove atlite from `[dependencies]` (conda-forge) if present,
and add it under `[pypi-dependencies]`:

```toml
[pypi-dependencies]
atlite = { path = "../atlite", editable = true }
```

The path is relative to the `pixi.toml` file. Adjust if your directory layout
differs. After editing, run:

```bash
pixi install
```

Verify the correct version is active:

```python
import atlite
print(atlite.__file__)   # should point to the fork, not site-packages
```

> **Note:** The `doc` feature in `pixi.toml` pins `atlite = "==0.4.1"` for
> documentation builds. That environment is isolated and unaffected by the
> fork.

---

## 3. Usage

### Standalone (plain Python)

```python
import atlite

cutout = atlite.Cutout(path="cutouts/europe-2013.nc")
derived = cutout.copy("cutouts/custom/europe-2013_fg10_lmlt.nc") # naming of the copy is up to user, not automated
derived.prepare(features=["wind_gust", "lake_s_temperature"])
```

No patching or helper scripts required — the features are registered natively
in the fork.

### Snakemake (PyPSA-Eur workflow)

The rule `build_damage_cutout` in `rules/damage.smk` handles
the copy-and-prepare step. Configure it via `config/damage_config.yaml`:

```yaml
cutout_dir: cutouts/          # absolute, ~/..., or relative to pypsa-eur root
cutout_names:                 # explicit list, or omit to process all .nc files
  - europe-2013-era5
features:
  - wind_gust
  - lake_s_temperature
feature_shortcodes:           # ERA5 GRIB shortcode per feature — used to name output files
  wind_gust: fg10
  lake_s_temperature: lmlt
  lake_t_temperature: ltlt
```

`feature_shortcodes` maps each feature to its ERA5 GRIB shortcode. The
Snakemake rule reads this to build the output filename suffix (e.g.
`europe-2013-era5_fg10_lmlt.nc`). Every feature listed under `features` must
have a corresponding entry here.

Run the convenience target to prepare all listed cutouts:

```bash
snakemake build_all_damage_cutouts --configfile config/config.yaml -j4
```

Output files are written to `cutouts/custom/` with a filename that encodes the
added features via their ERA5 shortcodes (e.g. `europe-2013-era5_fg10_lmlt.nc`).

---

## 4. Adding new variables in the future

1. Add a retrieval function to `atlite/datasets/era5.py` following the naming
   convention `get_data_{feature_name}`:

   ```python
   def get_data_my_variable(retrieval_params):
       ds = retrieve_data(variable=["cds_variable_name"], **retrieval_params)
       ds = _rename_and_clean_coords(ds)
       ds = ds.rename({"grib_shortname": "my_output_var"})
       return ds
   ```

2. Register it in the `features` dict near the top of `era5.py`:

   ```python
   features["my_variable"] = ["my_output_var"]
   ```

3. Add the ERA5 GRIB shortcode to `feature_shortcodes` in
   `config/damage_config.yaml` so the output filename is built correctly:

   ```yaml
   feature_shortcodes:
     ...
     my_variable: shortcode   # ERA5 GRIB shortname for this variable
   ```

atlite dispatches `get_data_*` functions by name via `globals()`, so the
function is available via `cutout.prepare(features=["my_variable"])` as soon as
it is defined and registered.

---

## 5. Caveats

- **Additional Processing for variables.** The get_data_my_variable() function is just an example of the basic way to get a variable from CDS. Looking at the era5.py script, it is possible that additional processing and sanitizing is still needed for your purposes (ex. get_data_wind(), get_data_influx()).  
- **ERA5 only.** The new features and the mixed-data type fix are implemented
  exclusively for the ERA5 dataset module. Other atlite dataset backends
  (SARAH, NCEP, CORDEX) are unaffected and untested with these additions.

- **Mixed-dataType handling is opportunistic.** The fallback in
  `open_with_grib_conventions` catches any exception from the initial
  `xr.open_dataset` call and retries with `dataType` filtering. This is
  intentionally broad to handle the `an`/`fc` split seen in wind gust and lake
  temperature variables, but it could mask unrelated GRIB parsing errors.

- **CDS API required.** `cutout.prepare()` downloads data from the Copernicus
  Climate Data Store. A valid CDS API key must be configured before use.
