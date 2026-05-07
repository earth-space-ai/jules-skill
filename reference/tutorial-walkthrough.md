# JULES Tutorial Walkthrough

This document walks through the official JULES training material that
new users are expected to complete before writing their own suites or
contributing changes. There are three tracks:

1. **FCM Fundamentals**: version control concepts, Subversion, FCM. Lives
   under `tutorial/bg_info/tutorial/`.
2. **JULES-Rose User**: how to check out a suite, edit ancillaries,
   change the radiation model, plot the output. Lives under
   `tutorial/bg_info/tutorial_julesrose/jr_user.html`.
3. **JULES-Rose Developer** (and Developer Advanced): how to make code
   changes, run rose-stem, manage tickets, write upgrade macros, work
   with HoT (head-of-trunk) branches.

The single best worked simulation case is the **Loobos** single-site
configuration shipped with every release as a rose-stem app.

---

## Pre-requisites checklist

Before starting any of the tracks the user must have:

- An approved MOSRS account (see `getting-started.md`).
- Rose, Cylc, and FCM installed and on `PATH`. The standard supported
  environments are:
  - The JULES VM (Ubuntu image with everything pre-installed; download
    instructions on the JULES MOSRS wiki).
  - JASMIN (UKRI scientific computing service): use the Cylc server and
    the `jasmin-lotus-intel` or `jasmin-gcc-nompi` JULES platforms.
  - MONSooN (Met Office and NERC supercomputer): use the
    `meto-xc40-cce` platform.
  - The Met Office desktop Linux fleet.
- FCM keywords for `jules.x`, `metomi`, and friends configured in
  `~/.metomi/fcm/keyword.cfg`.
- Read the Rose user guide at https://metomi.github.io/rose/doc/html/.

The tutorial portal index (`jules_logo.png` top-right) lists three quizzes
(pre-course, user, developer, developer advanced) that book-end the
practicals.

---

## Track 1: FCM Fundamentals

Six pages under `tutorial/bg_info/tutorial/`:

| Page | Topic |
|------|-------|
| `fcm_fundamentals.html` | Why version control, Subversion basics, scenarios, configuration, setting up your own and the tutorial repository |
| `fcm_essentials.html` | Day-to-day FCM commands: checkout, status, diff, commit, branch, merge |
| `fcm_specifics.html` | Advanced workflows: cherry-pick, conflict resolution, branch-of-branch |
| `fcm_glossary.html` | Vocabulary |
| `fcm_links.html` | Links to upstream FCM docs |
| `fcm_mqrg.html` | "My quick reference guide" template |

Plus three quizzes (`fundamentals_quiz.html`, `essentials_quiz.html`,
`specifics_quiz.html`).

The practicals require a sandbox repository created with the setup
script (only needed for Essentials and Specifics).

---

## Track 2: JULES-Rose User

The User track at `tutorial/bg_info/tutorial_julesrose/jr_user.html` has
five tutorials and six practicals:

### User tutorials

1. **Background of JULES**: what JULES is, why use it, who uses it.
   JULES is the Met Office community Land Surface Model (LSM), used both
   standalone (forcing-driven) and coupled inside the Unified Model
   (UM).
2. **Background of Rose (in fact Cylc)**: Cylc is an intelligent
   workflow scheduler created at NIWA New Zealand; Rose is the user-
   friendly wrapper the Met Office wrote around it.
3. **Background of MOSRS**: the Met Office Science Repository Service.
4. **Subversion**: the underlying VCS that FCM wraps.
5. **JULES Rose stem**: the regression test battery.
6. **JULES Rose suites**: structure of a `app/jules` plus `app/fcm_make`
   suite.

### User practicals

| Practical | Topic |
|-----------|-------|
| 1 | Background |
| 2 | Basic FCM (checkout, status, log, diff) |
| 3 | Checkout JULES code |
| 4 | Checkout an existing Rose suite |
| 5 | Change ancillaries (swap soil or vegetation properties) |
| 6 | Change the radiation model and plot the output |

After Practical 6 the user has a working Rose suite, a build of JULES, a
JULES run with modified physics, and Python (or IDL / R) plotting code
that consumes the NetCDF output.

---

## Track 3: JULES-Rose Developer

The Developer track at `tutorial/bg_info/tutorial_julesrose/jr_developer.html`
covers the mechanics of contributing code:

### Developer tutorials

1. **Basics of JULES rose-stem**. The regression test battery and how to
   run it. The principal command is `rose stem --group=all`.
   Quick-test groups: `tutorial`, `tutorial_linux`, `tutorial_xc40`.
2. **Basics of Coding (UMDP3, reviewing)**. UMDP3 is the Met Office's
   Unified Model Documentation Paper 3, which is the parent coding
   standard JULES inherits from. JULES coding standards layer on top
   (https://jules-lsm.github.io/coding_standards/).
3. **Running a basic JULES-Rose suite**.

### Developer practicals

| Practical | Topic |
|-----------|-------|
| Further JULES Rose Edit | Using the GUI for non-trivial edits |
| Further JULES-Rose suites: duration, domain | Changing run length and gridded domain |
| Further JULES rose-stem: namelists, modules | Adding new test apps |
| More on Tickets: populating | Trac ticket fields, evidence |
| HoT branches and merging branch into it | Working with head-of-trunk branches |
| UM-JULES | The coupled-mode workflow |
| Metadata | Authoring rose-meta for new namelist members |
| Upgrade macros | Authoring upgrade macros so future versions can
auto-upgrade old suites |

### Module leaders

Each science module has named leaders who must be Cc'd on tickets that
touch their area:

| Module | Leaders |
|--------|---------|
| Surface | John Edwards (john.m.edwards@metoffice.gov.uk), Richard Essery (richard.essery@ed.ac.uk) |
| Hydrology | Nic Gedney (nicola.gedney@metoffice.gov.uk), Anne Verhoef (a.verhoef@reading.ac.uk) |
| Vegetation | Anna Harper (a.harper@exeter.ac.uk), Lina Mercado (l.mercado@exeter.ac.uk) |
| Biochemistry | Eleanor Burke (eleanor.burke@metoffice.gov.uk), Sarah Chadburn (sarah.chadburn@metoffice.gov.uk / s.e.chadburn@exeter.ac.uk) |
| Biogenic Fluxes | Gerd Folberth (gerd.folberth@metoffice.gov.uk), Oliver Wild (o.wild@lancaster.ac.uk) |
| Technical Changes | TBC |
| Evaluation | Tristan Quaife (t.l.quaife@reading.ac.uk), Graham Weedon (graham.weedon@metoffice.gov.uk) |

These names were current at the time of the v7.4 user guide snapshot.
Cross-check the JULES MOSRS Trac wiki for the current list before
opening a ticket.

---

## Worked Loobos example

The Loobos site is a Dutch coniferous forest with a long FLUXNET tower
record. It is the canonical single-site test case shipped with JULES.

### The rose-stem Loobos app

```
rose-stem/app/loobos/
|-- rose-app.conf
|-- file/
|   |-- ancillaries.nml
|   |-- drive.nml
|   |-- initial_conditions.nml
|   |-- model_grid.nml
|   |-- output.nml
|   |-- timesteps.nml
|   \-- jules_*.nml (the standard 30+)
\-- opt/                  <- optional overrides per variant
```

Run it from the JULES root:

```bash
cd ~/jules/vn7.4/trunk
rose stem --group=loobos --new --name=loobos_check
```

The suite builds JULES (`fcm_make`), runs each Loobos variant, then runs
`nccmp` against the KGO. A successful run shows green tasks for every
`fcm_make`, every `loobos_*`, and every `nccmp_loobos_*`.

### Manual Loobos run

If you do not want the Cylc workflow:

```bash
cp -r examples/point_loobos /scratch/me/loobos_run
cd /scratch/me/loobos_run

# Edit timesteps.nml and drive.nml to point at the actual forcing path.
# Edit output.nml to choose what variables you want.

$JULES_ROOT/build/bin/jules.exe
ls output/
```

Output lands in NetCDF (or columnar ASCII if NetCDF was disabled). Use
`ncview`, `ncdump`, `xarray`, or your tool of choice to inspect.

---

## Sanity-checking variables

When eyeballing a Loobos run, a healthy result has:

- `latent_heat` (W m-2) tracking the diurnal and seasonal cycle, peaking
  in mid-summer noon at a few hundred W/m2.
- `sensible_heat` (W m-2) similar shape, smaller magnitude in summer at
  this site.
- `gpp` (kg C m-2 s-1) > 0 during daylight in the growing season.
- `soil_moisture` per layer staying within physically reasonable bounds
  (no values below the wilting point or above saturation).
- `snow_depth` (m) building in winter, melting in spring, zero in summer.

Compare against published Loobos results in the JULES literature
(Best et al. 2011 GMD is the standard reference).

---

## Where to next

- Build a custom suite: `running-jules.md`
- Pick non-default physics: `physics-options.md`
- Push your changes back: `contributing.md`
- Diagnose Loobos that comes out wrong: `debugging.md`
