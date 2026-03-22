# Session 2 — Build a pynxtools reader

**Duration:** 1 hour

**Goal:** Implement a small reader plugin that converts a mock HDF5 instrument file + a YAML metadata file into a validated NeXus output conforming to `NXdouble_slit`.


!!! tip
    This session contains a lot of information and exercises. The goal is of course to work through all of them, but depending on your familiarity with Python and NeXus/`pynxtools`, you may need longer.

    Try not to spend too much time with any step. There will be solutions given if you are stuck.

    There will be a fully filled NeXus file waiting at the end that you can use in Session 3, regardless of how far you make it here.     

---

## How a reader fits in

Here, we are building a small reader that writes a NeXus file compliant with `NXdouble_slit`. We are using the `MultiFormatReader` from `pynxtools`.

The architecture of the reader we want to build looks like this:

```ascii
mock_data.h5     --> handle_hdf5_file() --> self.hdf5_data --+
eln_data.yaml    --> handle_eln_file()  --> self.eln_data  --+
                                                             |
config_file.json <-------------------------------------------+
      |
      |  "@attrs:some/path"  --> get_attr(key, path)
      |  "@eln"              --> get_eln_data(key, path)
      |  "@data:array_name"  --> get_data(key, path)
      |
      v
  output.nxs  (validated against NXdouble_slit)
```

The idea is to separate the reading into three steps:

1. parsing raw measurement and ELN data (`handle_hdf5_file`, `handle_eln_file`)
2. mapping to NeXus data using a config file (`config_file.json`) and special callback methods ( `get_attr`, `get_eln_data`, `get_data`)
3. writing the resulting NeXus HDF5 file. 

The `MultiFormatReader` provides all the plumbing. You write the methods that know about your specific data.

---

## The reader class

Open `src/pynxtools_workshop/reader.py`. You will find the `DoubleSlitReader` class, which
inherits from `MultiFormatReader`:

```python
class DoubleSlitReader(MultiFormatReader):

    supported_nxdls = ["NXdouble_slit"]

    CONVERT_DICT = {"instrument": "INSTRUMENT[instrument]"}

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.hdf5_data = None   # populated by handle_hdf5_file
        self.eln_data  = None   # populated by handle_eln_file

        self.extensions = {     # routes each input file to its handler
            ".h5":   self.handle_hdf5_file,   # ← you implement this (Exercise 1)
            ".yaml": self.handle_eln_file,    # ← already implemented
            ".json": self.set_config_file,    # ← provided by MultiFormatReader
        }

    # ── file handlers ──────────────────────────────────────────────────
    def handle_hdf5_file(self, file_path): ...   # Exercise 1 — you implement
    def handle_eln_file(self, file_path):  ...   # already implemented

    # ── callbacks — called by the framework for each @token in config ──
    def get_eln_data(self, key, path): ...       # Exercise 4 — you implement
    def get_attr(self, key, path):     ...       # Exercise 3 — you implement
    def get_data(self, key, path):     ...       # Exercise 5 — you implement
```

**What `MultiFormatReader` does for you:**

1. Calls each handler registered in `self.extensions` for the matching input file.
2. Reads `config_file.json` and resolves every `@token` value by calling the matching callback on your class.
3. Writes the assembled dictionary to the output `.nxs` file and validates it against `NXdouble_slit`.

**What you implement:**

- `handle_hdf5_file` — load the HDF5 file and store a flat `{path: value}` dict on `self.hdf5_data` so that callbacks can look values up later.
- `get_attr`, `get_eln_data`, `get_data` — retrieve individual values when the framework resolves each token in the config file.

---

## Before you start — explore the data

Have a look at the data files `in tests/data/`. Either use VS Code (using the `h5web` extension ) or do it programatically:

```python
import h5py, yaml

# Inspect the HDF5 file
with h5py.File("tests/data/mock_data.h5", "r") as f:
    f.visititems(lambda name, obj: print(name, "→", type(obj).__name__))

# Inspect the ELN YAML
with open("tests/data/eln_data.yaml") as f:
    print(yaml.safe_load(f))
```

Have a look what `NXdouble_slit` defines/requires:

```bash
dataconverter generate-template --nxdl NXdouble_slit
dataconverter generate-template --nxdl NXdouble_slit --required
```

---

## Exercise 1 — `handle_hdf5_file`

**Goal:** Load the HDF5 file into `self.hdf5_data` as a flat dictionary `{path: value}`.

The result is stored on `self` so that the callback methods (`get_attr`, `get_data`) can look up values from it later when the framework processes the config file.

By default, `handle_...` shall return a dictionary (for reasons we will not go into here). We will just return an empty dict here.

Open `src/pynxtools_workshop/reader.py`. The stub looks like this:

```python
def handle_hdf5_file(self, file_path: str) -> None:
    """Load HDF5 data into self.hdf5_data as a flat dict {path: value}."""
    # TODO: implement
    return {}
```

Implement it using h5py.

You can test your solution by running:

```python
r = DoubleSlitReader()
r.handle_hdf5_file("tests/data/mock_data.h5")
```

??? tip
    ```python
    with h5py.File(file_path, "r") as f:
        result = {}
        def collect(name, obj):
            # TODO: implement collection logic here
        f.visititems(collect)
    self.hdf5_data = result
    ```
??? success "Full solution"

    ```python
    def handle_hdf5_file(self, file_path: str) -> None:
        import h5py
        result: dict[str, Any] = {}
        with h5py.File(file_path, "r") as f:
            def collect(name: str, obj: Any) -> None:
                if isinstance(obj, h5py.Dataset):
                    result[name] = obj[()]
            f.visititems(collect)
        self.hdf5_data = result
        return {}
    ```

**Check:** print `self.hdf5_data.keys()` — you should see paths like `data/detector_data`, `data/x_pixels`, `metadata/instrument/source/wavelength`, etc.

!!! tip
    It may seem unintuitive to first parse the HDF5 data, transform it, and then write it back to a NeXus file. We are doing this for learning purposes only; if you want to either transfer or link data from an HDF5 file to a NeXus file, `pynxtools` provides a specialized reader (called [`JsonMap`](https://fairmat-nfdi.github.io/pynxtools/reference/built-in-readers.html#the-jsonmapreader){:target="_blank" rel="noopener"}). You will learn more about it in the challenges on day 2.

---

## Exercise 2 — understand `handle_eln_file`

**Goal:** Understand how we load the ELN YAML file and convert it to flat template paths using `parse_yml`.

Not all metadata concepts defined in `NXdouble_slit` can be filled from the HDF5 file. We will add an additional metadata file. We call this the ELN file, since this is data typically recorded in an electronic lab notebook (ELN).

The ELN YAML uses lowercase keys that mirror the NeXus path structure:

```yaml
title: Double-slit interference experiment
start_time: "2026-03-22T10:00:00+01:00"
instrument:
  source:
    type: Laser
  double_slit:
    material: aluminum
```

We have already implemented the method for parsing YAML ELN files in `reader.py`. It is called automatically by `MultiFormatReader` because `.yaml` is registered in `self.extensions`. The implementation uses `parse_yml` from `pynxtools`, which parses the YAML file and wraps everything under `/ENTRY[entry]/…`. Keys that differ from the NeXus path name must be listed in `CONVERT_DICT`. Here only `instrument` needs renaming:

```python
CONVERT_DICT = {
    "instrument": "INSTRUMENT[instrument]",
}
```

This is the function that we implemented:

```python
from pynxtools.dataconverter.helpers import parse_yml

def handle_eln_file(self, file_path: str) -> None:
    self.eln_data = parse_yml(
        file_path,
        convert_dict=self.CONVERT_DICT,
        parent_key="/ENTRY",
    )
```

Try to understand what it would produce, i.e., how `self.eln_data` would look like.

??? success "Full solution"

    ```python
    {
        "/ENTRY[entry]/title": "Double-slit interference experiment",
        "/ENTRY[entry]/INSTRUMENT[instrument]/source/type": "Laser",
        "/ENTRY[entry]/INSTRUMENT[instrument]/double_slit/material": "aluminum"
    }
    ```

You can test the function by running:

```python
r = DoubleSlitReader()
r.handle_eln_file("tests/data/eln_data.yaml")
```

---

## How tokens and callbacks work

Before implementing the callbacks, it is important to understand how they are called.

Each value in `config_file.json` is either a literal (e.g. `"NXdouble_slit"`) or a **token**
starting with `@`. When `MultiFormatReader` processes the config file, it resolves each token
by calling a method on your reader class:

| Token | Framework calls | Argument passed as `path` |
|---|---|---|
| `"@eln"` | `get_eln_data(key, path)` | empty string — look up by `key` instead |
| `"@attrs:some/hdf5/path"` | `get_attr(key, path)` | `"some/hdf5/path"` |
| `"@data:array_name"` | `get_data(key, path)` | `"array_name"` |

In every case, `key` is the full NeXus template path — the left-hand side of the config entry.
`path` is the token suffix after the `:`.

This asymmetry is why `get_eln_data` looks up by `key` (because `parse_yml` already stored
values under full template paths) while `get_attr` and `get_data` look up by `path` (because
the HDF5 flat dict uses the raw HDF5 path as its key).

---

## Exercise 3 — `get_attr`

**Goal:** Return instrument metadata from `self.hdf5_data` by `path`.

The config file will contain entries like:

```json
"/ENTRY[entry]/INSTRUMENT[instrument]/source/wavelength": "@attrs:metadata/instrument/source/wavelength"
```

When the reader sees `@attrs:metadata/instrument/source/wavelength`, it calls:

```python
get_attr(key="/ENTRY[entry]/.../wavelength", path="metadata/instrument/source/wavelength")
```

Implement `get_attr` to look up `path` in `self.hdf5_data`:

```python
def get_attr(self, key: str, path: str) -> Any:
    # TODO: implement
    pass
```

??? success "Full solution"

    ```python
    def get_attr(self, key: str, path: str) -> Any:
        if self.hdf5_data is None:
            return None
        return self.hdf5_data.get(path)
    ```

---

## Exercise 4 — `get_eln_data`

**Goal:** Return metadata from `self.eln_data` by the full NeXus template path (`key`).

For ELN data, `parse_yml` already produces flat dictionary keys that are full template paths. So look up by `key`, not `path`:

```python
def get_eln_data(self, key: str, path: str) -> Any:
    # TODO: implement
    pass
```

??? success "Full solution"

    ```python
    def get_eln_data(self, key: str, path: str) -> Any:
        if self.eln_data is None:
            return None
        return self.eln_data.get(key)
    ```

---

## Exercise 5 — `get_data`

**Goal:** Return measurement arrays from `self.hdf5_data`.

Data arrays live under `data/` in the HDF5 file. For example, `data/detector_data`, `data/x_offset`, and `data/interference_data`. The `path` argument is the dataset name inside `data/`, so look up `f"data/{path}"`.

```python
def get_data(self, key: str, path: str) -> Any:
    # TODO: implement
    pass
```

??? success "Full solution"

    ```python
    def get_data(self, key: str, path: str) -> Any:
        if self.hdf5_data is None:
            return None
        return self.hdf5_data.get(f"data/{path}")
    ```

---

## ✅ Checkpoint — test the callbacks

```python
from pynxtools_workshop.reader import DoubleSlitReader

r = DoubleSlitReader()
r.handle_hdf5_file("tests/data/mock_data.h5")
r.handle_eln_file("tests/data/eln_data.yaml")

print(r.get_attr("", "metadata/instrument/source/wavelength"))  # → 532.0
print(r.get_eln_data("/ENTRY[entry]/title", ""))                 # → Double-slit interference experiment
print(r.get_data("", "detector_data").shape)                     # → (200, 100)
```

---

## Exercise 6 — Write the config file

The config file is the **semantic bridge** between your data and the NeXus template. Each key is a template path; each value tells the reader where to find the data.

If a callback returns `None` for a given key, that path is left unfilled. Required paths that remain `None` will trigger a validation warning or error; optional paths are silently skipped.

!!! tip
    Writing config files requires understanding their logic. You will likely not be able to fill out the whole config file in time. There is a prefilled `config_file.json` in `tests/data` that you can use.

### Step 1 — generate the template

```bash
dataconverter generate-template --nxdl NXdouble_slit >> config_file.json
```

### Step 2 — fill in the values

For each required or recommended path, decide how to fill it:

| Data source | Config value |
|---|---|
| HDF5 metadata | `"@attrs:metadata/instrument/source/wavelength"` |
| ELN YAML | `"@eln"` |
| Measurement array | `"@data:detector_data"` |
| Fixed literal | `"NXdouble_slit"` |

Example config fragment:

```json
{
  "/ENTRY[entry]/definition":"NXdouble_slit",
  "/ENTRY[entry]/title":"@eln",
  "/ENTRY[entry]/start_time":"@eln",
  "/ENTRY[entry]/INSTRUMENT[instrument]/source/wavelength":"@attrs:metadata/instrument/source/wavelength",
  "/ENTRY[entry]/INSTRUMENT[instrument]/source/wavelength/@units":"nm",
  "/ENTRY[entry]/INSTRUMENT[instrument]/detector/DATA[data]/data":"@data:detector_data",
  "/ENTRY[entry]/interference_pattern/data":"@data:interference_data",
  "/ENTRY[entry]/interference_pattern/x_offset":"@data:x_offset",
  "/ENTRY[entry]/interference_pattern/x_offset/@units":"mm"
}
```

You can find the full solution in `tests/data/config_file.json`.

## ✅ Run the conversion

```bash
dataconverter \
  --reader workshop \
  --nxdl NXdouble_slit \
  --config tests/data/config_file.json \
  --output output.nxs
  tests/data/mock_data.h5 \
  tests/data/eln_data.yaml \
```

Inspect the result. Use either `h5web` in VS Code or run:

```bash
python3 -c "
import h5py
with h5py.File('output.nxs', 'r') as f:
    f.visititems(lambda n, obj: print(n))
"
```

---

## ✅ Solution and correct NeXus file

You can find the full reader implementation here:

- [`reader.py`]()

As promised at the top, you can continue to the next session even if you have not yet finished writing the reader or the config file. Download the final NeXus file here:

[Download result.nxs](../downloads/result.nxs){:target="_blank" .md-button}

---

## Summary

| Exercise | What you built | Key concept |
|---|---|---|
| 1 | `handle_hdf5_file` | Flat dict from HDF5 |
| 2 | `handle_eln_file` + `CONVERT_DICT` | `parse_yml` maps YAML → template paths |
| 3 | `get_attr` | `@attrs:path` dispatch |
| 4 | `get_eln_data` | `@eln` dispatch uses `key`, not `path` |
| 5 | `get_data` | `@data:name` dispatch |
| 6 | `config_file.json` | Semantic mapping: source → NeXus |

---

## Further reading

- [pynxtools tutorial > Build your first reader](https://fairmat-nfdi.github.io/pynxtools/tutorial/build-a-reader/){:target="_blank" rel="noopener"}
- [pynxtools how-to > Use MultiFormatReader](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/use-multi-format-reader/){:target="_blank" rel="noopener"}
- [pynxtools how-to > Build a plugin](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/build-a-plugin/){:target="_blank" rel="noopener"}
