# A primer on NeXus

**Duration:** 30 minutes

**Goal:** Understand what NeXus is, why it exists, and how its building blocks fit together — enough to make sense of the rest of the workshop.

---

## The problem NeXus solves

Every instrument produces data in a different format. A beamline system may write a proprietary file; a home-built setup writes CSV; a third instrument uses HDF5 but with its own internal layout. When someone else wants to reuse your data, they have to reverse-engineer the format.

**NeXus** is a community standard, widely used in materials science, that defines a single layout for scientific data. If your file is NeXus-conformant, any tool that understands NeXus can read it without extra documentation.

---

## Three building blocks

```text
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

Each base class defines subgroups, fields, and attributes, and their types, but makes **all of them optional**. A base class is a vocabulary, not a contract.

Browse them at [manual.nexusformat.org/classes/base_classes](https://manual.nexusformat.org/classes/base_classes/){:target="_blank" rel="noopener"}.

### 2. Application definitions

A **contract** for a specific experimental technique. It selects which base classes to use, which subgroups, fields, and attributes are required/recommended/optional, and what the structure must look like. A file that claims to conform to `NXxps` must contain everything `NXxps` requires.

Application definitions are the key to interoperability: two XPS instruments from different manufacturers can both write `NXxps` files that any reader can open.

Browse them at [fairmat-nfdi.github.io/nexus_definitions](https://fairmat-nfdi.github.io/nexus_definitions/){:target="_blank" rel="noopener"}.

### 3. HDF5 files

For serialization of NeXus data, almost always HDF5 is used. NeXus files are HDF5 files with a specific internal layout. You can open them with any HDF5 tool (`h5py`, `HDFView`,  `h5web`, …) or with a NeXus-aware tool like `pynxtools`.

---

## The NeXus hierarchy

A minimal NeXus file looks like this:

```text
/                          ← NXroot
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
| Base class | `NXdetector` | The *class*, i.e., what kind of component |
| Group | `NXxps/NXinstrument/NXdetector` | A specific detector for XPS measurements, a specialization of the base class within the application definition |
| Instance name | `entry/instrument/detector` | The *actual* group name in the HDF5 file |
| Template path | `/ENTRY[entry]/INSTRUMENT[instrument]/DETECTOR[detector]/` | Outside = name of the concept, `[inside]` = name of the instance |

```yaml
# NXDL (schema) in YAML
detector(NXdetector):
  distance(NX_FLOAT):

# HDF5 file (instance)
/entry/instrument/my_pilatus_detector/@NX_class = "NXdetector"  # any valid name
/entry/instrument/my_pilatus_detector/distance = 0.5   # any valid name
```

The HDF5 group `my_pilatus_detector` satisfies the `detector(NXdetector)` requirement because its type (indicated by the `@NX_class` attribute) is `NXdetector`.

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

- [pynxtools > Learn > NeXus primer](https://fairmat-nfdi.github.io/pynxtools/learn/nexus/nexus-primer/){:target="_blank" rel="noopener"}
- [NeXus manual > Applying NeXus](https://manual.nexusformat.org/applying-nexus.html){:target="_blank" rel="noopener"}
- [NeXus manual > NXDL types and unit categories](https://manual.nexusformat.org/nxdl-types.html){:target="_blank" rel="noopener"}
