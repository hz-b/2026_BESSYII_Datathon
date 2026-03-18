# Session 4 — Upload to NOMAD

**Duration:** 45 minutes

**Goal:** Upload the `output.nxs` file you produced in Session 3 to NOMAD, explore it through the GUI, and learn how to filter entries by NeXus application definition.

---

## What is NOMAD?

NOMAD is a free, open research data platform that understands NeXus natively. When you upload a `.nxs` file, NOMAD:

1. Detects the file format and triggers parsing automatically
2. Extracts the full NeXus hierarchy into a searchable metadata schema
3. Renders the default plot (from the `NXdata` `@signal`/`@axes` attributes)
4. Makes the entry findable by technique, instrument, or application definition

---

## 4.1 Where to go

| Instance | URL | Notes |
|---|---|---|
| NOMAD production | `nomad-lab.eu/prod/v1/gui` | Public datasets — permanent |
| NOMAD test | `nomad-lab.eu/prod/v1/test/gui` | Safe to experiment — data is not permanent |
| Local deployment | `localhost:8080` | Your own NOMAD install |

You need a free NOMAD account to upload. Create one at **Login → Register** on any instance above. Browsing published data is always public.

---

## 4.2 Create an upload

1. Log in and go to **Publish → Your uploads**
2. Click **CREATE NEW UPLOAD**
3. Give it a name, e.g. `bessy-workshop-<your-name>`

The upload page has two views:

- **OVERVIEW** — manage the upload, see processed entries, publish
- **FILES** — full directory tree of everything in the upload

---

## 4.3 Drop your file

Drag and drop `output.nxs` onto the upload area, or click to browse for it.

NOMAD detects the `.nxs` extension, identifies the pynxtools NeXus parser, and processes the file automatically. Wait for the green **processed** indicator next to the entry.

!!! tip
    You can upload a `.zip` containing `output.nxs` plus any auxiliary files (ELN YAML, raw HDF5, config JSON). NOMAD extracts the archive and processes each file it recognises.

---

## 4.4 Explore the entry

Click the **→** arrow icon next to the entry. Three tabs are available:

| Tab | What you see |
|---|---|
| **OVERVIEW** | Summary cards: core metadata on the left, visualisations on the right |
| **FILES** | The raw `.nxs` file and any auxiliary files |
| **DATA** | The fully parsed NeXus tree — every group, field, and attribute from your NXDL |

The **DATA tab** is the most useful for NeXus work: it renders the HDF5 hierarchy using NOMAD's metainfo schema, with unit-aware values and inline documentation drawn from your NXDL `<doc>` strings.

The **OVERVIEW tab** shows a *NeXus* card with an interactive tree viewer — you can browse groups and fields without downloading the file.

---

## 4.5 Filter by application definition

Your entry is now searchable by its NeXus application definition:

1. Go to **Explore → Entries** in the top menu
2. Open the filter panel on the left
3. Under **Data**, expand the **NeXus** filter group
4. Set **Application definition** to `NXsimple`

You should see your entry (and any other workshop entries) appear in the list.

!!! note "Why this matters"
    Two participants who built readers for different instruments both targeting `NXsimple` produce entries that are discoverable together in the same search. This is the concrete benefit of agreeing on an application definition.

---

## 4.6 Analyse with NORTH

NORTH (NOMAD Remote Tools Hub) runs containerised Jupyter notebooks connected directly to your upload — no download needed.

1. From the entry **OVERVIEW** page, click **Analyse in NORTH** (or navigate to **Analyse → NORTH tools** from the top menu)
2. Choose a container. The generic **Jupyter** tool works for standard Python analysis.
3. NOMAD launches the container and mounts your upload directory. A Jupyter tab opens in the browser.
4. Access your file via the environment variable that NOMAD sets:

```python
import h5py, os

upload_path = os.environ.get("NOMAD_UPLOAD_PATH", ".")
with h5py.File(f"{upload_path}/output.nxs", "r") as f:
    data = f["entry/data/data"][()]
    print(data.shape)
```

5. Files you write back into the upload directory are stored in NOMAD. Click **Reprocess** on the upload page to re-index newly created entries.

!!! note
    NORTH availability depends on the NOMAD deployment. The public instance at `nomad-lab.eu` has NORTH enabled. Local deployments need a separate NORTH configuration.

---

## 4.7 Share and publish (optional)

| Action | How | Effect |
|---|---|---|
| Share with a collaborator | **EDIT UPLOAD MEMBERS** → add user as Coauthor or Reviewer | They can view (or edit) the upload |
| Make visible to everyone | Check **Visible to all** under Edit visibility | Searchable before publishing |
| Publish | **PUBLISH** button at the bottom | Immutable, permanently public |
| Get a DOI | Group entries into a Dataset, then publish the dataset | Citable with a persistent identifier |

!!! warning
    Once published, an upload cannot be deleted and files cannot be changed. Use the **test instance** to practice first.

---

## Checklist

- [ ] `output.nxs` uploaded and shows **processed** status
- [ ] Entry opens and the DATA tab shows the full NeXus hierarchy
- [ ] `NXsimple` filter in Explore → Entries returns your entry
- [ ] (Optional) NORTH Jupyter notebook opens and reads the data

---

## Further reading

- [NOMAD documentation > Upload and publish data](https://nomad-lab.eu/prod/v1/docs/tutorial/upload_publish.html)
- [NOMAD documentation > Explore data](https://nomad-lab.eu/prod/v1/docs/tutorial/explore.html)
- [pynxtools tutorial > NeXus files in NOMAD](https://fairmat-nfdi.github.io/pynxtools/tutorial/nexus-to-nomad/)
