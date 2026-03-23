# Session 0 — Setup

**Duration:** 30 minutes

**Goal:** Every participant has a working Python environment with `pynxtools`, the plugin template instantiated, and a NOMAD account.

---

## 1. Python environment

You need Python 3.10 or later. We recommend `uv` for fast, isolated installs, but plain `pip` works fine.

=== "uv (recommended)"

    ```bash
    # Install uv if you don't have it
    curl -LsSf https://astral.sh/uv/install.sh | sh

    # Create and activate a fresh environment
    uv venv
    source .venv/bin/activate   # Linux / macOS
    # .venv\Scripts\activate    # Windows
    ```

=== "pip / venv"

    ```bash
    # For this work, you need to install python separately
    python3 -m venv 
    source .venv/bin/activate   # Linux / macOS
    # .venv\Scripts\activate    # Windows
    ```

---

## 2. Install pynxtools

```bash
uv pip install pynxtools[convert]
```

Verify:

```bash
dataconverter --help
```

You should see the `dataconverter` CLI help text.

---

## 3. Install nyaml

This is the tool that allows us to easily write NeXus NXDL files.

```bash
uv pip install nyaml
```

Verify:

```bash
nyaml2nxdl --help
```

---

## 4. Instantiate the plugin template

We are creating a `pynxtools` reader plugin, i.e., a converter from raw data to NeXus. For this we are working with the [`pynxtools` plugin template](https://github.com/FAIRmat-NFDI/pynxtools-plugin-template){:target="_blank" rel="noopener"}.

The workshop uses a special `workshop` branch of the plugin template that gives you a pre-structured reader skeleton with exercise stubs.

```bash
uv pip install cookiecutter
cookiecutter gh:FAIRmat-NFDI/pynxtools-plugin-template --checkout bessy-datathon
```

Answer the prompts:

| Prompt | Suggested answer |
|--------|----------------|
| `reader_name` | `workshop` |
| `supported_nxdl` | `NXdouble_slit` |
| `technique` | `Double slit` |
| `reader` | `DoubleSlitReader` |
| `Author's full name` | Your name |
| `Author's email address` | Your email or empty |
| `Any other prompt` | Go with the default |

This creates a directory called `pynxtools-workshop/` (or whatever name you chose).

```bash
cd pynxtools-workshop
uv pip install -e ".[dev]"
```

Verify that the `double_slit` reader is registered:

```bash
dataconverter get-readers
```

You should see the `workshop` reader here. Readers are by default named liked their plugin.

```bash
dataconverter --reader workshop --nxdl NXdouble_slit --help
```

---

## 5. Download the workshop example data

The example data lives inside your plugin under `tests/data/`:

```bash
ls tests/data/
# mock_data.h5   eln_data.yaml   config_file.json   create_mock_data.py   README.md
```

Explore the HDF5 file. You can have a look at the HDF5 file using the `h5web` extension in VS Code.

Alternatively, open a new Python file (e.g. `explore_data.py`) and run:

```python
import h5py

with h5py.File("tests/data/mock_data.h5", "r") as f:
    f.visititems(lambda name, obj: print(name))
```

You should see paths like `data/detector_data`, `data/x_pixels`, `data/interference_data`,
and `metadata/instrument/source/wavelength`.

---

## 6. NOMAD account

If you haven't already, create a free account at [https://nomad-lab.eu/prod/v1/oasis-b/](https://nomad-lab.eu/prod/v1/oasis-b/gui/){:target="_blank" rel="noopener"}:

1. Click **Login → Register**
2. Fill in the form and confirm your e-mail

---

## Checklist

- [ ] `dataconverter --help` works
- [ ] `nyaml2nxdl --help` works
- [ ] Plugin directory exists and `uv pip install -e ".[dev]"` succeeded
- [ ] `tests/data/mock_data.h5` is readable with h5py
- [ ] NOMAD account created (or test instance accessible)

---

!!! tip "Stuck on setup?"
    Ask a neighbor or an instructor. Setup issues are normal and fixable — don't spend more than 5 minutes on any one problem before asking.
