# Getting Started with JULES

This guide walks from zero to a built `jules.exe` checked out from the Met
Office Science Repository Service (MOSRS). After this, see
`running-jules.md` to launch a real simulation and `tutorial-walkthrough.md`
for the official tutorial cases.

---

## Step 1: Understand where the code lives

JULES is hosted by the Met Office, not on a public GitHub. The
authoritative repository is on the Met Office Science Repository Service
(MOSRS), accessed through Subversion via the FCM wrapper.

| Location | What it is | URL |
|----------|-----------|-----|
| MOSRS Trac (auth-walled) | Source repository, branches, tickets | https://code.metoffice.gov.uk/trac/jules |
| User guide (public) | HTML user guide for every release | https://jules-lsm.github.io/ |
| Tutorial portal (public) | FCM and JULES-Rose tutorials | https://jules-lsm.github.io/tutorial/bg_info/ |
| Coding standards (public) | Fortran style guide | https://jules-lsm.github.io/coding_standards/ |
| GitHub mirror (announced 2025) | Read-only mirror, currently private | https://github.com/MetOffice/simulation-systems |

The FCM keyword `fcm:jules.x` resolves to the JULES MOSRS repository. The
trunk is `fcm:jules.x_tr`; branches are `fcm:jules.x_br`.

---

## Step 2: Request MOSRS access

JULES is freely available for non-commercial use but requires an account.
New users complete the request form at
https://jules-lsm.github.io/access_req/JULES_access.html. The form asks
for institutional email, affiliation, and a brief description of intended
use ("for land-surface modelling" alone is rejected; be specific). Account
creation can take up to two weeks; check the spam folder for the
confirmation email before chasing jules-support@metoffice.gov.uk.

A GitHub account is also required for new development since the late 2025
announcement of the GitHub mirror, even though that mirror is still
private.

The licence (`JULES_Licence.pdf`) must be accepted at submission time.

---

## Step 3: Install prerequisites

JULES has been tested on Linux. Cygwin is documented but untested for
Windows. macOS works with the Homebrew toolchain in practice but is not
officially supported.

### Fortran compiler

A modern compiler with Fortran 2003 support:
- **gfortran** (GCC, free): the default for `JULES_PLATFORM=meto-linux-gfortran`, `ceh`, `vm`, `uoe-linux-gfortran`
- **Intel ifort or ifx**: `JULES_PLATFORM=meto-linux-intel-mpi`, `meto-linux-intel-nompi`, `jasmin-lotus-intel`
- **NAG nagfor**: correctness checking only, not for production
- **Cray Compiler Environment**: `meto-xc40-cce` for the Cray XC40

### NetCDF (and Fortran bindings)

Required for any gridded run, strongly recommended even for single-point
because the analysis ecosystem (xarray, ncview, cdo, nccmp) all consume
NetCDF natively.

```bash
# macOS
brew install netcdf netcdf-fortran

# Ubuntu / Debian
sudo apt-get install libnetcdf-dev libnetcdff-dev

# Probe an existing install
nc-config --prefix
nc-config --flibs
nf-config --fflags
```

For MPI runs you must build NetCDF and HDF5 with parallel I/O enabled
against the same MPI implementation as your JULES build. This is not the
default for distro packages; see the NetCDF docs for the
`--enable-parallel-tests` flag.

### FCM (Met Office build / version control wrapper)

```bash
git clone https://github.com/metomi/fcm.git ~/fcm
export PATH=~/fcm/bin:$PATH
fcm --version
```

Documentation: http://metomi.github.io/fcm/doc

### Rose and Cylc (optional but standard)

Required to use the GUI, the upgrade macros, the rose-stem regression
tests, and the JULES suite system.

```bash
git clone https://github.com/metomi/rose.git ~/rose
git clone https://github.com/cylc/cylc-flow.git ~/cylc
export PATH=~/rose/bin:~/cylc/bin:$PATH
rose --version
cylc --version
```

Cylc 8 is the current major version; Rose 2 is the matching wrapper. The
JULES-Rose tutorial assumes both are installed and configured.

### Conda environment for the user guide build

If you want to build the HTML user guide locally from the JULES source
(useful for offline reference), the repository ships an
`environment.yml`:

```bash
conda env create -f environment.yml
conda activate jules-user-guide
cd user_guide/doc
make html
firefox build/html/index.html
```

---

## Step 4: Configure FCM keywords

FCM keywords map short names like `jules.x` to MOSRS URLs. The JULES MOSRS
Trac wiki has the canonical fragment to paste into
`~/.metomi/fcm/keyword.cfg`. A typical layout looks like:

```
location{primary, type:svn}[jules.x]            = https://code.metoffice.gov.uk/svn/jules/main
browser.loc-tmpl[jules.x]                       = https://code.metoffice.gov.uk/trac/jules/browser/{1}{2}
browser.comp-pat[jules.x]                       = (?msx-i:\A // [^/]+ /svn/ [^/]+ /(.*) \z)
browser.rev-tmpl[jules.x]                       = https://code.metoffice.gov.uk/trac/jules/changeset/{1}
```

Probe with:

```bash
fcm kp jules.x
fcm kp jules.x_tr
```

Both should resolve to MOSRS URLs without error.

---

## Step 5: Check out the trunk

```bash
mkdir -p ~/jules/vn7.4
cd ~/jules/vn7.4
fcm checkout fcm:jules.x_tr@vn7.4 trunk
cd trunk
ls
# expects: src/  etc/  rose-stem/  rose-meta/  examples/  utils/  ...
```

To check out the head of trunk (development tip), drop the `@vn7.4`. To
check out a branch, use `fcm:jules.x_br/dev/<owner>/<branchname>`.

---

## Step 6: Build with FCM make

The build is configured by environment variables consumed by
`etc/fcm-make/make.cfg`. The principal variables are:

| Variable | Purpose | Common values |
|----------|---------|---------------|
| `JULES_PLATFORM` | Pre-defined platform settings | `custom`, `vm`, `ceh`, `jasmin-lotus-intel`, `jasmin-gcc-nompi`, `meto-linux-gfortran`, `meto-linux-intel-mpi`, `meto-xc40-cce`, `uoe-linux-gfortran` |
| `JULES_COMPILER` | Compiler-specific flags | `gfortran`, `intel`, `nagfor`, `cray` |
| `JULES_BUILD` | Build type | `normal`, `debug`, `fast` |
| `JULES_OMP` | OpenMP toggle | `noomp`, `omp` |
| `JULES_MPI` | MPI toggle | `nompi`, `mpi` |
| `JULES_NETCDF` | NetCDF on or dummy | `nonetcdf`, `netcdf` |
| `JULES_NETCDF_PATH` | NetCDF install root | e.g. `$(nc-config --prefix)` |
| `JULES_NETCDF_INC_PATH` | Override `$JULES_NETCDF_PATH/include` | seldom needed |
| `JULES_NETCDF_LIB_PATH` | Override `$JULES_NETCDF_PATH/lib` | seldom needed |
| `JULES_FFLAGS_EXTRA` | Extra compiler flags | `"-check bounds"` (Intel), `"-fcheck=all"` (gfortran) |
| `JULES_LDFLAGS_EXTRA` | Extra linker flags | extra `-L`, `-l` |
| `JULES_REMOTE` | Build on remote machine | `local`, `remote` |
| `JULES_SOURCE` | Source path or URL | used by Rose `fcm_make` apps |

Print the current build environment:

```bash
env | grep JULES
```

Build commands:

```bash
# Generic build, no NetCDF (single point only, ASCII I/O):
fcm make -j 2 -f etc/fcm-make/make.cfg --new

# Fast build with NetCDF using Intel compiler:
export JULES_COMPILER=intel
export JULES_BUILD=fast
export JULES_NETCDF=netcdf
export JULES_NETCDF_PATH=/path/to/netcdf
fcm make -j 2 -f etc/fcm-make/make.cfg --new

# Met Office Linux gfortran with NetCDF (no path needed):
export JULES_PLATFORM=meto-linux-gfortran
export JULES_BUILD=fast
export JULES_NETCDF=netcdf
fcm make -j 2 -f etc/fcm-make/make.cfg --new

# MPI with bounds checking (Intel):
export JULES_COMPILER=intel
export JULES_MPI=mpi
export JULES_NETCDF=netcdf
export JULES_NETCDF_PATH=/path/to/parallel/netcdf
export JULES_FFLAGS_EXTRA="-check bounds"
fcm make -j 2 -f etc/fcm-make/make.cfg --new
```

Output:

```bash
ls build/bin/jules.exe
# absolute path, ~10-20 MB
```

If you change compiler flags or libraries, drop `--new` and use
`fcm make --ignore-lock` only after checking that incremental builds still
pick up the changes; the safest path is always a fresh `--new`.

---

## Step 7: Smoke test

```bash
cd examples/point_loobos     # or any example shipped with the release
$JULES_ROOT/build/bin/jules.exe
```

If the namelist directory is wired correctly, the model prints a banner
with the version, surface tile counts, soil-layer thicknesses, then
iterates through the simulation period. Output lands in `output/`.

For the Loobos walkthrough see `tutorial-walkthrough.md`.

---

## Common install pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `fcm: command not found` | FCM not installed or not on PATH | Install FCM, add `~/fcm/bin` to PATH |
| `fcm kp jules.x` returns nothing | FCM keyword config missing | Add JULES MOSRS keywords to `~/.metomi/fcm/keyword.cfg` |
| `Authentication failed` on `fcm checkout` | MOSRS account not approved or password not cached | Wait for account confirmation; cache password with `svn list` once |
| `cannot find -lnetcdff` | NetCDF Fortran binding missing | Install `netcdf-fortran` (separate from C library) |
| `Symbol not found: __netcdf_MOD_*` | Compiler / library mismatch (e.g. Intel build against gfortran-built NetCDF) | Rebuild NetCDF with the same compiler, or pick a matching `module load` |
| `Incomplete dump file detected` | Restart attempted after a code change that added a new prognostic | Cold-start; do not restart across science changes |
| `fcm make` succeeds but `jules.exe` missing | Earlier silent error, log not captured | Re-run with `fcm make -v 2`, search log for first `[FAIL]` |

---

## Where to next

- Run the model: `running-jules.md`
- Walk through the Loobos / JULES-Rose tutorial: `tutorial-walkthrough.md`
- Understand the source layout: `architecture.md`
- Pick physics options for your science: `physics-options.md`
