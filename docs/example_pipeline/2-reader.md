# Session 2 — Build a pynxtools reader

**Duration:** 1 hours

**Goal:** Implement a small reader plugin that converts a mock HDF5 instrument file + a YAML metadata file into a validated NeXus output conforming to `NXsimple`.

---

## How a reader fits in

```
mock_data.h5     ──► handle_hdf5_file()  ──►  self.hdf5_data  ──┐
eln_data.yaml    ──► handle_eln_file()   ──►  self.eln_data   ──┤
                                                                  │
config_file.json ◄────────────────────────────────────────────┘
      │
      │  "@attrs:some/path"  ──►  get_attr(key, path)
      │  "@eln"              ──►  get_eln_data(key, path)
      │  "@data:array_name"  ──►  get_data(key, path)
      │
      ▼
  output.nxs  (validated against NXsimple)
```

`MultiFormatReader` provides all the plumbing. You write the four methods that know about your specific data.

---

## Before you start — explore the data

```python
import h5py, yaml

# Inspect the HDF5 file
with h5py.File("tests/data/workshop-example/mock_data.h5", "r") as f:
    f.visititems(lambda name, obj: print(name, "→", type(obj).__name__))

# Inspect the ELN YAML
with open("tests/data/workshop-example/eln_data.yaml") as f:
    print(yaml.safe_load(f))

# See what NXsimple requires
dataconverter generate-template --nxdl NXsimple
```

Take 10 minutes to understand the data structure before writing any code.

---

## Exercise 1 — `handle_hdf5_file` (30 min)

**Goal:** Load the HDF5 file into `self.hdf5_data` as a flat dictionary `{path: value}`.

Open `src/pynxtools_simple/reader.py`. The stub looks like this:

```python
def handle_hdf5_file(self, file_path: str) -> None:
    """Load HDF5 data into self.hdf5_data as a flat dict {path: value}."""
    # TODO: implement
    pass
```

Implement it using h5py. For each dataset in the file, store `name → np.array_or_scalar`:

```python
with h5py.File(file_path, "r") as f:
    result = {}
    def collect(name, obj):
        if isinstance(obj, h5py.Dataset):
            result[name] = obj[()]   # [()] reads the full value
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
    ```

**Check:** print `self.hdf5_data.keys()` — you should see the HDF5 paths as keys.

---

## Exercise 2 — `handle_eln_file` (25 min)

**Goal:** Load the ELN YAML file and convert it to flat template paths using `parse_yml`.

The ELN YAML file has a structure that maps to NeXus paths:

```yaml
ENTRY:
  USER:
    name: John Doe
  title: My experiment
```

`parse_yml` converts this to:
```python
{"/ENTRY[entry]/USER[user]/name": "John Doe",
 "/ENTRY[entry]/title": "My experiment"}
```

You also need a `CONVERT_DICT` to rename your YAML keys to NeXus class names:

```python
CONVERT_DICT = {
    "ENTRY": "ENTRY[entry]",
    "USER": "USER[user]",
    "INSTRUMENT": "INSTRUMENT[instrument]",
    "SAMPLE": "SAMPLE[sample]",
}
```

Implement the handler:

```python
def handle_eln_file(self, file_path: str) -> None:
    """Load ELN YAML into self.eln_data as flat NeXus template paths."""
    # TODO: implement using parse_yml and CONVERT_DICT
    pass
```

??? success "Full solution"

    ```python
    from pynxtools.dataconverter.helpers import parse_yml

    CONVERT_DICT = {
        "ENTRY": "ENTRY[entry]",
        "USER": "USER[user]",
        "INSTRUMENT": "INSTRUMENT[instrument]",
        "SAMPLE": "SAMPLE[sample]",
    }

    def handle_eln_file(self, file_path: str) -> None:
        self.eln_data = parse_yml(
            file_path,
            convert_dict=self.CONVERT_DICT,
            entry_name="entry",
        )
    ```

---

## Exercise 3 — `get_attr` (20 min)

**Goal:** Return instrument metadata from `self.hdf5_data` by `path`.

The config file will contain entries like:

```json
"/ENTRY[entry]/INSTRUMENT[instrument]/SOURCE[source]/wavelength": "@attrs:metadata/source/wavelength"
```

When the reader sees `@attrs:metadata/source/wavelength`, it calls:

```python
get_attr(key="/ENTRY[entry]/.../wavelength", path="metadata/source/wavelength")
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

## Exercise 4 — `get_eln_data` (15 min)

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

## Exercise 5 — `get_data` (20 min)

**Goal:** Return measurement arrays from `self.hdf5_data`.

Data arrays live under `data/` in the HDF5 file — for example, `data/x_values` and `data/y_values`. The `path` argument will be `x_values` or `y_values`, so look up `f"data/{path}"`.

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
from pynxtools_simple.reader import SimpleReader

r = SimpleReader()
r.handle_hdf5_file("tests/data/workshop-example/mock_data.h5")
r.handle_eln_file("tests/data/workshop-example/eln_data.yaml")

print(r.get_attr("", "metadata/instrument/version"))        # → 1.0
print(r.get_eln_data("/ENTRY[entry]/title", ""))            # → My experiment
print(r.get_data("", "x_values").shape)                    # → (100,)
```

---

## Exercise 6 — Write the config file (35 min)

The config file is the **semantic bridge** between your data and the NeXus template. Each key is a template path; each value tells the reader where to find the data.

### Step 1 — generate the template

```bash
dataconverter generate-template --nxdl NXsimple
```

Copy the JSON output to `tests/data/workshop-example/config_file.json`.

### Step 2 — fill in the values

For each required or recommended path, decide:

| Data source | Config value |
|---|---|
| HDF5 metadata | `"@attrs:metadata/instrument/name"` |
| ELN YAML | `"@eln"` |
| Measurement array | `"@data:x_values"` |
| Fixed constant | `"532"` or `532` |

Example config fragment:

```json
{
  "/ENTRY[entry]/title":                  "@eln",
  "/ENTRY[entry]/start_time":             "@eln",
  "/ENTRY[entry]/INSTRUMENT[instrument]/SOURCE[source]/wavelength": "@attrs:metadata/source/wavelength",
  "/ENTRY[entry]/DATA[data]/x_values":    "@data:x_values",
  "/ENTRY[entry]/DATA[data]/data":        "@data:y_values"
}
```

### Step 3 — run the conversion

```bash
dataconverter \
  --reader simple \
  --nxdl NXsimple \
  --input-file tests/data/workshop-example/mock_data.h5 \
  --input-file tests/data/workshop-example/eln_data.yaml \
  --input-file tests/data/workshop-example/config_file.json \
  --output output.nxs
```

Inspect the result:

```bash
python3 -c "
import h5py
with h5py.File('output.nxs', 'r') as f:
    f.visititems(lambda n, obj: print(n))
"
```

---

## ✅ Final check — run the test suite

```bash
pytest tests/ -v
```

The existing tests use `ReaderTest`, which:
1. Runs the conversion
2. Compares the output to a reference file
3. Reports any differences

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

- [pynxtools tutorial > Build your first reader](https://fairmat-nfdi.github.io/pynxtools/tutorial/build-a-reader/)
- [pynxtools how-to > Use MultiFormatReader](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/use-multi-format-reader/)
- [pynxtools how-to > Build a plugin](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/build-a-plugin/)
