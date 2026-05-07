# JULES Architecture

This page is the map you need before you edit anything in `src/`. JULES is
a tile-based land surface model that solves the surface energy and water
balance per surface tile, then aggregates to the gridbox. Soil processes
sit underneath either a single shared soil column per gridbox or one soil
column per surface tile. River routing, dust, and the urban two-tile
scheme are bolted onto the gridbox-scale flow as optional sub-models.

---

## The conceptual model

For both gridded and single-point runs, JULES treats each gridbox as a
collection of surface types:

| Index | Surface type | Notes |
|-------|--------------|-------|
| 1 | Broadleaf trees | PFT |
| 2 | Needleleaf trees | PFT |
| 3 | C3 (temperate) grass | PFT |
| 4 | C4 (tropical) grass | PFT |
| 5 | Shrubs | PFT |
| 6 | Urban | non-vegetated |
| 7 | Inland water (lakes) | non-vegetated |
| 8 | Bare soil | non-vegetated |
| 9 | Land ice | non-vegetated; whole gridbox if used |

These nine types are the standard configuration. Since vn2.0 the user can
extend the surface type list (more PFTs or more non-vegetated tiles)
provided TRIFFID is not active. With TRIFFID, the natural PFTs are fixed
and the fractional cover evolves under competition.

A separate energy balance is computed for each surface tile. The gridbox
mean is found by area-weighting tile values by `frac_io` (the fractional
area of each tile). Land ice gridboxes are treated as 100 percent ice; all
other gridboxes are any mixture of types 1 through 8.

Soil processes are modelled in several layers (default 4 layers, often
extended to many more for permafrost work). Each surface tile can sit on
its own soil column (`l_tile_soil`) or all tiles share a single column.

---

## Top-level source layout

```
jules/
|-- src/
|   |-- control/                <- main, time loop, top-level driver
|   |-- science/
|   |   |-- surface/            <- per-tile energy balance
|   |   |   |-- sf_diag, sf_evap, sf_flux, sf_resist, sf_stom
|   |   |   |-- canopyht, sf_aero, blend
|   |   |-- soil/               <- soil hydraulics, infiltration, Richards eq.
|   |   |   |-- soilhyd, soilt, hydrol_jls, infiltration
|   |   |-- snow/               <- multi-layer or composite snow
|   |   |   |-- snow_intctl, snowpack, snowtherm, layersnow, compactn
|   |   |-- vegetation/         <- phenology, TRIFFID
|   |   |   |-- veg2, sparm, leaf_lit, dpm_rpm, triffid
|   |   |-- radiation/          <- two-stream canopy radiation
|   |   |   |-- ftsa, albpft, sice_albedo, two_stream
|   |   |-- river_routing/      <- TRIP and RFM
|   |   |-- urban/              <- MORUSES two-tile urban
|   |   |-- biogeochem/         <- ECOSSE, four-pool soil C and N
|   |   |-- hydrology/          <- TOPMODEL, PDM runoff
|   |   |-- dust/               <- mineral dust emission scheme
|   |   |-- crops/              <- crop model
|   |   |-- fire/               <- INFERNO fire scheme
|   |   |-- imogen/             <- pattern-scaling climate emulator
|   |   \-- diagnostics/
|   |-- io/
|   |   |-- file_mod/           <- common file API
|   |   |-- file_ascii/         <- ASCII driver
|   |   |-- file_ncdf/          <- NetCDF driver
|   |   |-- file_gridded/       <- gridded cube layer
|   |   |-- file_ts/            <- time-series file layer
|   |   |-- input/              <- read namelists, populate state
|   |   |-- output/             <- write profiles
|   |   \-- model_interface/    <- the populate_var / extract_var bridge
|   |-- initialisation/         <- read namelists, allocate state
|   |-- params/                 <- PFT and soil parameter modules
|   \-- util/                   <- maths and bookkeeping helpers
|-- etc/fcm-make/               <- FCM build configuration
|-- rose-stem/                  <- regression test suite
|-- rose-meta/                  <- Rose Edit metadata + upgrade macros
|-- examples/                   <- shipped namelist examples
\-- utils/
```

---

## The science modules and the namelists that drive them

| Science block | Namelist that turns it on or shapes it |
|---------------|----------------------------------------|
| Surface energy balance and aerodynamic resistance | `jules_surface.nml`, `jules_surface_types.nml` |
| Canopy radiation | `jules_radiation.nml` |
| Soil hydraulics and Richards' equation | `jules_soil.nml`, `jules_hydrology.nml` |
| Soil thermal | `jules_soil.nml` |
| Multi-layer snow | `jules_snow.nml` (`nsmax`) |
| Vegetation phenology | `jules_vegetation.nml` (`l_phenol`) |
| TRIFFID dynamic vegetation | `jules_vegetation.nml` (`l_triffid`, `l_veg_compete`, `l_nitrogen`, `l_ht_compete`) |
| Crop model | `jules_vegetation.nml` (set `ncpft > 0`), `crop_params.nml` |
| Soil biogeochemistry (4-pool or ECOSSE) | `jules_soil_biogeochem.nml`, `jules_soil_ecosse.nml` |
| Atmospheric deposition | `jules_deposition.nml` |
| River routing (RFM, TRIP) | `jules_rivers.nml` (`l_rivers`, `i_river_vn`, `l_riv_overbank`) |
| Water resources and irrigation | `jules_water_resources.nml`, `jules_irrig.nml` |
| Urban two-tile scheme (MORUSES) | `urban.nml` (`l_urban2t`, `l_moruses_*`) |
| Initial conditions | `initial_conditions.nml` |
| Time stepping and spin-up | `timesteps.nml` (`JULES_TIME`, `JULES_SPINUP`) |
| Model grid | `model_grid.nml` (`JULES_INPUT_GRID`, `JULES_LATLON`, `JULES_LAND_FRAC`, `JULES_MODEL_GRID`) |
| Driving data (atmospheric forcing) | `drive.nml` |
| PFT and non-veg parameters | `pft_params.nml`, `nveg_params.nml`, `triffid_params.nml` |
| Output profiles | `output.nml` |
| CABLE coupled mode (alternate physics) | `cable_*.nml` |
| Print control and diagnostics | `jules_prnt_control.nml` |
| Science fixes (interim bug-fix flags) | `science_fixes.nml` |
| Fire (INFERNO) | `fire.nml` |
| IMOGEN climate emulator | `imogen.nml` |
| Atmosphere coupling for rivers | `oasis_rivers.nml` |

JULES expects all 30+ namelist files to exist for every run, in the order
defined by `src/initialisation/`. Files for science you do not use can be
empty, but the file itself must be present.

---

## Soil and snow

**Soil layers.** The number of soil layers is set by `sm_levels` in
`jules_soil.nml`. The default is 4 layers; permafrost work commonly uses 8
or more. Each layer has its own thickness, hydraulic conductivity, and
thermal capacity. Soil texture is read either as Cosby parameters (sand,
silt, clay fractions) or via the pedotransfer scheme.

**Soil hydraulics.** Richards' equation is solved implicitly. Surface
infiltration, root uptake, and gravity drainage at the bottom are the
boundary conditions.

**Snow.** Two regimes:

- `nsmax = 0`: composite soil-snow layer (the JULES2.0 and earlier
  behaviour). Snow is represented by mass, depth, and a single bulk
  temperature.
- `nsmax > 0`: explicit multi-layer snow scheme (3 or more layers
  recommended). Snow density, grain size, layer thickness, and
  temperature are prognostic per layer. Compaction and densification are
  modelled. Snow on the canopy can be treated separately
  (`l_snowdep_surf`).

Graupel is normally bundled with snowfall (`graupel_options=0`) in
standalone JULES; the other settings exist for the UM-coupled case.

---

## Hydrology and runoff

| Switch | Effect |
|--------|--------|
| `l_top` | TOPMODEL-based subsurface runoff with explicit water table |
| `l_pdm` | PDM (Probability Distributed Model) saturation-excess runoff |
| `l_rivers` | Pass runoff to a river routing scheme |

These are not mutually exclusive but are not all valid in combination; see
`jules_hydrology.nml`.

---

## River routing

`jules_rivers.nml` selects one of three implementations via `i_river_vn`:

| Value | Scheme | Notes |
|-------|--------|-------|
| 1 | UM-coupled JULES TRIP | Not allowed in standalone JULES |
| 2 | RFM kinematic wave (Dadson and Bell 2010, Bell et al. 2007) | Standalone-capable |
| 3 | Standalone JULES TRIP (Oki et al. 1999) | Standalone-capable |

River routing introduces two extra grids: the river routing input grid
(2D, must be specified in `JULES_RIVERS_PROPS`) and the river routing
model grid (1D, internal compression to `np_rivers` valid points). Output
is on the routing grid by default, optionally regridded to the model grid.

Overbank inundation is enabled with `l_riv_overbank`, which computes
`frac_fplain_lp` (the fraction of each cell flooded). The `JULES_OVERBANK`
namelist becomes required when this is on.

The vn7.4 user guide explicitly cautions that the river routing code is
still in development; results should be checked.

---

## Dust

The mineral dust emission scheme runs over bare soil tiles (and
optionally low vegetation) and produces emission flux per dust bin. It is
controlled from `jules_surface.nml` and the `dust` namelist when JULES is
coupled to an atmospheric dust transport model; in standalone JULES the
emissions are diagnostic.

---

## MORUSES (urban two-tile scheme)

The MORUSES (Met Office Reading Urban Surface Exchange Scheme) splits the
urban surface into a `urban_canyon` tile (street + walls) and a
`urban_roof` tile, set up in `jules_surface_types.nml`. `urban.nml`
contains the MORUSES switches:

- `l_urban2t`: master switch, requires both canyon and roof tile types
- `l_moruses_albedo`: snow-free canyon albedo
- `l_moruses_emissivity`: canyon emissivity
- `l_moruses_rough`: heat roughness length
- `l_moruses_macdonald`: momentum roughness length (Macdonald 1998)
- `l_moruses_storage`: thermal inertia of canyon and roof

Ancillary geometry (canyon height-to-width ratio, building plan area
fraction) is read via `URBAN_PROPERTIES`. Anything MORUSES does not
override falls back to the values in `nveg_params.nml`.

---

## ECOSSE soil biogeochemistry

`jules_soil_ecosse.nml` enables the ECOSSE (Estimation of Carbon in
Organic Soils, Sequestration and Emissions) replacement for the simpler
4-pool RothC-style soil C+N model. Use with `soil_bgc_model = 3`
(layered, full ECOSSE). The 4-pool model is `soil_bgc_model = 2` and
requires TRIFFID; the prescribed-pools mode is `soil_bgc_model = 1`. A
layered version of the 4-pool C model is selected with
`l_layeredc = .TRUE.`.

---

## I/O framework (file_mod, model_interface_mod)

JULES vn3.1 introduced a layered I/O framework that is still in place in
vn7.x. The layers are:

1. **Common file handling API** (`src/io/file_mod`). Defines the abstract
   notions of file, dimension, variable, and the special `record`
   dimension (typically time). Driver implementations in `file_ascii`
   and `file_ncdf` provide ASCII and NetCDF backends respectively.
2. **Gridded file API** (`file_gridded_mod`). Imposes "cubes of gridded
   data" on top of the common API. Handles 1D vs 2D underlying grids
   and subgrid extraction (`file_gridded_read_var`,
   `file_gridded_write_var`).
3. **Time series file API** (`file_ts_mod`). Treats a sequence of files
   (monthly, yearly) as a single seekable stream.
4. **Input and output layers**. Talk to the model state through
   `model_interface_mod`, which exposes `populate_var` (for inputs) and
   `extract_var` (for outputs).

To add a new variable for output, the only edits required are in
`src/io/model_interface`. See `output-files.md`.

---

## Key code-level invariants

- All physical quantities use SI units. Where a routine uses non-SI for
  performance, it must be commented prominently.
- Variable kinds use the compiler default for each intrinsic type. There
  is no JULES-wide `kind_jules` parameter analogous to Noah-MP's
  `kind_noahmp`. Changing precision means changing the source.
- The longest possible run with the current variable types is around
  100 years on 32-bit machines. Longer runs must be split into chained
  sections each restarting from the previous dump.
- All routines and documentation use SI units. Standard SI prefixes are
  permitted.

---

## Where to next

- Compile and run a real simulation: `running-jules.md`
- Decide which physics switches to set: `physics-options.md`
- Add a new output variable: `output-files.md`
- Browse the contributor workflow: `contributing.md`
