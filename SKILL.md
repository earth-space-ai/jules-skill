---
name: jules
description: >
  Self-contained guide to JULES, the Joint UK Land Environment Simulator.
  Covers the Met Office MOSRS source workflow (FCM and Subversion), the
  Rose / Cylc suite system, single-site (Loobos, FLUXNET) and gridded
  (GSWP2, ERA-Interim) configurations, the namelist-driven user
  interface, the TRIFFID dynamic vegetation, MORUSES urban, RFM river
  routing and multi-layer snow physics options, the upgrade-macro
  workflow, and the UM coupling map (UM 11.5 through UM 13.4 against
  JULES vn5.6 through vn7.4). Progressive disclosure: start in this file
  for routing, drill into reference/ for depth.
version: 0.1.0
tags:
  - earth-science
  - hydrology
  - land-surface-model
  - fortran
  - jules
  - met-office
  - mosrs
  - rose-cylc
  - fcm
  - climate
  - unified-model
---

# JULES: Joint UK Land Environment Simulator

> **JULES** = Joint UK Land Environment Simulator
> Maintainer: Met Office (jules-support@metoffice.gov.uk)
> Source (authenticated): https://code.metoffice.gov.uk/trac/jules
> User guide: https://jules-lsm.github.io/
> GitHub mirror (announced 2025; private at time of writing, planned public): https://github.com/MetOffice/simulation-systems
> Best et al. 2011, GMD 4:677-699, doi:10.5194/gmd-4-677-2011 (energy and water)
> Clark et al. 2011, GMD 4:701-722, doi:10.5194/gmd-4-701-2011 (carbon and dynamic vegetation)
> Licence: BSD 3-Clause (vn7.x); JULES Terms and Conditions for legacy releases
> Latest user-guide release at time of writing: **vn7.9**

**What JULES does:** Solves the coupled land-surface energy, water, and
carbon balance over a column or 2D grid. Each gridbox is split into surface
tiles (broadleaf, needleleaf, C3 grass, C4 grass, shrub, urban, inland
water, bare soil, ice, plus user-extensible types). A separate energy
balance is computed per tile, then weighted to the gridbox mean. Soil
processes are modelled in a multi-layer column. Optional sub-models include
TRIFFID dynamic vegetation, the multi-layer snow scheme, the RFM kinematic
wave river routing, the MORUSES two-tile urban scheme, ECOSSE soil
biogeochemistry, the four-pool soil carbon model, and the IMOGEN climate
emulator.

**Who this skill is for:** Agents, students, and researchers who want to
request access to, check out, build, configure, run, debug, and contribute
to JULES, both in standalone mode and as the land surface of the Met Office
Unified Model (UM).

---

## Quick Decision Tree

```
"What do I need?"
|
|-- First time. What is JULES and how do I get the code?
|   Read: reference/getting-started.md
|   (MOSRS access form, FCM checkout, conda env, build prereqs)
|
|-- I want to understand how the code is laid out
|   Read: reference/architecture.md
|   (src/ tree, surface tiles, soil layers, snow, hydrology, river routing)
|
|-- I want to actually run a JULES simulation
|   Read: reference/running-jules.md
|   (FCM make, namelist directory, Rose suites, Loobos and FLUXNET)
|
|-- I need to choose physics options for my science
|   Read: reference/physics-options.md
|   (TRIFFID, multi-layer snow, RFM rivers, MORUSES urban, ECOSSE soils)
|
|-- I want to walk through the official tutorial cases
|   Read: reference/tutorial-walkthrough.md
|   (Loobos / Harvard Forest, JULES-Rose user and developer practicals)
|
|-- I need to control or extend model output
|   Read: reference/output-files.md
|   (output profiles, time-mean controls, model_interface_mod)
|
|-- I am working on a UM-coupled JULES branch
|   Read: reference/coupling-um.md
|   (UM 11.5 to UM 13.4 to JULES vn5.6 to vn7.4 mapping, fcm-keyword)
|
|-- It will not compile / will not run / output is wrong
|   Read: reference/debugging.md
|   (FCM dependency errors, namelist mismatches, dump restart drift)
|
\-- I want to submit my changes back to the trunk
    Read: reference/contributing.md
    (MOSRS Trac ticket, branch, rose-stem, code review committee)
```

---

## What's Inside JULES (vn7.x)

```
jules/                                      <- FCM working copy root
|-- src/                                    <- Fortran source
|   |-- control/                            <- top-level driver, time loop
|   |-- science/                            <- physics modules
|   |   |-- surface/                        <- surface energy balance, tiles
|   |   |-- soil/                           <- soil hydraulics, thermal
|   |   |-- snow/                           <- multi-layer snow
|   |   |-- vegetation/                     <- phenology, TRIFFID
|   |   |-- radiation/                      <- two-stream canopy radiation
|   |   |-- river_routing/                  <- TRIP and RFM
|   |   |-- urban/                          <- MORUSES two-tile urban
|   |   |-- biogeochem/                     <- ECOSSE, four-pool soil C
|   |   \-- diagnostics/
|   |-- io/                                 <- file_mod, model_interface_mod
|   |-- initialisation/                     <- read namelists, allocate state
|   \-- params/                             <- PFT and soil parameter modules
|-- etc/
|   \-- fcm-make/                           <- FCM build configuration files
|       \-- make.cfg                        <- top-level FCM make recipe
|-- rose-stem/                              <- regression test suite
|   |-- app/                                <- per-test apps
|   |-- bin/                                <- helper scripts
|   |-- include/
|   \-- suite.rc                            <- Cylc workflow definition
|-- rose-meta/                              <- metadata for Rose Edit GUI
|   |-- jules-fcm-make/                     <- build app metadata + macros
|   \-- jules-standalone/                   <- run app metadata + macros
|-- examples/                               <- sample namelists per release
\-- utils/                                  <- create_rose_app, helpers
```

For most users the only files you edit are the `*.nml` files in your run
directory (or via Rose Edit). Source edits live under `src/science/` plus
`src/io/model_interface_mod` if you add an output variable.

---

## Critical Rules

1. **Get MOSRS access first.** The JULES source lives on the Met Office
   Science Repository Service. New users must submit the request form at
   https://jules-lsm.github.io/access_req/JULES_access.html and wait up
   to two weeks. A GitHub mirror was announced in late 2025 but is still
   private. Without MOSRS credentials you cannot `fcm checkout` the trunk.

2. **Use FCM, not git, for the canonical workflow.** FCM is the Met
   Office build and version control wrapper around Subversion. Branches,
   merges, and trunk commits all happen through FCM keywords like
   `fcm:jules.x_tr` (trunk) and `fcm:jules.x_br` (branches). The GitHub
   mirror, when it goes public, will not replace MOSRS for the
   contribution flow.

3. **Run from the namelist directory, not the source tree.** Either
   `cd` into the directory containing `*.nml` and call `jules.exe` with no
   arguments, or pass the namelist directory as the only argument. Any
   relative paths inside the namelists are resolved against the current
   working directory, so portable suites use paths relative to the
   namelist directory and use the first form.

4. **All 30+ namelist files must exist for every run, even if empty.**
   JULES reads them in a fixed order. Missing files give a cryptic
   Fortran read error, not a friendly message. Use `examples/` or a Rose
   suite as the starting template, never write namelists from scratch.

5. **Use Rose to upgrade between versions.** Manual namelist edits across
   multiple JULES versions miss new switches and break old ones. Run
   `rose app-upgrade -M $JULES_ROOT/rose-meta -C app/jules <version>`
   followed by `rose macro --fix -C app/jules` to apply all upgrade
   macros and validators.

6. **rose-stem is the gate for trunk commits.** Every change must pass
   the `--group=all` rose-stem battery (Loobos, GSWP2, ERA-Interim,
   tutorial regression tests) on each supported platform (Met Office
   Linux, XC40, JASMIN). Compare new output against KGO (Known Good
   Output) using `nccmp`. Justified KGO changes need module leader
   approval on the Trac ticket.

7. **Match the JULES version to the UM version when coupling.** UM 13.4
   is built against JULES vn7.4; UM 11.5 against vn5.6; the symlink table
   in the user-guide tree (`um11.5 -> vn5.6`, ..., `um13.4 -> vn7.4`) is
   the authoritative map. Mismatched versions break the UM build.

8. **NetCDF is mandatory for gridded runs.** Without NetCDF, JULES
   builds against a dummy library and accepts only columnar ASCII for a
   single point. For MPI runs, NetCDF must be compiled with parallel I/O
   enabled (HDF5 + MPI).

---

## Quick Start (standalone JULES with FCM, single point)

```bash
# 1. Get MOSRS credentials (one-time, takes up to two weeks)
#    https://jules-lsm.github.io/access_req/JULES_access.html

# 2. Configure FCM keywords (one-time, see Rose user guide for details)
mkdir -p ~/.metomi/fcm
$EDITOR ~/.metomi/fcm/keyword.cfg
# Add the JULES MOSRS keywords as documented on the MOSRS Trac wiki.

# 3. Check out the trunk at the desired version
mkdir -p ~/jules/vn7.4
cd ~/jules/vn7.4
fcm checkout fcm:jules.x_tr@vn7.4 trunk
cd trunk

# 4. Set build environment variables
export JULES_PLATFORM=meto-linux-gfortran   # or your platform
export JULES_BUILD=normal                   # debug | normal | fast
export JULES_NETCDF=netcdf
export JULES_NETCDF_PATH=$(nc-config --prefix)

# 5. Build with FCM make
fcm make -j 4 -f etc/fcm-make/make.cfg --new
ls build/bin/jules.exe                      # should exist

# 6. Run an example (e.g. Loobos)
cd examples/point_loobos
$JULES_ROOT/build/bin/jules.exe             # reads ./*.nml

# 7. Inspect NetCDF output
ncdump -h output/loobos.Daily.nc | head
```

For Rose-based workflows (recommended for serious work) see
`reference/running-jules.md`.

---

## Reference Documents

| Document | What's inside |
|----------|---------------|
| `reference/getting-started.md` | MOSRS access request, FCM keyword setup, conda env for the user guide, prerequisites (gfortran or Intel, NetCDF, FCM, Rose, Cylc), first checkout, first build |
| `reference/architecture.md` | `src/` tree, the surface tile model, soil column layers, multi-layer snow, hydrology and runoff (TOPMODEL, PDM), river routing (RFM, TRIP), dust, MORUSES, ECOSSE, file_mod and model_interface_mod IO |
| `reference/running-jules.md` | FCM make environment variables, namelist directory rules, single-site (Loobos, FLUXNET) workflows, gridded (GSWP2) workflows, OpenMP and MPI launch, Rose suite execution with and without Cylc |
| `reference/physics-options.md` | TRIFFID dynamic vegetation, photosynthesis options, multi-layer snow (`nsmax`), thermal soil scheme, RFM and TRIP river routing, MORUSES urban, ECOSSE four-pool soil carbon, key switches |
| `reference/tutorial-walkthrough.md` | The Loobos tutorial case, the JULES-Rose User and Developer tutorials, what each practical does (Practical 1-6), JASMIN / VM / MONSooN setup |
| `reference/output-files.md` | Output profiles, JULES_OUTPUT and JULES_OUTPUT_PROFILE namelists, snapshot vs mean vs min/max/sum, dump files, adding a new output variable via `model_interface_mod` |
| `reference/coupling-um.md` | The UM-to-JULES version map (UM 11.5 to UM 13.4 maps to JULES vn5.6 to vn7.4), fcm-keyword configuration, the standalone-versus-coupled code split since vn3.1 |
| `reference/debugging.md` | FCM build dependency errors, missing namelist files, NetCDF link errors, restart drift, water and energy balance violations, soil moisture frozen all year, common Rose Edit pitfalls |
| `reference/contributing.md` | MOSRS Trac ticket workflow, branch naming, working practices, rose-stem with KGO, module leader review, code review committee, GitHub mirror plans |

---

## Where the documentation lives

- Full user guide (release-pinned): https://jules-lsm.github.io/
- vn7.4 chapter index used as the source for this skill: `https://jules-lsm.github.io/vn7.4/index.html`
- Latest at time of writing: vn7.9
- Tutorial portal (FCM and JULES-Rose): https://jules-lsm.github.io/tutorial/bg_info/tutorial_julesrose/index.html
- Coding standards: https://jules-lsm.github.io/coding_standards/
- MOSRS (authenticated): https://code.metoffice.gov.uk/trac/jules
- Mailing lists: jules-users@reading.ac.uk (science questions),
  jules@reading.ac.uk (announcements)
