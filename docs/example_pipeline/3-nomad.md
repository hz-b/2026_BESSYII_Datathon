# Session 3 — Upload to NOMAD

**Duration:** 45 minutes

**Goal:** Upload the `output.nxs` file you produced in [Session 2](2-reader.md) to NOMAD, explore it through the GUI, and learn how to filter entries by NeXus application definition.

---

## What is NOMAD?

NOMAD is a free, open research data platform that understands NeXus natively. When you upload a `.nxs` file, NOMAD:

1. Detects the file format and triggers parsing automatically
2. Extracts the full NeXus hierarchy into a searchable metadata schema
3. Renders the default plot (from the `NXdata` `@signal`/`@axes` attributes)
4. Makes the entry findable by technique, instrument, or application definition

---

## Where to go

There exist different NOMAD deployments that are used for different purposes:

| Instance | URL | Notes |
|---|---|---|
| NOMAD production | `nomad-lab.eu/prod/v1/gui` | Public datasets — permanent |
| NOMAD test | `nomad-lab.eu/prod/v1/test/gui` | Safe to experiment — data is not permanent |
| Local deployment | `localhost:8080` | Your own NOMAD install |

**For this workshop, we are working on a specialized deployment called `oasis-b`**:

| Instance | URL | Notes |
|---|---|---|
| NOMAD oasis-b | `https://nomad-lab.eu/prod/v1/oasis-b/gui/` | Safe to experiment for this workshop |

**Please only use this URL!**

You need a free NOMAD account to upload. Create one at **Login → Register** on any instance above. Browsing published data is always public.

---

## Create an upload

1. Log in and go to **Publish → Your uploads**
2. Click **CREATE NEW UPLOAD**
3. Give it a name, e.g. `bessy-workshop-<your-name>`

The upload page has two views:

- **OVERVIEW** — manage the upload, see processed entries, publish
- **FILES** — full directory tree of everything in the upload

---

### Drop your file

Drag and drop `output.nxs` (or the reference `output.nxs` file) onto the upload area, or click to browse for it.

NOMAD detects the `.nxs` extension, identifies the pynxtools NeXus parser, and processes the file automatically. Wait for the green **processed** indicator next to the entry.

!!! tip
    You can upload a `.zip` containing `output.nxs` plus any auxiliary files (ELN YAML, raw HDF5, config JSON). NOMAD extracts the archive and processes each file it recognizes.

---

### Explore the entry

Click the **→** arrow icon next to the entry. Four tabs are available:

| Tab | What you see |
|---|---|
| **OVERVIEW** | Summary cards: core metadata on the left, visualizations on the right |
| **FILES** | The raw `.nxs` file and any auxiliary files |
| **DATA** | The fully parsed NeXus tree — every group, field, and attribute from your NXDL |
| **LOG** | Logging information about the processing of your file |

The **DATA tab** is the most useful for NeXus work: it renders the HDF5 hierarchy using NOMAD's Metainfo schema, with unit-aware values and inline documentation drawn from your NXDL `<doc>` strings. We also have `h5web` integrated there directly, so that you can explore the parsed data directly in the NeXus field.

The **OVERVIEW tab** shows a *NeXus* card with an interactive tree viewer using `h5web` — you can browse groups and fields without downloading the file.

---

## Filter by application definition

Your entry is now searchable by its NeXus application definition:

1. Go to **Explore → NeXus** in the top menu
2. Select the filter widget **NeXus class**
4. Set the filter to `NXdouble_slit`

You should see your entry (and any other workshop entries) appear in the list.

!!! note "Why this matters"
    Two participants who built readers for different instruments both targeting `NXdouble_slit` produce entries that are discoverable together in the same search. This is the concrete benefit of agreeing on an application definition.

---

## Analyze with NORTH

NORTH (NOMAD Remote Tools Hub) runs containerized Jupyter notebooks connected directly to your upload — no download needed.

1. From the entry **OVERVIEW** page, click **Analyze in NORTH** (or navigate to **Analyze → NORTH tools** from the top menu)
2. Choose a container. The generic **Jupyter** tool works for standard Python analysis.
3. NOMAD launches the container and mounts your upload directory. A Jupyter tab opens in the browser.
4. Access your file via the environment variable that NOMAD sets:

```python
import h5py, os

upload_path = os.environ.get("NOMAD_UPLOAD_PATH", ".")
with h5py.File(f"{upload_path}/output.nxs", "r") as f:
    data = f["entry/interference_pattern/data"][()]
    print(data.shape)
```

5. Files you write back into the upload directory are stored in NOMAD. Click **Reprocess** on the upload page to re-index newly created entries.

!!! note
    NORTH availability depends on the NOMAD deployment. The public and the `oasis-b` instance have NORTH enabled. Local deployments need to install the [`nomad-north-jupyter`](https://github.com/FAIRmat-NFDI/nomad-north-jupyter){:target="_blank" rel="noopener"} plugin.

---

## Share and publish (optional)

!!! danger
    Once published, an upload cannot be deleted and files cannot be changed. Use the **oasis-b instance** to practice first.


| Action | How | Effect |
|---|---|---|
| Share with a collaborator | **EDIT UPLOAD MEMBERS** → add user as Coauthor or Reviewer | They can view (or edit) the upload |
| Make visible to everyone | Check **Visible to all** under Edit visibility | Searchable before publishing |
| Publish | **PUBLISH** button at the bottom | Immutable, permanently public |
| Get a DOI | Group entries into a Dataset, then publish the dataset | Citable with a persistent identifier |

---

## Checklist

- [ ] `output.nxs` uploaded and shows **processed** status
- [ ] Entry opens and the DATA tab shows the full NeXus hierarchy
- [ ] `NXdouble_slit` filter in Explore → Entries returns your entry
- [ ] (Optional) NORTH Jupyter notebook opens and reads the data

---

## Further reading

- [NOMAD documentation > Upload and publish data](https://nomad-lab.eu/prod/v1/docs/tutorial/upload_publish.html){:target="_blank" rel="noopener"}
- [NOMAD documentation > Explore data](https://nomad-lab.eu/prod/v1/docs/tutorial/explore.html){:target="_blank" rel="noopener"}
- [pynxtools tutorial > NeXus files in NOMAD](https://fairmat-nfdi.github.io/pynxtools/tutorial/nexus-to-nomad.html){:target="_blank" rel="noopener"}
- [pynxtools how-to guide > Use pynxtools with NOMAD](https://fairmat-nfdi.github.io/pynxtools/how-tos/pynxtools/use-with-nomad.html){:target="_blank" rel="noopener"}
