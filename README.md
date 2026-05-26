# JULES Skill

A progressive-disclosure knowledge package for the
[Joint UK Land Environment Simulator (JULES)](https://jules-lsm.github.io/),
the community Land Surface Model maintained by the Met Office and used both
standalone and as the land surface of the Met Office Unified Model (UM).

> **Maintainer of JULES:** Met Office (jules-support@metoffice.gov.uk)
> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0
> **Skill licence:** MIT

> ⚠️ **Disclaimer — please read before using this skill.**
> This skill is **not a gold-standard reference**. It is a helper that lowers
> the barrier for new users to **get their hands dirty** with the model. AI
> agents (and the humans drafting this material) make mistakes; commands, file
> paths, namelist options, and physics explanations here can be wrong,
> incomplete, or out of date. **Always cross-check with the official model
> documentation, the source code, and a human expert before trusting any
> output for research, publication, or operational use.**

## What This Is

A self-contained skill that teaches AI agents (and humans) how to
**request access to, check out, build, configure, run, debug, and
contribute to** JULES. It targets the standalone offline configuration
driven by Fortran namelists, the Rose / Cylc workflow used by the Met
Office and JASMIN community, and the UM-coupled mode.

Procedural knowledge that is normally only transmitted by sitting next to
an experienced JULES developer is captured here: what FCM keywords to set
up, why every namelist file must exist for every run, the order in which
upgrade macros must be applied, and the version-mapping table between UM
releases and JULES code versions.

**Progressive disclosure:**
- `SKILL.md`: routing hub with decision tree, repo layout, quick start,
  critical rules
- `reference/*.md`: deep-dive docs loaded on demand

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point: decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | MOSRS access, FCM keyword setup, prerequisites, first checkout, first build |
| `reference/architecture.md` | `src/` layout, surface tile model, soil column, snow, hydrology, river routing, dust, MORUSES, ECOSSE |
| `reference/running-jules.md` | FCM make, namelist directory rules, Rose suites, single-site and gridded runs, MPI and OpenMP |
| `reference/physics-options.md` | TRIFFID, multi-layer snow, RFM rivers, MORUSES urban, ECOSSE soils, key switches |
| `reference/tutorial-walkthrough.md` | Loobos case, JULES-Rose User and Developer tutorials, JASMIN / VM / MONSooN setup |
| `reference/output-files.md` | Output profiles, namelist controls, dump files, adding a new output variable |
| `reference/coupling-um.md` | UM 11.5 to 13.4 to JULES vn5.6 to vn7.4 map, fcm-keyword config, coupled-versus-standalone code split |
| `reference/debugging.md` | FCM build errors, namelist mismatches, restart drift, balance violations |
| `reference/contributing.md` | MOSRS Trac ticket workflow, branch and rose-stem, module leader review |

## Sources

This skill is grounded in:

1. **The JULES user guide** (vn7.4 chapter set, latest release vn7.9 at the
   time of writing): https://jules-lsm.github.io/
2. **The JULES tutorial portal** (FCM fundamentals and JULES-Rose user /
   developer tutorials): https://jules-lsm.github.io/tutorial/bg_info/
3. **JULES coding standards**:
   https://jules-lsm.github.io/coding_standards/
4. **JULES technical documentation** (Best et al. 2011 and Clark et al.
   2011 GMD model description papers)
5. **JULES upgrade procedures** (Met Office internal procedure, Best 2010)
6. The local clone of the user-guide tree under
   `ESM-bench/data/repos/jules/`, which is a snapshot of the published
   user guide for every JULES release from vn3.3 through vn7.4 plus the
   UM-coupled mappings `um11.5` through `um13.4`

The MOSRS Trac, ticket queue, and branch list itself are auth-walled and
not directly readable. This skill links to them and explains the access
flow rather than reproducing private content.

## Install

This skill follows the same layout as
[laps-skill](https://github.com/huangzesen/laps-skill) and the noahmp
sibling skill at https://github.com/ktwu01/noahmp-skill:

```
jules-skill/
|-- SKILL.md              <- routing hub (read first)
|-- README.md             <- this file
|-- LICENSE
|-- .gitignore
\-- reference/            <- deep-dive docs
```

To use with a Claude Code or LingTai agent, drop the directory into your
skills library and refresh.

## Licence

This skill (the documentation in this repository) is released under the
MIT licence. See `LICENSE`.

JULES itself is governed by the JULES Terms and Conditions for legacy
releases and the BSD 3-Clause licence for vn7.x:
https://jules-lsm.github.io/access_req/JULES_Licence.pdf
