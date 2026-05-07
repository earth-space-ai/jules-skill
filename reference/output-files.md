# JULES Output Files

JULES output is grouped into one or more **profiles**, each of which is
written to its own file at its own frequency. Profiles can hold either
model-grid variables or river-routing-grid variables, but never both.
Output is NetCDF when JULES is built with a real NetCDF library, and
columnar ASCII otherwise.

This page covers the namelist controls that shape output, the time
semantics of each output type, dump (restart) files, and the procedure
for adding a new variable to the output catalogue.

---

## Output profiles

`output.nml` contains exactly one `JULES_OUTPUT` namelist at the top,
followed by `nprofiles` instances of `JULES_OUTPUT_PROFILE`.

### `JULES_OUTPUT` (one occurrence)

| Member | Type | Notes |
|--------|------|-------|
| `output_dir` | character | Output directory. Absolute or relative to CWD. |
| `run_id` | character | Tag inserted in every output file name. |
| `nprofiles` | integer | Number of `JULES_OUTPUT_PROFILE` namelists below. |
| `dump_period` | integer | Period between model dumps. Units depend on `dump_period_unit`. |
| `dump_period_unit` | character(1) | `'Y'` (years, default) or `'T'` (seconds of calendar day). |

Note on `dump_period`. In `'Y'` mode, dumps land on calendar-year
boundaries: a run starting in 2012 with `dump_period=5` writes dumps at
the start of 2015, 2020, 2025, ... In `'T'` mode it counts seconds into
the calendar day, so `dump_period=10800` writes dumps at T00:00, T03:00,
... T21:00. Dumps are also always written at the start and end of the
main run and at the start of each spin-up cycle.

### `JULES_OUTPUT_PROFILE` (one per profile)

| Member | Purpose |
|--------|---------|
| `profile_name` | Name embedded in output filenames. Make it unique. Conventional names reflect content (`carbon`, `water`) or frequency (`daily`, `monthly`). |
| `output_period` | Output period in seconds, or one of the special values `-1` (calendar months) and `-2` (calendar years). |
| `output_main_run` | Toggle output during the main run. |
| `output_spinup` | Toggle output during spin-up. |
| `output_initial` | Toggle a separate initial-state file per section. Only snapshot values for state variables are valid in the initial-state file; non-state variables exist in the file but contain garbage. |
| `output_start`, `output_end` | Restrict the profile to a sub-window of the main run (e.g. a special observation period). |
| `var` | List of variable identifiers to write. |
| `var_name` | Optional override of the output variable name (defaults to the identifier). |
| `output_type` | Per-variable: `'S'` snapshot, `'M'` mean, `'N'` minimum, `'X'` maximum, `'A'` accumulation. |
| `l_land_frac` | Toggle inclusion of the land-fraction field. |

Each variable carries a CF `cell_methods` attribute that records the
time-processing: `time: point` for snapshots, `time: mean` for means,
`time: minimum`, `time: maximum`, `time: sum` for accumulations.

---

## Time semantics

Each output file holds two time-related coordinates:

- `time`: for each output period, the time at the **end** of the output
  period. This is the time at which any snapshot value applies.
- `time_bounds`: two values per output period, the start and end. Means,
  minima, maxima, and accumulations are computed over the half-open
  interval `time_bounds(1) < time <= time_bounds(2)`.

JULES captures values for output at the **end** of each model timestep,
after all the science code has run. This means snapshot output at a
given timestep contains:

- The **state** of the model at the end of the timestep.
- The **fluxes** that produced that state over the timestep.

By construction, all variables in a snapshot at time t are physically
consistent.

---

## Dump (restart) files

JULES writes a dump file at:

- After initialisation, immediately before the start of the run
  (initial-state dump).
- Before starting each cycle of spin-up.
- Before starting the main run.
- At the end of the run (final-state dump).
- At the start of each calendar year.
- At the period set by `dump_period` and `dump_period_unit`.

Each dump is named with the model date and time at which it was written.
From vn4.3 onwards, dump files include ancillary data so the model can
optionally restart from the dump rather than re-reading the ancillary
namelists. Latitude and longitude are also written but not read, to help
debugging.

Dumps are bit-identical regardless of MPI decomposition, so a dump
written by an N-task run can restart an M-task run with the same overall
grid and produce identical results.

---

## Output formats

| Build | Output format |
|-------|---------------|
| `JULES_NETCDF=netcdf` | NetCDF-4 (HDF5 underneath). MPI-parallel I/O if NetCDF was built with parallel HDF5. |
| `JULES_NETCDF=nonetcdf` | Columnar ASCII with headers; single-point only. |

NetCDF output carries CF-1.x metadata, including `cell_methods`,
`standard_name`, `units`, and time bounds. The output is suitable for
direct consumption by `xarray`, `cdo`, `nccmp`, and the wider analysis
ecosystem.

---

## Built-in output variables

The full catalogue (over 200 variables) is documented at
`https://jules-lsm.github.io/vn7.4/output-variables.html`. Categories
include:

- Surface energy balance: `latent_heat`, `sensible_heat`, `surf_ht_flux`,
  `radnet`, `swnet`, `lwnet`
- Hydrology: `precip`, `rainfall`, `snowfall`, `runoff`, `sub_surf_roff`,
  `surf_roff`, `et_stom`, `ecan`, `esoil`, `transp`
- Soil: `smc_avail_top`, `smc_avail_tot`, `soil_moisture`, `t_soil`,
  `unfrozen_moist`, `frozen_moist`
- Snow: `snow_amount`, `snow_depth`, `snow_grnd`, `snow_can`, `swe_snow`
- Carbon: `gpp`, `npp`, `resp_p`, `resp_s`, `cs`, `cv`, `lai`,
  `canht_ft`
- Surface state: `tstar`, `t1p5m`, `q1p5m`, `surface_temperature`
- River routing (separate-grid profiles): `rivers_sto_rp`,
  `rivers_dis_rp`

Profiles can mix as many as you want from the catalogue; only the file-
size budget and the time semantics constrain the choice.

---

## Adding a new output variable

The vn3.1 I/O refactor concentrated all model-to-IO communication in
`src/io/model_interface_mod`. To expose a new variable for output (and
optionally for input), the only edits required are in that module.

### Step 1: make the variable accessible to `model_interface_mod`

The variable must live in a module that `model_interface_mod` can
`USE`:

```fortran
! In src/science/.../my_module.F90:
MODULE my_module
  REAL, ALLOCATABLE :: my_var(:)
  ! ... your science code that updates my_var ...
END MODULE my_module
```

```fortran
! In src/io/model_interface/model_interface_mod.F90:
USE my_module, ONLY : my_var
```

If the variable is already in a module imported by
`model_interface_mod`, skip this step.

### Step 2: bump the variable count

In `model_interface_mod.F90`, increment the constant `N_VARS`. The
metadata array is sized by this parameter; if you forget to increment
it, the module will fail to compile (and you will be glad it did).

### Step 3: edit `populate_var.inc` (input only)

For variables you want to accept on input, add a SELECT case to
`populate_var.inc`. The routine takes a variable identifier and a cube
of data on the full model grid, then assigns into the model state.

### Step 4: edit `extract_var.inc` (output only)

For variables you want to write on output, add a SELECT case to
`extract_var.inc`. The routine takes a variable identifier, reads from
the model state, and returns a cube of data on the full model grid.

### Step 5: edit `variable_metadata.inc` (always)

Add a new DATA element of type `var_metadata`. Example for the existing
`latitude` variable:

```fortran
DATA metadata(1) / var_metadata(            &
  'latitude',                              & ! string identifier (used in output.nml var=...)
  VAR_TYPE_SURFACE,                         & ! variable type / levels dimensions
  "Gridbox latitude",                       & ! long name
  "degrees"                                 & ! units
) /
```

Variable types determine the levels dimension(s):

| Type | Levels dimension |
|------|------------------|
| `VAR_TYPE_SURFACE` | none |
| `VAR_TYPE_PFT` | `npft` |
| `VAR_TYPE_NVG` | `nnvg` |
| `VAR_TYPE_TYPE` | `ntype` (= `npft + nnvg`) |
| `VAR_TYPE_TILE` | `nsurft` (1 if `l_aggregate=T`, else `ntype`) |
| `VAR_TYPE_SOIL` | `sm_levels` |
| `VAR_TYPE_SCPOOL` | `dim_cs1` (4 if `l_triffid`, else 1) |

The full type list is in `get_var_levs_dims.inc`.

### Step 6: rebuild and request the variable

```bash
cd ~/jules/vn7.4/trunk
fcm make -f etc/fcm-make/make.cfg     # incremental, picks up your edits
```

Then in your run namelists, add the new identifier to a profile's `var`
list:

```fortran
&JULES_OUTPUT_PROFILE
  profile_name      = 'mine',
  output_period     = -1,                ! monthly
  output_main_run   = T,
  output_spinup     = F,
  nvars             = 3,
  var               = 'my_new_var', 'gpp', 'lai',
  output_type       = 'M', 'M', 'M',
/
```

Run, and the new variable appears in `mine.<run_id>.<period>.nc`.

---

## Implementing a new file format

The `file_mod` interface defines what a file driver must implement
(open, close, read, write, seek, advance, define dimensions and
variables, ...). The two existing drivers are `file_ascii_mod` and
`file_ncdf_mod`. To add a third (e.g. Zarr or Parquet), implement the
interface, declare the implementation in `file_mod`, and JULES will
dispatch reads and writes to your driver based on file extension or an
explicit type flag in the namelist.

The user guide notes that this work has not been done for any third
backend; both existing drivers cover the practical use cases.

---

## Where to next

- Run a configuration that uses these profiles: `running-jules.md`
- Decide which physics produces the variables you want: `physics-options.md`
- Diagnose missing or wrong-valued output: `debugging.md`
- Submit your new output variable upstream: `contributing.md`
