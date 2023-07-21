## Setting Up a working HIPSCAT/LSDB python environment on epyc
`bash` commands to get a working demo lsdb environment.

```bash
#create python environment
conda create -n lsdb_demo python=3.10
source activate lsdb_demo

#clones directory
cd ~
mkdir hipscat_clones
cd hipscat_clones

#clone demo branch of hipscat
git clone https://github.com/astronomy-commons/hipscat
cd hipscat
git checkout origin
git checkout sean/tutorial-demo
python -m pip install -e .
cd ../

#clone demo branch of lsdb
git clone https://github.com/astronomy-commons/lsdb
cd lsdb
git checkout origin
git checkout sean/tutorial-demo
python -m pip install -e .

#other python setup dependencies
ipython kernel install --user --name=lsldb_demo
python -m pip install ray ipywidgets
```

## Available datasets on epyc
All hipscats are located in the `/data3/epyc/data3/hipscat` directory. There is a README on those datasets, and as of today (07/21/2023) it read:

```
README
=============

Useful contents of the /data3/epyc/data3/hipscat/ directory.

`catalogs/`
-----------

All catalogs within this directory should be "hipscatted" (processed in hipscat format)

**Likely OK to use**

If you run into ANY problems with these catalogs, PLEASE reach out to
delucchi@andrew.cmu.edu or LSSTC slack #hipscat-users

- `dr16q` - known AGNs, pulled from SDSS
- `dr16q_constant` - same data as above, partitioned at a constant healpix order of 5
- `ps1/`
    - PanStarrs data, retrieved from MAST
    - `ps1_otmo` - object table
    - `ps1_detection` - detections (source)
- `ztf_axs/`
    - ZTF catalog, using data originally matched/calibrated with AXS
    - `ztf_dr14` - the object table
    - `ztf_source` - detections (source)
        - transformed to one row per detection
        - co-partitioned with `ztf_dr14`
- `zubercal` - ubercalibratedd ZTF. missing some g-band detections.
- `allwise` - allwise object catalog
- `neowise_yr8` - neowise year 8. source catalog for allwise.
- `tic_1` - TESS input catalog. objects only

**NOT ok to use**

- `hst/`
    - Hubble Space Telescope - no usable data here yet
- `ztf_dr14` - DEPRECATED. Please don't use! We're keeping this around because it
    causes interesting failures. Do you want those interesting failures to happen
    to YOU?!

`raw/`
-------
- allwise_raw - downloaded from IRSA. raw csv data from ALLWISE.
- neowise_raw - downloaded from IRSA. raw csv.

`tmp/`
-----------
Used for temp storage during pipeline execution. Useful to have it on `/data3/`
```

## Using LSDB
lsdb python api examples. We will:
* load ztf and gaia objects
* cull our gaia objects with a cone search
* then join with ztf_sources to get lightcurve data

### Set up a ray client
```python
import ray
from ray.util.dask import enable_dask_on_ray, disable_dask_on_ray
client = ray.init(
    num_cpus = 4
)
enable_dask_on_ray()
```

### Loading hipscats
```python
import lsdb
import numpy as np
gaia = lsdb.read_hipscat("/data3/epyc/projects3/ivoa_demo/gaia/catalog")
ztf = lsdb.read_hipscat("/data3/epyc/data3/hipscat/catalogs/ztf_axs/ztf_dr14")
#sources load takes a minute, since it creates a healpix alignment on load
ztf_sources = lsdb.read_hipscat("/data3/epyc/data3/hipscat/catalogs/ztf_axs/ztf_source")
```

### Performing conesearches
```python
result = gaia.cone_search(
    ra=30,
    dec=30,
    radius=1,
)

result.compute()
```

### Performing crossmatches
```python
xmatch = ztf.crossmatch(result)
xmatch.compute()
```

### Performing joining to sources
```python
join = xmatch.join(
    ztf_sources, left_on="ps1_objid_ztf_dr14", right_on="ps1_objid"
).compute()
```

### Cull our join for objects with comprehensive lightcurve data
```python
join.query(
    "nobs_g_ztf_dr14 > 20 and nobs_r_ztf_dr14 > 20 and nobs_i_ztf_dr14 > 20"
)
```

### Shutdown ray
```python
disable_dask_on_ray()
ray.shutdown()
```
