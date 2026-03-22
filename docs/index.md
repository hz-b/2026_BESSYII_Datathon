# BESSY II Datathon 2026

Welcome - this site hosts the documentation for the BESSY II Datathon 2026, a hands-on lunch-to-lunch workshop with the goal of **standardizing science through FAIR data**

In this event you will turn raw instrument data into fully FAIR, community-standard NeXus files, upload them to a shared NOMAD instance, and make them searchable for anyone. By the end you will have a working data conversion pipeline for your own data.

## What you will learn

| | |
|---|---|
| **NeXus** | How the community file format works and why it makes data reusable |
| **NeXus application definitions** | Writing the schema that describes your own experiment |
| **pynxtools** | Python package for working with NeXus data |
| **NOMAD** | Upload, explore, filter, and share your data on a research data platform |

## Before we start

Use the "Getting started" page to setup your environment. Each challenge page contains the goal and the files you will receive.

- [Getting started](getting-started.md)

Optionally, you can already learn more about NeXus:

- [A primer on NeXus](nexus.md)

---

## Agenda

### Day 1 — Building a parsing pipeline

On day 1, we will build an end-to-end parsing pipeline for converting an example data set into NeXus and visualizing it in NOMAD.

| Time | Session | Page |
|------|---------|------|
| 12:00 | Registration and Welcome | — |
| 13:00 | Inspiration: Familiarize with NeXus and NOMAD | [Home](index.md) |
| 13:45 | **Session 0** — Setup for day 1 (15 min) | [Setup](example_pipeline/0-setup.md) |
| 14:00 | **Session 1** — Write a simple application definition (1 h) | [Application definitions](example_pipeline/1-appdef.md) |
| 15:00 | Coffee break | — |
| 15:30 | **Session 2** — Build a small pynxtools reader (1 h) | [Reader workshop](example_pipeline/2-reader.md) |
| 16:30 | **Session 3** — Upload to NOMAD (45 min) | [NOMAD](example_pipeline/3-nomad.md) |
| 17:15 | Feedback + open questions | — |
| 17:30 | End of Day 1 | — |

### Day 2 

On day 2, we split up:

- either you work on your own data or
- you work on the provided challenges 

| Time | Session |
|------|---------|
| 09:00 | Recap + open questions |
| 09:15 | **Session 4** — Working on challenges or your own data| [Challenges](challenges/overview) |
| 11:00 | Break | — |
| 11:15 | **Session 5** — Continue session 4 |
| 12:40 | Presentation — show and tell | — |
| 13:00 | Close | — |

#### Challenges

These are four data challenges on converting mySpot cuvette-scan experimental data into the NeXus format using `pynxtools`.

You can see them all on the overview page:

- [Challenges](challenges/overview.md)

#### Working on your own data

If you brought your own data, you can also work directly on converting those to NeXus.

Learn more on this page:

- [Bring your own data](your-own-data.md)


## Resources

| Resource | URL |
|---|---|
| pynxtools documentation | [fairmat-nfdi.github.io/pynxtools](https://fairmat-nfdi.github.io/pynxtools/){:target="_blank" rel="noopener"} |
| pynxtools plugin template | [github.com/FAIRmat-NFDI/pynxtools-plugin-template](https://github.com/FAIRmat-NFDI/pynxtools-plugin-template){:target="_blank" rel="noopener"} |
| NeXus definitions | [fairmat-nfdi.github.io/nexus_definitions](https://fairmat-nfdi.github.io/nexus_definitions/){:target="_blank" rel="noopener"} |
| NeXus manual | [manual.nexusformat.org](https://manual.nexusformat.org){:target="_blank" rel="noopener"} |
| NOMAD | [nomad-lab.eu](https://nomad-lab.eu/prod/v1/gui){:target="_blank" rel="noopener"} |


<!-- Optional: badge linking to site preview or CI status -->