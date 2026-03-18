# Session 1 — Write your first application definition

**Duration:** 1 hour

**Goal:** Create `NXdouble_slit` — a complete, valid application definition for the [double-slit experiment](https://en.wikipedia.org/wiki/Double-slit_experiment) — starting from a blank YAML file.

---

## Why a double-slit experiment?

It's physically simple (one source, two slits, one detector), but captures every structural challenge in data modelling:

- Instrument geometry (distances, gaps)
- A 2D measurement array with calibrated axes
- A distinction between raw and processed data
- Optional metadata (material, coherence length)

The skills you learn here transfer directly to XPS, MPES, reflectometry, or any other technique.

---

## The data model we want to build

```
NXentry
 ├── definition = "NXdouble_slit"
 ├── title, start_time, end_time
 ├── NXinstrument
 │   ├── source(NXsource)
 │   │   ├── wavelength  [NX_WAVELENGTH]
 │   │   ├── coherence_length  [NX_LENGTH, recommended]
 │   │   └── type  [Laser | Filtered lamp | LED, optional]
 │   ├── double_slit(NXslit)
 │   │   ├── x_gap, slit_separation  [NX_LENGTH]
 │   │   └── height, material  [optional]
 │   └── detector(NXdetector)
 └── interference_pattern(NXdata)  ← calibrated default plot
     ├── data  dims (n_x, n_y)
     ├── x_offset  [NX_LENGTH]
     └── y_offset  [NX_LENGTH]
```

- **`interference_pattern`** contains spatial axes (mm from centre). This is the used as the default plot by NOMAD and other viewers.

Note that the `NXdouble_slit` application definition is already part of `pynxtools`. We will still build it from scratch here, but this will help us with using it directly, without needing to manually inject it there.

---

## Tools

```bash
# Convert YAML ↔ NXDL XML
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
nxdl2nyaml NXdouble_slit.nxdl.xml --output-file NXdouble_slit.yaml

# Generate a template JSON showing all required paths
dataconverter generate-template --nxdl NXdouble_slit
```

---

## Step 0 — nyaml

NeXus definitions must follow the [NXDL language](https://manual.nexusformat.org/nxdl.html). Typically, they are written in XML.

Here, we are using a simpler YAML notation. THe [`nyaml` tool](https://github.com/FAIRmat-NFDI/nyaml) you installed earlier will help us losslessly convert between YAML and XML.


---

## Step 1 — Skeleton (10 min)

Create `NXdouble_slit.yaml` in a new working directory:

```yaml
category: application
doc: |
  Application definition for a double-slit interference experiment.
type: group
NXdouble_slit(NXobject):
  (NXentry):
    definition:
      enumeration: [NXdouble_slit]
    title:
    start_time(NX_DATE_TIME):
```

You can see its XML representation by running the converter:

```bash
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
```

---

## Step 2 — Instrument and source (15 min)

Add the instrument and light source inside `(NXentry)`:

```yaml
    (NXinstrument):
      source(NXsource):
        wavelength(NX_FLOAT):
          unit: NX_WAVELENGTH
          doc: |
            Central wavelength of the light source.
        coherence_length(NX_FLOAT):
          unit: NX_LENGTH
          exists: recommended
          doc: |
            Temporal coherence length — determines fringe visibility.
        type(NX_CHAR):
          exists: optional
          enumeration: [Laser, Filtered lamp, LED]
```

!!! note "Unit categories"
    `unit: NX_WAVELENGTH` is a *category*, not a unit. It says the field must store a wavelength-equivalent quantity (nm, Å, µm, …). The actual unit is written by the file producer as an HDF5 attribute `wavelength/@units`.

---

## Step 3 — Slit and detector (15 min)

Add the slit and detector inside `(NXinstrument)`:

```yaml
      double_slit(NXslit):
        x_gap(NX_FLOAT):
          unit: NX_LENGTH
          doc: Width of each individual slit.
        slit_separation(NX_FLOAT):
          unit: NX_LENGTH
          doc: Center-to-center distance between the two slits.
        height(NX_FLOAT):
          unit: NX_LENGTH
          exists: optional
        material(NX_CHAR):
          exists: optional

      detector(NXdetector):
        distance(NX_FLOAT):
          unit: NX_LENGTH
          doc: Distance from the slit plane to the detector surface.
        data(NXdata):
          doc: |
            Raw 2D pixel data collected by the detector with no calibration
            applied. Indexed by integer pixel coordinates.
          \@signal:
            enumeration: [data]
          \@axes:
            enumeration: [['x', 'y']]
          data(NX_NUMBER):
            exists: recommended
            unit: NX_ANY
            dimensions:
              rank: 2
              dim: (n_x, n_y)
          x(NX_INT):
            doc: Pixel indices along the horizontal axis (0-based).
            dimensions:
              rank: 1
              dim: (n_x,)
          y(NX_INT):
            doc: Pixel indices along the vertical axis (0-based).
            dimensions:
              rank: 1
              dim: (n_y,)
```

Add the dimension symbols at the top of the file, before `type: group`:

```yaml
symbols:
  doc: Dimension symbols used in this definition.
  n_x: Number of detector pixels along x.
  n_y: Number of detector pixels along y.
```

---

## Step 4 — Optional processing + default plot (15 min)

Add these two groups as siblings of `(NXinstrument)` inside `(NXentry)`.

**Optional processing provenance** (uses `nameType="partial"`):

```yaml
    processID(NXprocess):
      nameType: partial
      exists: optional
      doc: |
        One step in the pipeline from raw pixels to calibrated offsets.
        Replace 'ID' with a short name, e.g. 'pixel_calibration'.
        Multiple NXprocess groups are allowed.
      sequence_index(NX_INT):
        doc: Step order in the chain (1-based).
      description(NX_CHAR):
        doc: What this step does.
      program(NX_CHAR):
        exists: optional
      version(NX_CHAR):
        exists: optional
      date(NX_DATE_TIME):
        exists: optional
```

**Default plot:**

Note how here we are using `enumeration` to indicate the `\@signal` and `\@axes` attribute
must match the fields defined within `NXdata`.

```yaml
    interference_pattern(NXdata):
      doc: |
        Default plot: calibrated 2D interference pattern.
        Axes are physical spatial offsets derived from pixel indices.
      \@signal:
        enumeration: [data]
      \@axes:
        enumeration: [['x_offset', 'y_offset']]
      data(NX_NUMBER):
        unit: NX_ANY
        dimensions:
          rank: 2
          dim: (n_x, n_y)
      x_offset(NX_FLOAT):
        unit: NX_LENGTH
        doc: Horizontal spatial offset from detector centre.
        dimensions:
          rank: 1
          dim: (n_x,)
      y_offset(NX_FLOAT):
        unit: NX_LENGTH
        doc: Vertical spatial offset from detector centre.
        dimensions:
          rank: 1
          dim: (n_y,)
```

---

## Step 5 — Validate (10 min)

Convert to NXDL XML and inspect:

```bash
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
```

As discussed earlier, the application definition is already used inside `pynxtools`.

Run:

```bash
dataconverter generate-template --nxdl NXdouble_slit
```

You should now see a dictionary-style output. You should see e.g. `/ENTRY[entry]/title` and `/ENTRY[entry]/start_time` in the output. This is a template for `NXdouble_slit` that we will fill in the next step. 

!!! note "Concept vs. instance paths"
    The template shows `ENTRY` (upper case) — the concept — and `[entry]` (lower case in brackets) — the default instance name. The concept is the schema; the instance is what gets written to the file.

Look for these paths in the output:
- `/ENTRY[entry]/INSTRUMENT[instrument]/detector/DATA[data]/x`
- `/ENTRY[entry]/INSTRUMENT[instrument]/detector/DATA[data]/y`
- `/ENTRY[entry]/interference_pattern/x_offset`
- `/ENTRY[entry]/interference_pattern/y_offset`
- `/ENTRY[entry]/PROCESSINGID[processID]/description` (optional)

??? success "Complete NXdouble_slit.yaml"

    ```yaml
    category: application
    doc: |
      Application definition for a double-slit interference experiment.
      Records the light source, aperture geometry, detector layout, and the
      measured 2D interference pattern needed to determine fringe spacing and
      source coherence length.
    symbols:
      doc: Dimension symbols used in this definition.
      n_x: Number of detector pixels along x.
      n_y: Number of detector pixels along y.
    type: group
    NXdouble_slit(NXobject):
      (NXentry):
        definition:
          enumeration: [NXdouble_slit]
        title:
        start_time(NX_DATE_TIME):
          doc: ISO 8601 datetime of the measurement start.
        end_time(NX_DATE_TIME):
          exists: recommended
        (NXinstrument):
          source(NXsource):
            wavelength(NX_FLOAT):
              unit: NX_WAVELENGTH
              doc: Central wavelength of the light source.
            coherence_length(NX_FLOAT):
              unit: NX_LENGTH
              exists: recommended
              doc: Temporal coherence length.
            type(NX_CHAR):
              exists: optional
              enumeration: [Laser, Filtered lamp, LED]
          double_slit(NXslit):
            x_gap(NX_FLOAT):
              unit: NX_LENGTH
              doc: Width of each individual slit.
            slit_separation(NX_FLOAT):
              unit: NX_LENGTH
              doc: Center-to-center distance between the two slits.
            height(NX_FLOAT):
              unit: NX_LENGTH
              exists: optional
            material(NX_CHAR):
              exists: optional
          detector(NXdetector):
            distance(NX_FLOAT):
              unit: NX_LENGTH
              doc: Distance from the slit plane to the detector surface.
            data(NXdata):
              doc: Raw 2D pixel data. Indexed by integer pixel coordinates.
              \@signal:
                enumeration: [data]
              \@axes:
                enumeration: [['x', 'y']]
              data(NX_NUMBER):
                exists: recommended
                unit: NX_ANY
                dimensions:
                  rank: 2
                  dim: (n_x, n_y)
              x(NX_INT):
                dimensions:
                  rank: 1
                  dim: (n_x,)
              y(NX_INT):
                dimensions:
                  rank: 1
                  dim: (n_y,)
        processID(NXprocess):
          nameType: partial
          exists: optional
          doc: |
            One processing step converting raw pixels to calibrated offsets.
          sequence_index(NX_INT):
          description(NX_CHAR):
          program(NX_CHAR):
            exists: optional
          version(NX_CHAR):
            exists: optional
          date(NX_DATE_TIME):
            exists: optional
        interference_pattern(NXdata):
          doc: Default plot — calibrated 2D interference pattern.
          \@signal:
            enumeration: [data]
          \@axes:
            enumeration: [['x_offset', 'y_offset']]
          data(NX_NUMBER):
            unit: NX_ANY
            dimensions:
              rank: 2
              dim: (n_x, n_y)
          x_offset(NX_FLOAT):
            unit: NX_LENGTH
            dimensions:
              rank: 1
              dim: (n_x,)
          y_offset(NX_FLOAT):
            unit: NX_LENGTH
            dimensions:
              rank: 1
              dim: (n_y,)
    ```

---

## Advanced: specialize a base class (bonus)

If your source is always a laser, you can create a dedicated `NXlaser` base class rather than repeating the specialization in every application definition:

```yaml
# NXlaser.yaml
category: base
doc: A specialization of NXsource for coherent laser sources.
type: group
NXlaser(NXsource):
  wavelength(NX_FLOAT):
    doc: Central wavelength of the laser line.
  coherence_length(NX_FLOAT):
    unit: NX_LENGTH
    exists: recommended
  type(NX_CHAR):
    enumeration: [Laser]
```

Then in `NXdouble_slit`, replace `source(NXsource)` with `source(NXlaser)` and only list which fields are required or recommended — everything defined in `NXlaser` is inherited automatically.

---

## Appendix: How to add your definition to pynxtools

As we said above, `NXdouble_slit.nxdl.xml` is already part of `pynxtools`, so you don't need to add it for the following steps. Hoever, if you create another application definition `NXmytechnique`, there are two possibilities of adding them.

**Local development (fastest):**

In order to use your application definitions directly, you will need to add to the NeXus defintitions stored in `pynxtools`. For this, you need to install `pynxtools` in editable mode. You can learn more in the `pynxtools` [development guide](https://fairmat-nfdi.github.io/pynxtools/tutorial/contributing.html#development-installation).

Install `pynxtools` with the `-e` option in the same virtual environment that you are already working in. Instantiate the `definitions` submodule.

Then you can place your application definition NXDL XML file in `pynxtools`:

```bash

cp NXmytechnique.nxdl.xml src/pynxtools/definitions/contributed_definitions/
dataconverter generate-template --nxdl NXmytechnique
```


```bash
cp NXdouble_slit.nxdl.xml \
    src/pynxtools/definitions/contributed_definitions/
dataconverter generate-template --nxdl NXdouble_slit
```

**Community contribution (permanent):**

The more permant way is to add the application definition (or new base classes) to the FAIRmat NeXus definitions repository. That ensures that others can use it and that it can eventually be brought to standardization with the NeXus International Advisory Committee (NIAC):

1. Fork [FAIRmat-NFDI/nexus_definitions](https://github.com/FAIRmat-NFDI/nexus_definitions)
2. Add your NXDL file to `contributed_definitions/`
3. Open a pull request
4. Once merged, update the pynxtools submodule: `./scripts/definitions.sh update`

---

## Further reading

- [pynxtools tutorial > Write your first application definition](https://fairmat-nfdi.github.io/pynxtools/tutorial/writing-an-application-definition/)
- [NeXus manual > Applying NeXus](https://manual.nexusformat.org/applying-nexus.html)
- [nyaml documentation](https://fairmat-nfdi.github.io/nyaml/)
