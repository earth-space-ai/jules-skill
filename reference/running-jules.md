# Running JULES

This document covers four ways of running JULES, in increasing order of
sophistication:

1. Bare `jules.exe` against a directory of namelists
2. A Rose suite without Cylc (Rose just generates the namelists)
3. A Rose suite with Cylc (the production workflow on JASMIN, MONSooN,
   the Met Office VM, and the Met Office desktop)
4. The rose-stem regression battery

It also explains the two parallel modes (OpenMP, MPI) and how the namelist
directory is structured for single-site (Loobos, FLUXNET) versus gridded
(GSWP2, ERA-Interim) runs.

---

## The namelist directory

The user interface of JULES is a directory containing one file per
namelist with the extension `.nml`. JULES expects all 30+ files to exist
for every run, in the order defined by `src/initialisation/`. Files for
science you do not use can be empty stubs; the file itself must be
present.

Run JULES one of two ways:

```bash
# (a) Run from inside the namelist directory:
cd /path/to/namelist/dir
/path/to/jules.exe

# (b) Run with the namelist directory as the only argument:
/path/to/jules.exe /path/to/namelist/dir
```

**Critical detail.** Any relative paths inside the namelists (e.g. `file
= './forcing/Loobos.dat'` in `JULES_FRAC` or `drive.nml`) are interpreted
relative to the *current working directory*, not the namelist directory.

Practical implication:

- For portable suites, use paths relative to the namelist directory and
  always use form (a).
- For batch submission where the launching directory differs from the
  namelist directory, use absolute paths inside the namelists, or use
  form (a) by `cd`-ing inside the job script.

---

## Single-site runs (Loobos, FLUXNET)

Single-site runs are the standard development and validation workflow.
The reference site shipped with JULES is **Loobos** (Netherlands;
needleleaf forest, multi-year FLUXNET tower record). The example sits in
`examples/point_loobos/`.

A typical single-site namelist directory looks like:

```
namelist/
|-- jules_prnt_control.nml      <- diagnostic verbosity
|-- jules_surface_types.nml     <- npft, nnvg, surface type indices
|-- model_environment.nml       <- coupling environment (standalone here)
|-- jules_surface.nml           <- surface scheme switches
|-- jules_radiation.nml         <- canopy radiation switches
|-- jules_hydrology.nml         <- TOPMODEL / PDM
|-- jules_soil.nml              <- soil layers, hydraulic params
|-- jules_vegetation.nml        <- TRIFFID, phenology, crops
|-- jules_soil_biogeochem.nml   <- soil C+N model
|-- jules_soil_ecosse.nml       <- ECOSSE config (often empty)
|-- jules_deposition.nml        <- atmospheric deposition (often empty)
|-- jules_snow.nml              <- nsmax, snow params
|-- jules_rivers.nml            <- usually empty for single-site
|-- jules_water_resources.nml   <- usually empty
|-- jules_irrig.nml             <- irrigation
|-- science_fixes.nml           <- interim bug-fix flags
|-- timesteps.nml               <- start, end, dt, spin-up
|-- model_grid.nml              <- single-point grid
|-- ancillaries.nml             <- soil and topo ancillary files
|-- initial_conditions.nml      <- starting state
|-- drive.nml                   <- atmospheric forcing file paths
|-- urban.nml                   <- usually empty
|-- prescribed_data.nml         <- prescribed LAI etc.
|-- pft_params.nml              <- PFT parameter values
|-- nveg_params.nml             <- non-vegetation tile parameters
|-- triffid_params.nml          <- TRIFFID parameters (used if l_triffid)
|-- crop_params.nml             <- crop parameters
|-- imogen.nml                  <- IMOGEN emulator (often empty)
|-- fire.nml                    <- INFERNO fire (often empty)
|-- output.nml                  <- output profiles (this is the one you edit most)
\-- (cable_*.nml if l_cable)
```

For single-site runs:

- `model_grid.nml` declares one land point (`points_file`,
  `force_1d_grid`).
- `drive.nml` points at a multi-column ASCII file or a NetCDF time series
  with the seven required forcing variables (downward shortwave,
  downward longwave, precipitation, surface pressure, near-surface air
  temperature, near-surface specific humidity, near-surface wind).
- `timesteps.nml` sets `timestep_len` (commonly 1800 s for FLUXNET
  half-hourly forcing) and the start and end dates.
- `output.nml` configures the profiles you want to write.

To launch the run:

```bash
cd /path/to/namelist
$JULES_ROOT/build/bin/jules.exe
```

A 1-year half-hourly Loobos run finishes in roughly a minute on a
modern laptop / login node.

---

## Gridded runs (GSWP2, ERA-Interim)

Gridded runs use the same namelists but with a 2D `model_grid.nml` and
NetCDF forcing files. Two common references are:

- **GSWP2**: Global Soil Wetness Project Phase 2, 1986-1995, 1 degree
  global. Used as a benchmark in the JULES rose-stem suite.
- **ERA-Interim** (or its modern replacements ERA5 and ERA5-Land): the
  ECMWF reanalysis stream commonly used for global standalone runs.

Differences from single-site:

- `JULES_NETCDF=netcdf` is mandatory (gridded runs cannot use ASCII).
- `JULES_INPUT_GRID` declares the input grid topology; `JULES_LATLON`
  and `JULES_LAND_FRAC` provide latitude, longitude, and the land mask;
  `JULES_MODEL_GRID` declares the subdomain to actually run.
- For large domains, MPI is essential. The rose-stem `gswp2_*` apps run
  with `mpiexec -n N`.

---

## Building and running with FCM make

See `getting-started.md` for the full FCM environment-variable list. The
canonical commands are:

```bash
# Move into JULES root
cd ~/jules/vn7.4/trunk

# Build
fcm make -j 4 -f etc/fcm-make/make.cfg

# Move into the namelist directory
cd /path/to/namelist/dir

# Run
$JULES_ROOT/build/bin/jules.exe
```

For a `--new` clean build, append `--new` to `fcm make`. For incremental
rebuilds after editing one source file, drop `--new`; FCM make analyses
dependencies and rebuilds only what is needed.

---

## OpenMP

Compile with `JULES_OMP=omp`, then set the thread count at run time:

```bash
export OMP_NUM_THREADS=4
$JULES_ROOT/build/bin/jules.exe
```

OpenMP gives modest speedup at the loop level on shared-memory nodes; it
does not span machines.

---

## MPI

Compile with `JULES_MPI=mpi` and `JULES_NETCDF=netcdf` against a
parallel-I/O NetCDF + HDF5 build. Launch with the MPI runtime of your
cluster:

```bash
mpirun -n 16 $JULES_ROOT/build/bin/jules.exe
```

Decomposition is automatic; JULES picks a rectangular split of the model
grid based on the number of tasks. Each task reads its slice of the
input, runs the column physics, and writes its slice of the output. The
only inter-task communication is for dump file consistency.

Important property: dump files are bit-identical regardless of MPI
decomposition. A dump from any run can restart any other run with the
same overall model grid and produce identical results, irrespective of
the task counts used in either run.

---

## Rose suites

A JULES Rose suite contains two applications:

- `app/fcm_make/`: the build, parameterised by the same `JULES_*` env
  vars as a manual FCM build
- `app/jules/`: the run, with all the namelists wrapped behind a Rose
  Edit GUI and a `rose-app.conf`

### Without Cylc

If Cylc is not installed, Rose can still generate the namelists and you
run JULES manually:

```bash
rose app-run -i -C /path/to/rose/suite/app/jules
$JULES_ROOT/build/bin/jules.exe
```

`rose app-run -i` writes the namelist files to the current directory
based on the suite's metadata.

### With Cylc

The production form. From the suite root:

```bash
rose suite-run -C /path/to/rose/suite
```

This sets the suite running and launches the Cylc GUI to monitor task
status, view stderr/stdout, and re-trigger failed tasks.

### Creating a Rose suite from existing namelists

The `create_rose_app` utility converts a directory of plain namelists
into a Rose suite, optionally upgrading the namelists to a newer version
in the same step:

```bash
# From the directory containing your *.nml files:
$JULES_ROOT/utils/create_rose_app vn3.4 vn4.7 . my_suite $JULES_ROOT
```

Arguments: source version, target version, namelist path (full or
relative), suite name, JULES root. The suite lands in `~/roses/<name>`.

To upgrade in place without converting to a suite, pass the same
version twice for source and target.

### Upgrading an existing Rose suite

```bash
rose app-upgrade -M $JULES_ROOT/rose-meta -C path/to/suite/app/jules --all-versions
rose app-upgrade -M $JULES_ROOT/rose-meta -C path/to/suite/app/jules <version>
rose macro --fix -C path/to/suite/app/jules
```

The first call lists the upgrade chain. The second applies upgrade macros
in order. The third fixes any validators (default values, type coercion)
flagged by the rose-meta.

### Configuring with Rose Edit (the GUI)

```bash
# Edit the whole suite:
rose config-edit -M $JULES_ROOT/rose-meta -C path/to/suite

# Edit just the namelists:
rose config-edit -M $JULES_ROOT/rose-meta -C path/to/suite/app/jules
```

Click any variable name in the editor to jump to the matching page in the
JULES user guide. This is the recommended way to explore unfamiliar
options.

---

## rose-stem (regression test battery)

The rose-stem suite at `rose-stem/` is the gate for any commit to the
JULES trunk. It runs JULES against Loobos, GSWP2, and ERA-Interim
configurations, then compares the output against Known Good Output (KGO)
using `nccmp`.

```bash
# From the JULES working copy root:
cd ~/jules/vn7.4/trunk

# Run the full battery for your platform:
rose stem --group=all --new --name=my_full_test

# Or quick preliminary tests (tutorial subset):
rose stem --group=tutorial --new --name=my_quick_test
```

Available groups (configured in `rose-stem/suite.rc`):

| Group | Scope |
|-------|-------|
| `loobos` | Loobos single-site configurations |
| `gswp2` | Global GSWP2 gridded configurations |
| `linux` | Met Office Linux full set |
| `xc40` | Met Office Cray XC40 (and MONSooN) full set |
| `tutorial` | Subset for VM, JASMIN, cehlw1, NIWA |
| `tutorial_linux` | Met Office Linux tutorial subset |
| `tutorial_xc40` | Met Office XC40 tutorial subset |
| `all` | Everything appropriate to your platform |

The workflow per app is a chain:

```
fcm_make -> JULES configurations (Loobos, GSWP2, ERAI) -> nccmp vs KGO -> housekeep
```

KGO files live at `/jules/rose-stem-kgo/vn<X>.<Y>/` (Met Office) or in
the equivalent JASMIN / MONSooN location. `nccmp` failures must be
justified on the Trac ticket: a new configuration (no prior KGO), a bug
fix with plots showing improvement and module-leader approval, or
intentional new science with peer-reviewed evidence.

---

## Common run-time pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Cryptic Fortran read error at start | A namelist file is missing | Confirm all 30+ `*.nml` files exist; copy from `examples/` |
| `cannot open ... .nc` | Forcing file path resolved relative to wrong dir | Use absolute paths or run with form (a) `cd` into namelist dir |
| Output dates shifted | Driving data not at the timestamp JULES expects | Check `drive.nml` time origin and `data_period`; verify with `ncdump -v time` |
| Restart fails after code change | Dump file lacks new prognostic | Cold-start; do not restart across science changes |
| Different output between MPI task counts | None expected: MPI runs are bit-identical to serial | If you see drift, file a ticket; investigate compiler flags first |
| `Symbol __netcdf_MOD_*` undefined at link time | NetCDF Fortran library missing or compiler mismatch | See `getting-started.md` step 6 |

---

## Where to next

- Walk through the Loobos tutorial step by step: `tutorial-walkthrough.md`
- Pick physics switches for your science: `physics-options.md`
- Control output: `output-files.md`
- Diagnose failures: `debugging.md`
