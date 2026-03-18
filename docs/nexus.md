# A primer on NeXus

**Duration:** 30 minutes
**Goal:** Understand what NeXus is, why it exists, and how its three building blocks fit together — enough to make sense of the rest of the workshop.

---

## The problem NeXus solves

Every instrument produces data in a different format. A beamline XPS system writes a proprietary `.sle` file; a home-built setup writes CSV; a third instrument uses HDF5 but with its own internal layout. When someone else wants to reuse your data, they have to reverse-engineer the format.

**NeXus** is a community standard — agreed on by neutron, muon, and X-ray facilities — that defines a single HDF5-based layout for scientific data. If your file is NeXus-conformant, any tool that understands NeXus can read it without extra documentation.

---

## Three building blocks

```
Base classes          Application definitions          HDF5 file
─────────────         ───────────────────────          ─────────
NXsource              NXxps                           /entry/
NXdetector            NXmpes                             instrument/
NXsample              NXdouble_slit  ──validates──►        source/
NXprocess                                                   detector/
...                                                      sample/
                                                         data/
```

### 1. Base classes

Reusable components that describe parts of an experiment: `NXsource`, `NXdetector`, `NXsample`, `NXprocess`, …

Each base class defines fields and their types but makes **all of them optional** — a base class is a vocabulary, not a contract.

Browse them at [manual.nexusformat.org/classes/base_classes](https://manual.nexusformat.org/classes/base_classes/).

### 2. Application definitions

A **contract** for a specific experimental technique. It selects which base classes to use, which fields are required/recommended/optional, and what the structure must look like. A file that claims to conform to `NXxps` must contain everything `NXxps` requires.

Application definitions are the key to interoperability: two XPS instruments from different manufacturers can both write `NXxps` files that any reader can open.

Browse them at [fairmat-nfdi.github.io/nexus_definitions](https://fairmat-nfdi.github.io/nexus_definitions/).

### 3. HDF5 files

NeXus files are HDF5 files with a specific internal layout. You can open them with any HDF5 tool (h5py, HDFView, …) or with a NeXus-aware tool like `pynxtools`.

---

## The NeXus hierarchy

A minimal NeXus file looks like this:

```
/                          ← HDF5 root
└── entry/                 ← NXentry: one measurement
    ├── definition = "NXxps"
    ├── title = "Cu 2p XPS at room temperature"
    ├── start_time = "2024-03-15T14:32:00+01:00"
    ├── instrument/        ← NXinstrument
    │   ├── source/        ← NXsource
    │   └── detector/      ← NXdetector
    ├── sample/            ← NXsample
    └── data/              ← NXdata (default plot)
        ├── @signal = "intensity"
        ├── @axes = ["energy"]
        ├── energy = [280, 281, ..., 295]  eV
        └── intensity = [...]
```

`NXentry` wraps one complete measurement. `NXdata` marks the default plottable dataset — tools like NOMAD use `@signal` and `@axes` to render a plot without any configuration.

---

## Naming rules

| Concept | Name | What it means |
|---|---|---|
| Base class type | `NXdetector` | The *class* — what kind of component |
| Instance name | `detector` (lowercase) | The *actual* group name in the HDF5 file |
| Template path | `/ENTRY[entry]/INSTRUMENT[instrument]/DETECTOR[detector]/` | Upper case = class, `[lower]` = instance |

```yaml
# NXDL (schema)
detector(NXdetector):   # concept name (schema): can be any name
  distance(NX_FLOAT):

# HDF5 file (instance)
/entry/instrument/my_pilatus_detector/distance = 0.5   # any valid name
```

The HDF5 group `my_pilatus_detector` satisfies the `detector(NXdetector)` requirement because its type is `NXdetector`.

---

## Fields, types, and units

```yaml
wavelength(NX_FLOAT):
  unit: NX_WAVELENGTH    # a unit *category*, not a specific unit
  doc: Central wavelength of the source.
```

- **Type**: `NX_FLOAT`, `NX_INT`, `NX_CHAR`, `NX_DATE_TIME`, `NX_NUMBER`, …
- **Unit category**: `NX_LENGTH`, `NX_ENERGY`, `NX_WAVELENGTH`, … — stored in a sibling HDF5 attribute `wavelength/@units`
- **Optionality**: fields in application definitions are **required by default**; mark others with `exists: recommended` or `exists: optional`

---

## Key takeaways

1. NeXus = HDF5 + agreed naming + community schemas
2. Base classes = vocabulary; application definitions = contract
3. `NXdata` with `@signal`/`@axes` = the default plot
4. Files are validated against an application definition with `pynxtools`

---

## Further reading

- [pynxtools > Learn > NeXus primer](https://fairmat-nfdi.github.io/pynxtools/learn/nexus/nexus-primer/)
- [NeXus manual > Applying NeXus](https://manual.nexusformat.org/applying-nexus.html)
- [NeXus manual > NXDL types and unit categories](https://manual.nexusformat.org/nxdl-types.html)
