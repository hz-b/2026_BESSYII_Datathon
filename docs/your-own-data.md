# Bring your own data

**Duration:** ~3 hours (self-paced, with instructor support)

**Goal:** Apply the `MultiFormatReader` pattern from Day 1 to your own instrument data and produce a validated NeXus file.

**What to bring:** one or more raw data files from your instrument.

---

## The core idea — always the same three steps

No matter what format your data comes in, the workflow is identical to Day 1:

```
Step A  Read your file(s) into a flat Python dict on self
Step B  Write a config file that maps dict keys → NeXus paths
Step C  Run the converter and fix validation errors
```

The only part that changes between techniques and formats is **Step A** — the reading logic. Steps B and C are identical to Day 1.

---

## Step 0 — Setup (~10 min)

Repeat the steps shown in the [setup of Day 1](example_pipeline/0-setup.md/#4-instantiate-the-plugin-template). The goal is to have another `pynxtools` reader plugin
instantiated with the `pynxtools-plugin-template`.

---

## Step 1 — Know your format (~30 min)

Before writing any reader code, understand what you are working with.

### Identify your format

| Format | Typical extensions | How to recognise |
|---|---|---|
| HDF5 / NeXus | `.h5`, `.hdf5`, `.nxs` | Binary; starts with `\x89HDF` |
| HDF5 (instrument brand) | `.h5m`, `.hsp`, `.he5`, … | Same magic bytes; vendor-specific internal layout |
| VAMAS | `.vms`, `.vamas` | First line: `VAMAS Surface Chemical Analysis` |
| Igor Pro wave | `.ibw` | Binary with `IGOR` header |
| CSV / TSV | `.csv`, `.txt`, `.dat`, `.asc` | Human-readable columns |
| JSON | `.json` | `{` or `[` as first non-whitespace character |
| YAML | `.yaml`, `.yml` | Key-value pairs with indentation |
| NetCDF | `.nc`, `.cdf`, `.netcdf` | Binary; readable with `netCDF4` or `xarray` |
| TIFF (detector images) | `.tiff`, `.tif` | Binary image; use `tifffile` or `PIL` |

### Explore the data before coding

```python
import h5py

# For HDF5 files
with h5py.File("your_file.h5", "r") as f:
    f.visititems(lambda name, obj: print(name, "→", type(obj).__name__))

# For text/CSV files
with open("your_file.csv") as f:
    for i, line in enumerate(f):
        print(line.rstrip())
        if i > 20:
            break
```

Take 10 minutes to understand the structure before writing any code.

---

## Step 2 — Implement `handle_*_file` (~45 min)

Pick the section below that matches your format and implement the corresponding handler.

### HDF5 (any vendor)

This is the same recursive reader from Day 1. It works for any HDF5 file — vendor-specific layouts, NeXus files, everything.

```python
import h5py
from typing import Any

def handle_hdf5_file(self, file_path: str) -> None:
    result: dict[str, Any] = {}

    def collect(name: str, obj: Any) -> None:
        if isinstance(obj, h5py.Dataset):
            result[name] = obj[()]
        # optionally capture attributes:
        for k, v in obj.attrs.items():
            result[f"{name}/@{k}"] = v

    with h5py.File(file_path, "r") as f:
        f.visititems(collect)

    self.hdf5_data = result
```

After running, print the keys to understand what is available:

```python
r.handle_hdf5_file("your_file.h5")
for k in sorted(r.hdf5_data):
    print(k)
```

---

### CSV / TSV / columnar text

```python
import numpy as np
from typing import Any

def handle_csv_file(self, file_path: str) -> None:
    # Adjust delimiter, skiprows, and encoding for your file
    data = np.genfromtxt(
        file_path,
        delimiter=",",    # "\t" for TSV, None for whitespace
        names=True,       # use first row as column names
        encoding="utf-8",
    )
    self.data = {name: data[name] for name in data.dtype.names}
```

Or with pandas for messy headers:

```python
import pandas as pd
from typing import Any

def handle_csv_file(self, file_path: str) -> None:
    meta: dict[str, Any] = {}
    data_start = 0
    with open(file_path) as f:
        for i, line in enumerate(f):
            if line.startswith("#"):
                key, _, value = line[1:].partition("=")
                meta[key.strip()] = value.strip()
            else:
                data_start = i
                break

    df = pd.read_csv(file_path, skiprows=data_start, comment="#")
    self.data = {col: df[col].to_numpy() for col in df.columns}
    self.data.update(meta)
```

---

### VAMAS (`.vms`)

VAMAS is common for XPS and other surface science data.

```python
from typing import Any

def handle_vamas_file(self, file_path: str) -> None:
    try:
        from vamas import Vamas
    except ImportError:
        raise ImportError("pip install vamas")

    vms = Vamas(file_path)
    block = vms.blocks[0]   # first spectrum; iterate for multiple

    self.data = {
        "kinetic_energy":  block.x,
        "intensity":       block.y,
        "source_energy":   block.source_energy,
        "pass_energy":     block.analyser_pass_energy,
        "dwell_time":      block.signal_collection_time,
        "sample_id":       block.sample_id,
        "technique":       block.technique,
        "comment":         block.comment,
    }
```

---

### Igor Pro IBW (`.ibw`)

```python
import numpy as np
from typing import Any

def handle_ibw_file(self, file_path: str) -> None:
    import igor2.igorpy as igor

    wave = igor.load(file_path)
    self.data = {"data": wave.data}

    # axis scaling
    for dim, (offset, delta) in enumerate(zip(wave.sfB, wave.sfA)):
        n = wave.data.shape[dim]
        self.data[f"axis_{dim}"] = offset + delta * np.arange(n)

    # JSON-encoded note (common in Scienta files)
    import json
    try:
        meta = json.loads(wave.notes.decode())
        for k, v in meta.items():
            self.data[f"meta/{k}"] = v
    except (json.JSONDecodeError, AttributeError):
        pass
```

---

### NetCDF (`.nc`)

```python
from typing import Any

def handle_netcdf_file(self, file_path: str) -> None:
    import xarray as xr

    ds = xr.open_dataset(file_path)
    self.data = {}
    for var in ds.data_vars:
        self.data[var] = ds[var].values
    for coord in ds.coords:
        self.data[f"axis/{coord}"] = ds.coords[coord].values
    for k, v in ds.attrs.items():
        self.data[f"attrs/{k}"] = v
```

---

### TIFF / detector images

```python
from typing import Any

def handle_tiff_file(self, file_path: str) -> None:
    import tifffile

    with tifffile.TiffFile(file_path) as tif:
        data = tif.asarray()   # (frames, H, W) or (H, W)
        meta = tif.imagej_metadata or {}
        if not meta and tif.pages[0].tags:
            meta = {t.name: t.value for t in tif.pages[0].tags.values()}

    self.data = {"detector/image": data}
    self.data.update({f"meta/{k}": v for k, v in meta.items()})
```

---

### Anything else — the fallback pattern

```python
from typing import Any

def handle_my_format(self, file_path: str) -> None:
    self.data = {}

    with open(file_path, "rb") as f:   # or "r" for text
        raw = f.read()

    # --- parse raw bytes or text here ---
    # e.g. use struct, regex, or your vendor's SDK

    self.data["signal"] = ...
    self.data["energy_axis"] = ...
    self.data["sample_name"] = ...
```

Then register the extension in `__init__`:

```python
self.extensions[".myext"] = self.handle_my_format
```

---

## Step 3 — Update the callbacks (~20 min)

If you used `self.data` (not `self.hdf5_data`), update the three callbacks:

```python
from typing import Any

def get_attr(self, key: str, path: str) -> Any:
    if self.data is None:
        return None
    value = self.data.get(path)
    if isinstance(value, bytes):
        return value.decode()
    return value

def get_eln_data(self, key: str, path: str) -> Any:
    if self.eln_data is None:
        return None
    return self.eln_data.get(key)

def get_data(self, key: str, path: str) -> Any:
    if self.data is None:
        return None
    return self.data.get(path)
```

---

## Step 4 — Find your application definition (~20 min)

### Does one already exist?

Check whether a community definition exists for your technique:

| Technique | Application definition | Plugin |
|---|---|---|
| XPS | `NXxps` | `pynxtools-xps` |
| ARPES / multi-photon | `NXmpes`, `NXmpes_arpes`, `NXarpes` | `pynxtools-mpes` |
| Raman | `NXraman` | `pynxtools-raman` |
| Ellipsometry | `NXellipsometry` | `pynxtools-ellips` |
| Electron microscopy | `NXem` | `pynxtools-em` |
| X-ray diffraction | `NXxrd` | `pynxtools-xrd` |
| Generic / workshop | `NXsimple` | this workshop |

Test whether it is installed:

```bash
dataconverter generate-template --nxdl NXmpes
```

### No definition? Write a minimal one.

Use the skills from [Session 2](example_pipeline/1-appdef.md). Start with the smallest possible skeleton:

```yaml
# NXmytechnique.yaml
category: application
doc: Application definition for my technique.
type: group
NXmytechnique(NXobject):
  (NXentry):
    definition:
      enumeration: [NXmytechnique]
    title:
    (NXinstrument):
      name(NX_CHAR):
    (NXsample):
      name(NX_CHAR):
    (NXdata):
```

Convert it:

```bash
nyaml2nxdl NXmytechnique.yaml --output-file NXmytechnique.nxdl.xml
```

In order to use your application definitions directly, you will need to add to the NeXus defintitions stored in `pynxtools`. For this, you need to install `pynxtools` in editable mode. You can learn more in the `pynxtools` [development guide](https://fairmat-nfdi.github.io/pynxtools/tutorial/contributing.html#development-installation).

Install `pynxtools` with the `-e` option in the same virtual environment that you are already working in. Instantiate the `definitions` submodule.

Then you can place your application definition NXDL XML file in `pynxtools`:

```bash

cp NXmytechnique.nxdl.xml src/pynxtools/definitions/contributed_definitions/
dataconverter generate-template --nxdl NXmytechnique
```

---

## Step 5 — Write the config file (~40 min)

Generate the template first:

```bash
dataconverter generate-template --nxdl <YOUR_NXDL> > config.json
```

For each path in the output, fill in the config:

| Where is the value? | Config value |
|---|---|
| `self.data["some/key"]` or `self.hdf5_data["some/key"]` | `"@attrs:some/key"` |
| `self.eln_data["/ENTRY[entry]/..."]` | `"@eln"` |
| `self.data["signal_array"]` | `"@data:signal_array"` |
| Fixed constant | `"eV"` or `532` |

Learm more about the config file in the [`pynxtools` documentation for the `MultiFormatReader`](https://fairmat-nfdi.github.io/pynxtools/learn/pynxtools/multi-format-reader.html#parsing-the-config-file).

---

## Step 6 — Convert, validate, iterate (~20 min)

```bash
dataconverter \
    your_file.ext \
    eln_data.yaml \
    config_file.json \
    --reader <your-reader> \
    --nxdl <YOUR_NXDL> \
    --output output.nxs
```

Read the output messages:

| Level | Meaning | Action |
|---|---|---|
| **ERROR** | Required field missing | Add to config or provide ELN data |
| **WARNING** | Recommended field missing | Add if possible |
| **INFO** | Optional field missing | Safe to skip |

Inspect the result:

```python
import h5py
with h5py.File("output.nxs", "r") as f:
    f.visititems(lambda n, o: print(n))
```

Repeat until no errors remain.

---

## Common errors and fixes

| Error / symptom | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: <vendor lib>` | Library not installed | `pip install <library>` |
| `KeyError: 'some/path'` in callback | Path missing from `self.data` | `print(sorted(self.data.keys()))` to find the right key |
| Required field missing in output | Config doesn't map it | Add the path to config file |
| `bytes` in output string field | h5py byte string | Add `.decode()` in the callback |
| All `get_eln_data` return `None` | Wrong CONVERT_DICT keys | Print `self.eln_data.keys()` vs the `key` argument |
| Validation passes but file looks incomplete | Application definition has no required fields | Add `required` fields to the NXDL |

---

## Checklist before you leave

- [ ] `dataconverter` runs without errors on your own data
- [ ] All required fields are present in `output.nxs`
- [ ] Units are set for every numeric field
- [ ] `reader.py` and `config_file.json` are committed to your repository
- [ ] You know which application definition matches your technique (or have written a minimal one)

---

## Further reading

- [pynxtools tutorial > Build a reader](https://fairmat-nfdi.github.io/pynxtools/tutorial/build-a-reader/)
- [pynxtools how-to > Use the MultiFormatReader](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/use-multi-format-reader/)
- [pynxtools how-to > Build a plugin](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/build-a-plugin/)
- [pynxtools reference > Available plugins](https://fairmat-nfdi.github.io/pynxtools/reference/plugins/)
- [NeXus application definitions](https://fairmat-nfdi.github.io/nexus_definitions/)
