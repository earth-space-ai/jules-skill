# Contributing to JULES

JULES is a community model maintained by the Met Office. New science,
bug fixes, and documentation improvements all go through the Met Office
Science Repository Service (MOSRS) using the FCM workflow with Trac
tickets, branches, and a multi-stage review chain. This is more involved
than a GitHub pull-request flow but enforces the bit-comparison and
backwards-compatibility guarantees the UM coupling requires.

This page describes the canonical contribution flow. It is grounded in
the Met Office "Procedures for implementing developments into a new
version of JULES" document (Best 2010), the JULES-Rose Developer
tutorial, and the JULES coding standards.

---

## The 12-step procedure

The Best 2010 document defines twelve sequential steps from idea to
release:

1. **Notify the module leader(s)** at the earliest possible opportunity.
   Module leaders maintain a register of in-flight development for the
   science steering committee.
2. **Module leader maintains a development register** for the science
   steering committee meetings.
3. **Notify the module leader of an intended completion date** when
   nearing completion.
4. **Run the benchmarking system.** The development must
   (a) be backwards compatible (gated on a switch that defaults to the
   prior behaviour) and (b) not produce an unacceptable performance
   regression in any module.
5. **Update technical and user documentation** to cover the new
   development.
6. **Hand the code and documentation to the module leader** for code
   review. The review checks coding standards, benchmarking, and
   documentation.
7. **Module leader presents the development to the management
   committee** for approval to include in a future release.
8. **Management committee schedules releases** and decides which
   developments go into which release.
9. **Pass completed code and documentation to the JULES coordinator**
   who consolidates all changes into a single version.
10. **Coordinator runs the consolidated code through benchmarking**
    again: backwards compatibility plus performance.
11. **Coordinator runs UM-coupled tests** including bit comparison
    across PE configurations.
12. **Coordinator freezes the code and releases** the new version.

### Exceptions

I. **Control-code-only developments**: step 10 only checks backwards
   compatibility (no science perf check); step 11 (UM tests) is
   skipped because control code does not enter the UM build.

II. **Large, multi-stage developments needing an interim consolidation**:
    steps 4 and 10 only check backwards compatibility; step 6 only
    checks coding standards, documentation, and backwards compatibility
    of the benchmarking tests. Interim upgrades are uncommon and
    require management-committee approval routed through the module
    leader as in step 7.

---

## What this means in day-to-day terms

### Step 0: Decide your scope

Read the JULES MOSRS Trac wiki to find the **module leader** for the
science area you will touch. The current list (current at the v7.4 user
guide snapshot) is reproduced in `tutorial-walkthrough.md`. Email the
relevant leader(s) before you write any code.

### Step 1: Open a Trac ticket

On the MOSRS JULES Trac (`https://code.metoffice.gov.uk/trac/jules`):

1. **New Ticket**.
2. Title: short imperative (e.g. "Add MORUSES storage heat-flux
   diagnostics").
3. Description: motivation, scope, the new switch name and default,
   any new namelist members, the scientific reference.
4. Cc the module leader(s) for every affected module.
5. Set the milestone if you know which release it targets.

### Step 2: Branch off the trunk

Use FCM to open a branch under the Trac ticket. The branch name
convention is `<owner>/<short_description>` and the branch type is `dev`:

```bash
fcm branch-create --type dev --name <short_description> --ticket <ticket_number> fcm:jules.x_tr
```

Or shorter:

```bash
fcm bc -k <ticket> -t dev <branch_name> fcm:jules.x_tr
```

This creates `fcm:jules.x_br/dev/<owner>/<branch_name>` and writes a
ticket comment.

### Step 3: Check out the branch

```bash
mkdir -p ~/jules/branches/<branch_name>
cd ~/jules/branches/<branch_name>
fcm checkout fcm:jules.x_br/dev/<owner>/<branch_name> .
```

### Step 4: Make changes

Code edits live under `src/`. Honour the JULES coding standards
(https://jules-lsm.github.io/coding_standards/):

- All routines and documentation use SI units (with standard SI prefixes
  permitted).
- New science is gated on a new namelist switch defaulting to the prior
  behaviour. The contribution must not change existing-config output
  bit-for-bit.
- Add the new namelist member to the relevant `*.nml` declaration in
  `src/initialisation/`, document it in the user-guide docstring above
  the variable, and write a Rose meta entry in `rose-meta/`.
- Write an upgrade macro in `rose-meta/jules-standalone/versions.py` (or
  the equivalent location for your version) so that existing suites can
  auto-upgrade.

### Step 5: Run rose-stem locally

```bash
cd ~/jules/branches/<branch_name>
rose stem --group=tutorial --new --name=<branch>_quick    # smoke test first
rose stem --group=all --new --name=<branch>_full          # then full
```

Investigate every `nccmp` failure. Acceptable reasons:

1. **New configuration**: no prior KGO existed.
2. **Bug fix with evidence**: plots and numbers showing improvement,
   plus module leader approval recorded on the Trac ticket. The module
   leader must be Cc'd.
3. **New science with peer-reviewed reference**: cite the paper, attach
   plots/figures.

For each KGO change you accept, update the KGO file under
`/jules/rose-stem-kgo/vn<X>.<Y>/` (Met Office) or the equivalent JASMIN
or MONSooN location, and record the change on the ticket.

### Step 6: Commit to the branch

```bash
fcm status                            # review your changes
fcm diff                              # eyeball the diff
fcm commit -m "Add foo. Refs #<ticket>."
```

The Trac ticket comment system picks up `Refs #NNN` and `Closes #NNN`
keywords.

### Step 7: Request review

Update the Trac ticket:
- Set status to `Code Review`.
- Cc the module leader(s).
- Attach the `nccmp` summary and any KGO-change justifications.
- Link the rose-stem run output (the `cylc-run/<suite>/trac.log` is the
  evidence the reviewer wants).

The module leader either reviews the code themselves or delegates to
another scientist familiar with the area. The review checks:

- Coding standards adherence
- Benchmarking pass (rose-stem all-green or justified KGO)
- Backwards compatibility (existing configs bit-identical)
- Documentation (user guide and rose-meta)
- Scientific correctness (against the cited reference)

Iterate on review comments by pushing additional commits to the same
branch.

### Step 8: Management committee approval

After code review passes, the module leader presents to the management
committee. The management committee decides which release the change
lands in.

### Step 9: Hand off to the JULES coordinator

The coordinator pulls the approved branch into the consolidation
branch for the next release, runs the consolidated code through
benchmarking and UM-coupled tests, then freezes and tags the release.

---

## The GitHub mirror plan

In late 2025 the Met Office announced the first GitHub mirror release of
the Simulation Systems repository (which includes JULES). At the time of
this skill's writing, the GitHub repository is private. When it goes
public, GitHub accounts will be required for new development. The MOSRS
Trac will remain the system of record for tickets and the canonical
build configuration; the GitHub mirror is for visibility and
contribution-discovery, not as a replacement for MOSRS.

Practical implication: do not invest time in a GitHub-only contribution
flow until the Met Office documents the bridge between GitHub PRs and
MOSRS Trac tickets. Today, every change still needs a MOSRS ticket and
a Subversion / FCM branch on MOSRS.

---

## Documentation

Two documentation surfaces must be updated for a science contribution:

1. **The user guide** (`user_guide/doc/`). Sphinx + reStructuredText.
   Local build:
   ```bash
   conda env create -f environment.yml
   conda activate jules-user-guide
   cd user_guide/doc
   make html
   firefox build/html/index.html
   ```

2. **The Rose meta** (`rose-meta/jules-standalone/HEAD/rose-meta.conf`).
   Every namelist member needs:
   - `description` (multi-line OK)
   - `help` (the long form shown in Rose Edit)
   - `type` (integer, real, logical, character, etc.)
   - `length` (for arrays)
   - `range` or `values` (validators)
   - `compulsory` if the user must set it
   - `trigger` rules to grey out members when their controlling switch
     is off

The Rose Edit GUI shows the description and help text from rose-meta.
This is the surface most users see; treat it as primary documentation.

---

## Upgrade macros

For every JULES version that introduces, removes, or renames a namelist
member, an upgrade macro must be added so that
`rose app-upgrade -M $JULES_ROOT/rose-meta -C app/jules <newversion>`
can transform an old suite into a new-version suite without manual
editing.

The macros live under `rose-meta/jules-standalone/versions.py` (and the
equivalent for the build app). Each macro is a Python class deriving
from `rose.upgrade.MacroUpgrade` and implementing `upgrade(self, config,
meta_config=None)`.

A typical macro adds a new variable with a default value:

```python
class vn720_t1234(rose.upgrade.MacroUpgrade):
    """ticket #1234: add l_my_new_switch (default F)."""
    BEFORE_TAG = "vn7.1"
    AFTER_TAG = "vn7.2_t1234"
    def upgrade(self, config, meta_config=None):
        self.add_setting(config, ["namelist:jules_vegetation", "l_my_new_switch"], ".false.")
        return config, self.reports
```

The user-guide section "Known limitations" (`code/known-limitations.html`)
documents one historical case (`JULES_VEGETATION_PROPS` namelist added at
vn5.7 but the upgrade macro not added until vn6.1) where this discipline
slipped. Do not add to that list.

---

## Common contribution pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| KGO regression on every test | New science is on by default | Gate on a switch defaulting to prior behaviour |
| KGO regression only on Intel platform | Compiler-dependent rounding or uninitialised variable | Initialise all locals; check with `-check uninit` (Intel) or `-finit-real=snan` (gfortran) |
| Module leader rejects review for "no upgrade macro" | New namelist member not paired with rose-meta and macro | Add to `rose-meta/jules-standalone/HEAD/rose-meta.conf` and write the upgrade macro |
| Trac ticket comment system does not link to commits | Forgot `Refs #<ticket>` in commit message | Use `fcm commit -m "Description. Refs #<ticket>."` |
| UM rose-stem fails after JULES branch merge | Coupled-mode code path broken | Run UM rose-stem against your JULES branch *before* asking for code review |
| `fcm bc` fails with "ticket does not exist" | Trac ticket not created or wrong number | Open the ticket first, copy the number, then branch |

---

## Where to next

- Set up your MOSRS access and FCM keywords: `getting-started.md`
- Use rose-stem to test your changes: `running-jules.md`
- Check that your output variable lands correctly: `output-files.md`
- Verify the UM-coupled side still builds: `coupling-um.md`
- Diagnose why a test fails: `debugging.md`
