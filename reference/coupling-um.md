# Coupling JULES to the Met Office Unified Model

JULES is the land surface scheme of the Met Office Unified Model (UM).
Since JULES vn3.1 a single source repository serves both standalone
JULES and JULES-in-the-UM. The UM build pulls in a specific JULES code
version, and the two are released together. Mismatched versions break
the UM build.

This page documents the version-mapping table, the `fcm-keyword`
configuration, the historical split, and the standalone-vs-coupled code
boundary.

---

## Historical context

JULES grew out of MOSES (the Met Office Surface Exchange Scheme).
According to the v7.9 user guide release notes:

> After that initial split, the MOSES and JULES code bases evolved
> separately, but with JULES v2.1 these differences were reconciled with
> the UM. As of JULES v3.1, a single code repository is used for both
> standalone JULES and JULES in the UM.

Practical implications since vn3.1:
- One MOSRS Trac repository serves both standalone and coupled.
- Source-tree changes happen on JULES branches; the UM branch only
  records which JULES revision to pull.
- Tests (rose-stem) cover standalone configurations. UM-coupled testing
  happens in the UM rose-stem battery.

---

## The UM-to-JULES version map

The JULES documentation tree exposes the canonical mapping as
filesystem symlinks at the top of the user-guide repository:

| UM release | JULES code version |
|------------|--------------------|
| `um11.5` -> | `vn5.6` |
| `um11.6` -> | `vn5.7` |
| `um11.7` -> | `vn5.8` |
| `um11.8` -> | `vn5.9` |
| `um11.9` -> | `vn6.0` |
| `um12.0` -> | `vn6.1` |
| `um12.1` -> | `vn6.2` |
| `um12.2` -> | `vn6.3` |
| `um13.0` -> | `vn7.0` |
| `um13.1` -> | `vn7.1` |
| `um13.2` -> | `vn7.2` |
| `um13.3` -> | `vn7.3` |
| `um13.4` -> | `vn7.4` |

This list is drawn from the user-guide tree at the time of the v7.4
snapshot. Newer pairings (UM 13.5 onwards, JULES vn7.5 onwards) follow
the same one-to-one pattern. Confirm the current pairing on the JULES
MOSRS Trac wiki before starting a new UM build.

There is no skip: every UM release ships with exactly one named JULES
version, and that JULES version is the only one supported by that UM
build.

---

## How the UM build picks up JULES

The UM is itself an FCM project on MOSRS. Its build configuration
contains an `fcm-keyword` for JULES (typically `jules.x` resolving to
the JULES MOSRS repository), and the UM `fcm-make` configuration pins
the JULES revision via either:

- An explicit revision number (e.g. `JULES_SOURCE = fcm:jules.x_tr@12345`).
- A version tag (e.g. `JULES_SOURCE = fcm:jules.x_tr@vn7.4`).
- A branch URL when developing UM features that need JULES changes
  (`JULES_SOURCE = fcm:jules.x_br/dev/<owner>/<branch>`).

The UM `fcm-make` then extracts the JULES source under the UM build tree
and compiles it together with the UM atmospheric and ocean code into a
single `um.exe` (or the equivalent regional / climate-model
executable).

For coupled-mode development, set `JULES_SOURCE` to your branch and
ensure the UM-side `JULES_REV` (or the equivalent variable in your
suite's `fcm_make` app) points at the latest revision of that branch.

---

## Standalone vs coupled code paths

A small fraction of the JULES source is conditional on whether JULES
was built as part of the UM:

- The **river routing** scheme `i_river_vn=1` (UM-coupled TRIP) is only
  available inside the UM. Standalone runs must use
  `i_river_vn=2` (RFM) or `i_river_vn=3` (standalone TRIP).
- The **graupel options** in `jules_snow.nml` (`graupel_options=1` or
  `2`) only have an effect when the UM is supplying separate snow and
  graupel forcings. Standalone JULES has no separate graupel input, so
  only `graupel_options=0` is meaningful.
- The **`oasis_rivers.nml`** namelist drives a coupling layer to the
  ocean component for river-ocean exchange. It is only relevant in
  coupled UM-NEMO configurations.
- The **dust** scheme produces emission fluxes in standalone mode but
  has no atmospheric transport; in the UM it feeds the UM's interactive
  aerosol scheme.
- The **chemistry deposition** namelist (`jules_deposition.nml`) drives
  the UKCA chemistry component in coupled mode; in standalone mode it
  produces deposition diagnostics only.
- The **CABLE coupled mode** (`cable_*.nml`) is supported in both, but
  it is rarely used in production UM configurations.

Inside the source, conditional compilation keys (e.g. `#if defined(UM_JULES)`)
gate the coupled-only paths. The build system sets these keys based on
who is building. JULES authors must keep both code paths working; the
rose-stem battery and the UM-side tests check this on every commit.

---

## Configuring fcm-keyword for UM coupling

Both the UM and JULES MOSRS Trac wikis publish snippets to drop into
`~/.metomi/fcm/keyword.cfg`. A typical excerpt:

```
location{primary, type:svn}[jules.x]            = https://code.metoffice.gov.uk/svn/jules/main
browser.loc-tmpl[jules.x]                       = https://code.metoffice.gov.uk/trac/jules/browser/{1}{2}

location{primary, type:svn}[um.x]               = https://code.metoffice.gov.uk/svn/um/main
browser.loc-tmpl[um.x]                          = https://code.metoffice.gov.uk/trac/um/browser/{1}{2}
```

Probe with:

```bash
fcm kp jules.x
fcm kp um.x
```

For a UM-coupled JULES branch, your developer flow is:

1. Open a JULES MOSRS Trac ticket. Branch off the JULES trunk:
   `fcm bc -k <ticket> -t dev <branch_name> fcm:jules.x_tr@<revision>`.
2. Open a companion UM MOSRS Trac ticket. Branch off the UM trunk.
3. In the UM branch, point `JULES_SOURCE` at your JULES branch revision.
4. Run UM rose-stem against the JULES branch.
5. Run JULES rose-stem on the JULES branch separately.
6. Iterate until both pass; have both branches code-reviewed.
7. Commit JULES branch first, then update the UM branch's `JULES_REV`
   to the trunk revision and commit.

This order matters: the UM trunk should never reference a JULES branch
or a JULES revision that has not yet landed on the JULES trunk.

---

## Coupled testing: UM rose-stem

The UM rose-stem battery includes JULES-coupled tests (`atmos_only_*`,
`ukca_*`, `coupled_*` and similar groups). When you change JULES code
that affects coupled mode, you must run the relevant subset of UM
rose-stem on at least one supported platform (typically `meto-xc40-cce`
on MONSooN). Module leaders for the affected JULES module and the
relevant UM module must be Cc'd on both tickets.

---

## Bit comparison and the "PE comparison" requirement

The JULES upgrade procedure (Best 2010) requires that every JULES code
change passes a series of UM-coupled tests including bit comparison
across PE configurations. That is, the UM-coupled JULES must produce
bit-identical output for the same model state on different MPI task
counts and on different platforms. This is checked by the JULES
coordinator before a release is frozen.

In practice this means new JULES code must:

1. Be deterministic (no uninitialised variables, no thread-race in
   OpenMP regions).
2. Avoid relying on operation order that varies with decomposition
   (e.g. summing reductions in domain order rather than a fixed canonical
   order).
3. Avoid implicit IEEE-non-conformant optimisations (`-ffast-math` and
   friends are off in production builds for this reason).

When porting code from a non-coupled scheme into JULES, the most common
break is parallel non-determinism. Run with two different task counts
and `cdo diff` the dumps as a smoke test.

---

## Where to next

- Set up your standalone JULES checkout: `getting-started.md`
- Build the standalone executable: `running-jules.md`
- Submit a JULES branch through the MOSRS workflow: `contributing.md`
- Diagnose coupled-mode build or run failures: `debugging.md`
