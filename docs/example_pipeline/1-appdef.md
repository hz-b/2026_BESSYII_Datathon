# Session 1 — Write your first application definition

**Duration:** 1 hour

**Goal:** Create `NXdouble_slit` — a complete, valid application definition for the [double-slit experiment](https://en.wikipedia.org/wiki/Double-slit_experiment){:target="_blank" rel="noopener"} — starting from a blank YAML file.

!!! info
    We will go through this example for writing an application definition rather quickly. If you want to later similar material at a more incremental later, you can have a look at the `pynxtools` tutorial for writing an application definition: 
    
    [`pynxtools` > Tutorial > Writing your first application definition](https://fairmat-nfdi.github.io/pynxtools/tutorial/writing-an-application-definition.html){:target="_blank" rel="noopener"}

    Each step you will see in this guide is already pre-defined and ready to be copied. For each step, try to understand what the individual concepts are and how the notation works before you copy the snippets to your application definition.

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

```text
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

## Step 0 — Tools

### nyaml

NeXus definitions must follow the [NXDL language](https://manual.nexusformat.org/nxdl.html){:target="_blank" rel="noopener"}. Typically, they are written in XML. Here, we are using a simpler YAML notation. The [`nyaml` tool](https://github.com/FAIRmat-NFDI/nyaml){:target="_blank" rel="noopener"} you installed earlier will help us losslessly convert between YAML and XML.

```bash
# Convert YAML ↔ NXDL XML
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
nxdl2nyaml NXdouble_slit.nxdl.xml --output-file NXdouble_slit.yaml
```

### pynxtools dataconverter

Generate a template JSON showing all required paths:
```bash
dataconverter generate-template --nxdl NXdouble_slit
```

---

## Step 1 — Skeleton

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
    start_time(NX_DATE_TIME): |
      ISO 8601 datetime of the measurement start.
```

You can see its XML representation by running the converter:

```bash
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
```

---

## Step 2 — Instrument and source

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

## Step 3 — Slit and detector

Add the slit and detector inside `(NXinstrument)`:

```yaml
      double_slit(NXslit):
        x_gap(NX_FLOAT):
          unit: NX_LENGTH
          doc: |
            Width of each individual slit.
        slit_separation(NX_FLOAT):
          unit: NX_LENGTH
          doc: |
            Center-to-center distance between the two slits.
        height(NX_FLOAT):
          exists: optional
          unit: NX_LENGTH
          doc: |
            Height of the two slits.
      detector(NXdetector):
        distance(NX_FLOAT):
          unit: NX_LENGTH
          doc: Distance from the slit plane to the detector surface.
```

---

## Step 4 — Optional processing + default plot

Add these two groups as siblings of `(NXinstrument)` inside `(NXentry)`.

### Optional processing provenance (uses `nameType="partial"`)

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

### Default plot

Note how here we are using `enumeration` to indicate the `\@signal` and `\@axes` attribute
must match the fields defined within `NXdata`.

```yaml
    interference_pattern(NXdata):
      doc: |
        Default plot: the calibrated 2D interference pattern with spatial
        axes. The signal data may be identical to the raw detector array or
        derived from it via one or more NXprocess steps.
      \@signal:
        enumeration: [data]
      \@axes:
        enumeration: [['x', 'y']]
      data(NX_NUMBER):
        unit: NX_ANY
        doc: |
          2D interference intensity after any processing steps.
        dimensions:
          rank: 2
          dim: (n_x, n_y)
      x(NX_FLOAT):
        unit: NX_LENGTH
        doc: |
          Horizontal spatial offset from the detector centre, derived from
          pixel index and pixel pitch.
        dimensions:
          rank: 1
          dim: (n_x,)
      y(NX_FLOAT):
        unit: NX_LENGTH
        doc: |
          Vertical spatial offset from the detector centre, derived from
          pixel index and pixel pitch.
        dimensions:
          rank: 1
          dim: (n_y,)
```

Here, we are using symbolic names (`n_x`, `n_y`) to name the array dimensions and reference them in <dimensions>. Using symbolic names instead of hardcoded integers makes the definition self-documenting and allows validation tools to verify dimensional consistency across fields.

Add the dimension symbols at the top of the file, outside of the class `NXdouble_slit` and before `type: group`:

```yaml
symbols:
  doc: |
    Dimension symbols used in this definition.
  n_x: |
    Number of detector pixels along x.
  n_y: |
    Number of detector pixels along y.
```

---

## Step 5 — Validate

Convert to NXDL XML and inspect:

```bash
nyaml2nxdl NXdouble_slit.yaml --output-file NXdouble_slit.nxdl.xml
```

As discussed earlier, the application definition is already used inside `pynxtools`

Run:

```bash
dataconverter generate-template --nxdl NXdouble_slit
```

You should now see a dictionary-style output. You should see e.g. `/ENTRY[entry]/title` and `/ENTRY[entry]/start_time` in the output. This is a template for `NXdouble_slit` that we will fill in the next step. 

!!! note "Concept vs. instance paths"
    The template shows `ENTRY` (upper case) — the concept — and `[entry]` (lower case in brackets) — the default instance name. The concept is the schema; the instance is what gets written to the file.

Look for these paths in the output:

- `/ENTRY[entry]/interference_pattern/x_offset`
- `/ENTRY[entry]/interference_pattern/y_offset`
- `/ENTRY[entry]/processID[processID]/description` (optional)
- `/ENTRY[entry]/INSTRUMENT[instrument]/detector/distance`


You can also pass the `--required` flag to only see those paths that are required to be filled:

```bash
dataconverter generate-template --nxdl NXdouble_slit --required
```

Note that the complet version of `NXdouble_slit` uses even more concepts to illustrate all the possibilities the NeXus data model provides. Have a look at the complete `NXdouble_slit.yaml`. Which additional ideas can you detect?

??? success "Complete NXdouble_slit.yaml"

    ```yaml
    category: application
    doc: |
      Application definition for a double-slit interference experiment.
      Records the light source, aperture geometry, detector layout, and the
      measured 2D interference pattern needed to determine fringe spacing and
      source coherence length.
      
      See https://en.wikipedia.org/wiki/Double-slit_experiment.
    symbols:
      doc: |
        Dimension symbols used in this definition.
      n_x: |
        Number of detector pixels along x.
      n_y: |
        Number of detector pixels along y.
    type: group
    NXdouble_slit(NXobject):
      (NXentry):
        definition:
          enumeration: [NXdouble_slit]
        title:
        start_time(NX_DATE_TIME):
          doc: |
            ISO 8601 datetime of the measurement start.
        end_time(NX_DATE_TIME):
          exists: recommended
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
                Temporal coherence length of the source.
            type(NX_CHAR):
              exists: optional
              enumeration: [Laser, Filtered lamp, LED]
          double_slit(NXslit):
            x_gap(NX_FLOAT):
              unit: NX_LENGTH
              doc: |
                Width of each individual slit.
            slit_separation(NX_FLOAT):
              unit: NX_LENGTH
              doc: |
                Center-to-center distance between the two slits.
            height(NX_FLOAT):
              exists: optional
              unit: NX_LENGTH
              doc: |
                Height of the two slit.
          detector(NXdetector):
            distance(NX_FLOAT):
              unit: NX_LENGTH
              doc: |
                Distance from the slit plane to the detector surface.
        processID(NXprocess):
          nameType: partial
          exists: optional
          doc: |
            Describes one step in the processing chain that converts raw detector
            pixel data to the calibrated interference pattern stored in
            ``interference_pattern``. The 'ID' suffix in the group name is replaced
            by a short identifier chosen by the writer, e.g. 'pixel_calibration'
            or 'background_correction'. Multiple NXprocess groups may be present;
            their order is given by sequence_index.
          sequence_index(NX_POSINT):
            doc: |
              Sequence index of processing, for determining the order of multiple
              NXprocess steps. Starts with 1.
          description(NX_CHAR):
            doc: |
              Free-text description of what this processing step does.
          program(NX_CHAR):
            exists: optional
            doc: |
              Version string of the software.
          version(NX_CHAR):
            exists: optional
          date(NX_DATE_TIME):
            exists: optional
        interference_pattern(NXdata):
          doc: |
            Default plot: the calibrated 2D interference pattern with spatial
            axes. The signal data may be identical to the raw detector array or
            derived from it via one or more NXprocess steps.
          \@signal:
            enumeration: [data]
          \@axes:
            enumeration: [['x', 'y']]
          data(NX_NUMBER):
            unit: NX_ANY
            doc: |
              2D interference intensity after any processing steps.
            dimensions:
              rank: 2
              dim: (n_x, n_y)
          x(NX_FLOAT):
            unit: NX_LENGTH
            doc: |
              Horizontal spatial offset from the detector centre, derived from
              pixel index and pixel pitch.
            dimensions:
              rank: 1
              dim: (n_x,)
          y(NX_FLOAT):
            unit: NX_LENGTH
            doc: |
              Vertical spatial offset from the detector centre, derived from
              pixel index and pixel pitch.
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

```yaml
```yaml
NXdouble_slit(NXobject):
  (NXentry):
    (NXinstrument):
      source(NXlaser):
        wavelength(NX_FLOAT):
        coherence_length(NX_FLOAT):
          exists: recommended
```

!!! info
    This is just an exercise for you to understand NeXus better. We will not use `NXlaser` in the upcoming examples. Find a complete `NXlaser.nxdl.xml` in the [`pynxtools` examples](https://github.com/FAIRmat-NFDI/pynxtools/tree/master/examples/custom-application-definition).

---

## Appendix: How to add your definition to pynxtools

As we said above, `NXdouble_slit.nxdl.xml` is already part of `pynxtools`, so you don't need to add it for the following steps. However, if you create another application definition `NXmytechnique`, there are two possibilities of adding it.

**Local development (fastest):**

In order to use your application definitions and base classes directly, you will need to add them to the NeXus definitions stored in `pynxtools`. For this, you need to install `pynxtools` in editable mode. You can learn more in the `pynxtools` [development guide](https://fairmat-nfdi.github.io/pynxtools/tutorial/contributing.html#development-installation){:target="_blank" rel="noopener"}.

Install `pynxtools` with the `-e` option in the same virtual environment that you are already working in. Instantiate the `definitions` submodule.

Then you can place your application definition NXDL XML file in `pynxtools`:

```bash

cp NXmytechnique.nxdl.xml src/pynxtools/definitions/contributed_definitions/
dataconverter generate-template --nxdl NXmytechnique
```

**Community contribution (permanent):**

The more permanent way is to add the new application definition (or base class) to the FAIRmat NeXus definitions repository. That ensures that others can use it and that it can eventually be brought to standardization with the NeXus International Advisory Committee (NIAC):

1. Fork [FAIRmat-NFDI/nexus_definitions](https://github.com/FAIRmat-NFDI/nexus_definitions){:target="_blank" rel="noopener"}
2. Add your NXDL file to `contributed_definitions/`
3. Open a pull request
4. Once merged, update the pynxtools submodule: `./scripts/definitions.sh update`

---

## Further reading

- [pynxtools tutorial > Writing your first application definition](https://fairmat-nfdi.github.io/pynxtools/tutorial/writing-an-application-definition.html){:target="_blank" rel="noopener"}
- [pynxtools: how to write an application definition](https://fairmat-nfdi.github.io/pynxtools/how-tos/nexus/write-an-application-definition.html){:target="_blank" rel="noopener"}
- [NeXus manual > Applying NeXus](https://manual.nexusformat.org/applying-nexus.html){:target="_blank" rel="noopener"}
- [nyaml documentation](https://fairmat-nfdi.github.io/nyaml/){:target="_blank" rel="noopener"}
