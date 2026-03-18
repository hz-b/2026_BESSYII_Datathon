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

```bash
uv pip install nyaml
```

Verify:

```bash
nyaml2nxdl --help
```

---

## 4. Instantiate the plugin template

The workshop uses a special `workshop` branch of the plugin template that gives you a pre-structured reader skeleton with exercise stubs.

```bash
pip install cookiecutter
cookiecutter gh:FAIRmat-NFDI/pynxtools-plugin-template --checkout workshop
```

Answer the prompts:

| Prompt | Suggested answer |
|--------|----------------|
| `reader_name` | `simple` (or your instrument name) |
| `supported_nxdl` | `NXsimple` |
| `technique` | `Simple` |

This creates a directory called `pynxtools-simple/` (or whatever name you chose).

```bash
cd pynxtools-simple
uv pip install -e ".[dev]"
```

Verify:

```bash
dataconverter --reader simple --nxdl NXsimple --help
```

---

## 5. Download the workshop example data

The example data lives inside your plugin under `tests/data/workshop-example/`:

```bash
ls tests/data/workshop-example/
# mock_data.h5   eln_data.yaml   config_file.json   NXsimple.nxdl.xml
```

Explore the HDF5 file:

```python
import h5py

with h5py.File("tests/data/workshop-example/mock_data.h5", "r") as f:
    f.visititems(lambda name, obj: print(name))
```

---

## 6. NOMAD account

If you haven't already, create a free account at [https://nomad-lab.eu/prod/v1/oasis-b/](https://nomad-lab.eu/prod/v1/oasis-b/gui/):

1. Click **Login → Register**
2. Fill in the form and confirm your e-mail

---

## Checklist

- [ ] `dataconverter --help` works
- [ ] `nyaml2nxdl --help` works
- [ ] Plugin directory exists and `pip install -e ".[dev]"` succeeded
- [ ] `tests/data/workshop-example/mock_data.h5` is readable with h5py
- [ ] NOMAD account created (or test instance accessible)

---

!!! tip "Stuck on setup?"
    Ask a neighbour or an instructor. Setup issues are normal and fixable — don't spend more than 5 minutes on any one problem before asking.
