# Debugging JULES

A field guide to the most common JULES failure modes (compile errors,
runtime crashes, output that looks wrong) and how to diagnose them
quickly. The pattern across all categories is: read the log, locate the
first error, isolate the offending namelist or source change, then
rebuild from a known-good baseline.

---

## 1. Build / compile errors

### `fcm: command not found`

FCM is not installed or not on `PATH`. See `getting-started.md` step 3.

```bash
which fcm
export PATH=~/fcm/bin:$PATH
fcm --version
```

### `fcm kp jules.x` returns nothing

FCM keyword config missing. Add the JULES MOSRS keyword block to
`~/.metomi/fcm/keyword.cfg`. The current snippet is on the JULES MOSRS
Trac wiki.

### `Authentication failed` on `fcm checkout`

MOSRS account not yet approved, or the credentials are not cached.

```bash
# Force a Subversion auth attempt, which will then cache
svn list https://code.metoffice.gov.uk/svn/jules/main/trunk
fcm checkout fcm:jules.x_tr@vn7.4 trunk
```

If the prompt rejects your password, the account is not active. Wait for
the approval email (up to two weeks); check spam first; then chase
jules-support@metoffice.gov.uk.

### `cannot find -lnetcdff` / `cannot find -lnetcdf`

NetCDF Fortran (or C) library missing or not on the link path. The
Fortran binding is shipped in a separate package on most distributions.

```bash
nf-config --flibs       # should print -lnetcdff -lnetcdf ...
nf-config --fflags      # include path
nc-config --prefix      # install prefix

# Either set JULES_NETCDF_PATH:
export JULES_NETCDF_PATH=$(nc-config --prefix)
# Or override the inc / lib paths individually:
export JULES_NETCDF_INC_PATH=/where/netcdf.mod/lives
export JULES_NETCDF_LIB_PATH=/where/libnetcdff.a/lives

fcm make -f etc/fcm-make/make.cfg --new
```

### `Symbol not found: __netcdf_MOD_*` at link time

NetCDF was built with a different compiler than your JULES build, or you
mixed module loads. Rebuild NetCDF with the same compiler, or `module
load` a matched NetCDF.

### `fcm make` fails inside the dependency analysis with "external file not found"

You added a new `USE` of an external library, but FCM make's dependency
analysis cannot resolve it. Add the external module name to the FCM make
exclusions (or put the module's include path in `JULES_FFLAGS_EXTRA`):

```
extract.location{primary}[some_external] = /path/to/external
```

In most cases, the simpler fix is to wrap the external call in an
internal module that JULES already knows about.

### `make` succeeds but `jules.exe` missing

A silent error in a later FCM stage. Re-run with verbose:

```bash
fcm make -v 2 -f etc/fcm-make/make.cfg --new 2>&1 | tee compile.log
grep -i '\[FAIL\]' compile.log | head
grep -i 'error' compile.log | head
```

The first `[FAIL]` is the root cause; everything after it cascades.

### Strange link errors after editing build env vars

Stale object files. Always rebuild with `--new` after changing compiler
flags or library paths:

```bash
fcm make -f etc/fcm-make/make.cfg --new
```

---

## 2. Namelist and runtime errors

### Cryptic Fortran read error at start-up

Almost always a missing `*.nml` file or a member that does not match the
namelist's source-side declaration. JULES expects all 30+ namelist files
to exist for every run.

Diagnose:

```bash
ls -la *.nml | wc -l       # should be 30+
diff <(ls *.nml) <($JULES_ROOT/examples/<a recent example>/*.nml | xargs -n1 basename)
```

Copy any missing file (even as an empty stub with just the namelist
opening and closing markers) from `examples/`.

### `cannot open ... .nc`

A path inside a namelist resolved to nothing. Two common causes:

1. The path is relative and you launched JULES from a directory other
   than the namelist directory. Either use absolute paths, or `cd` into
   the namelist directory before launching.
2. The path itself is wrong. Probe with `ls -la` from the directory
   JULES would resolve from.

### Output dates shifted

`drive.nml` time origin or `data_period` incorrect. Confirm with:

```bash
ncdump -v time forcing.nc | head
```

against the start time and timestep declared in `timesteps.nml`.

### `Incomplete dump file detected` when restarting

You changed physics options or upgraded JULES between the run that wrote
the dump and the run that is reading it. The new run requires
prognostics that the old dump does not contain.

Cold-start: comment out the dump path in `initial_conditions.nml` and
re-spin.

### Run hangs or is extremely slow

For gridded runs, check that you actually built MPI (`JULES_MPI=mpi`)
and launched with `mpirun`. A serial executable on a CONUS-class domain
looks like a hang for the first several hours.

For single-site runs that hang, the most common cause is a bad
hydrology configuration (e.g. zero soil moisture combined with
`l_top=T` causing the TOPMODEL solver to fail to converge). Add
`JULES_FFLAGS_EXTRA="-O0 -g -fcheck=all -fbacktrace -ffpe-trap=invalid,zero,overflow"`,
rebuild, re-run, and read the backtrace.

### "Branches have diverged" on rose-stem

KGO drift after a science change. See `contributing.md` for the
acceptable justifications and how to update the KGO with module-leader
approval.

---

## 3. Output looks wrong

### `latent_heat` zero or constant year-round

LAI input mismatch with the vegetation switches. With `l_phenol=F` and
`l_triffid=F`, JULES uses prescribed `lai_io` (or
`JULES_PRESCRIBED`); if neither is set, LAI is zero and transpiration
collapses. With `l_phenol=T`, the phenology model needs valid driving
PFT properties.

### Soil moisture stuck at saturation or wilting

Initial conditions in `initial_conditions.nml` outside the physical
range. Confirm with the dump from the first model timestep:

```bash
ncdump -v soil_moisture initial.dump.nc | head
```

Reset to a sensible start value (often the field capacity) and re-spin.

### Soil frozen all year in a temperate climate

Initial soil temperatures below freezing in `initial_conditions.nml`,
combined with no spin-up. JULES will not thaw a several-metre soil
column in a single timestep. Either provide warmer initial conditions or
add a spin-up cycle in `JULES_SPINUP`.

### Energy or water balance violation in the log

The diagnostic prints a per-timestep balance check. Common causes:

- A new prognostic was added without updating the balance accumulator.
- Snow mass not conserved between the canopy snowpack and the ground
  snowpack (a common bug when extending the snow model).
- River-routing exchange with the surface not accounted for in the
  surface water budget.

Localise by setting `iprnt = 9` in `jules_prnt_control.nml` to dump
per-tile balance terms.

### River discharge orders of magnitude wrong

`a_thresh` (RFM) or the routing-grid resolution mismatch. The JULES
documentation explicitly warns that RFM and TRIP parameters are
resolution-dependent and need testing per domain. Compare against
observations for a major catchment (Amazon, Congo, Mississippi) before
trusting global discharge.

### Output variable is all zeros or `_FillValue`

The variable was added in `model_interface_mod` but `extract_var.inc`
was not updated. See `output-files.md` step 4. Both `populate_var.inc`
(if you also want input) and `extract_var.inc` (for output) must be
edited symmetrically.

### Output is bit-different across MPI task counts

JULES is supposed to be bit-identical across decompositions. If you see
drift, the cause is almost always:

1. An uninitialised variable.
2. A reduction whose order depends on task layout (sum the values in a
   canonical order, not the one the loop happens to visit).
3. A compiler optimisation that violates IEEE 754 (drop
   `-ffast-math`, `-fp-model fast=2`, and friends).

Reproduce on a single node with `mpirun -n 1` vs `mpirun -n 4` and `cdo
diff` the dumps.

---

## 4. Helpful diagnostics

### Inspect a NetCDF output file

```bash
ncdump -h output/Loobos.Daily.nc          # header
ncdump -v gpp output/Loobos.Daily.nc | head -50
```

### Compare two output files variable-by-variable

```bash
nccmp -df run_a.nc run_b.nc
cdo diff run_a.nc run_b.nc
```

### Check restart contents

```bash
ncdump -h dump.spinup_end.nc
ncdump -v soil_moisture dump.spinup_end.nc | head
```

### Verbose FCM make

```bash
fcm make -v 2 -f etc/fcm-make/make.cfg --new 2>&1 | tee compile.log
grep -i '\[FAIL\]' compile.log
grep -i 'error' compile.log
```

### Inspect Rose suite logs

```bash
ls cylc-run/<suite>/log/job/1/<task>/NN/
less cylc-run/<suite>/log/job/1/<task>/NN/job.err
less cylc-run/<suite>/trac.log
```

### Source-tree status

```bash
cd ~/jules/vn7.4/trunk
fcm status
fcm log -l 5 .
```

---

## Where to next

- The overall execution flow you are debugging: `running-jules.md`
- The science switches that drive the values you see: `physics-options.md`
- Add diagnostics by exposing a new output variable: `output-files.md`
- File a ticket and propose a fix upstream: `contributing.md`
