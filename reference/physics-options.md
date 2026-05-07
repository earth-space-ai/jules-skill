# JULES Physics Options

JULES has a rich switch landscape: any individual run is a particular
combination of options across vegetation, soil biogeochemistry, snow,
hydrology, river routing, and urban schemes. This page documents the
high-leverage switches and the standard combinations.

The authoritative per-namelist documentation is on the JULES user guide
under `https://jules-lsm.github.io/vn7.4/namelists/`. Always verify
defaults and option numbers against the version you are actually running:
they drift between releases.

---

## Vegetation: phenology, TRIFFID, crops

`jules_vegetation.nml::JULES_VEGETATION`

| Switch | Default | Effect |
|--------|---------|--------|
| `l_phenol` | F | Vegetation phenology model. Uses leaf turnover rate to give a time-varying LAI. |
| `l_triffid` | F | TRIFFID dynamic vegetation. Soil C is modelled with the four-pool scheme (biomass, humus, decomposable plant material, resistant plant material). |
| `l_veg_compete` | T | TRIFFID inter-PFT competition modifies the surface tile fractions. Only used if `l_triffid=T`. |
| `l_ht_compete` | F | Height-based competition (recommended). Allows a generic number of PFTs. |
| `l_nitrogen` | F | Nitrogen limitation on vegetation growth. Only used if `l_triffid=T`. |
| `l_trif_crop` | F | Agricultural PFT competition with explicit harvest carbon (when crop_io > 0 reserves a fraction of the gridbox for agricultural PFTs). |
| `l_trait_phys` | F | Trait-based physiology: Vcmax computed from leaf nitrogen (nmass) and leaf mass per area (LMA), as in Kattge et al. 2009. Two extra parameters `vint`, `vsl`. When F, Vcmax uses `nl0` and `neff`. |
| `ncpft` | 0 | Number of crop PFTs. Setting >0 turns on the crop model. |
| `lai_io` | per PFT | Constant LAI fallback if neither phenology nor crops are used. Time-varying LAI via `JULES_PRESCRIBED`. |

Standard recipes:

- **Prescribed LAI, no carbon dynamics**: `l_phenol=F`, `l_triffid=F`,
  prescribed `lai_io`. Standard for forcing-driven historical reanalyses
  where you do not want feedbacks.
- **Phenology only**: `l_phenol=T`, `l_triffid=F`. LAI evolves but
  fractional cover is fixed.
- **Full TRIFFID with N**: `l_phenol=T`, `l_triffid=T`,
  `l_veg_compete=T`, `l_ht_compete=T`, `l_nitrogen=T`,
  `soil_bgc_model=2`. Standard CMIP-class config.
- **TRIFFID with crops**: same as above plus `l_trif_crop=T` and a
  non-zero `crop_io`. Crop and TRIFFID **cannot** both be active in the
  full crop-model sense; the crop fraction inside TRIFFID is the
  workaround.

---

## Soil biogeochemistry: 4-pool vs ECOSSE

`jules_soil_biogeochem.nml`

| `soil_bgc_model` | Meaning |
|------------------|---------|
| 1 | Prescribed soil C pools. Required if TRIFFID is off. |
| 2 | Four-pool soil C model (RothC-style). Requires TRIFFID. |
| 3 | ECOSSE: layered soil C and N. Configured via `jules_soil_ecosse.nml`. |

`l_layeredc=T` with `soil_bgc_model=2` activates a layered version of the
4-pool model.

ECOSSE adds explicit nitrogen mineralisation, immobilisation, leaching,
and N2O emission. It is the recommended scheme for soil-process work but
has more parameters and higher cost.

---

## Snow: composite vs multi-layer

`jules_snow.nml::JULES_SNOW`

| Switch | Default | Effect |
|--------|---------|--------|
| `nsmax` | 0 | 0 = composite soil-snow layer (JULES2.0 behaviour). >0 = explicit multi-layer scheme with up to `nsmax` layers. 3 or more is recommended. |
| `dzsnow` | per layer | Maximum thickness per snow layer (real array of length `nsmax`). |
| `l_snowdep_surf` | F | Use equivalent canopy snow depth in surface calculations on tiles with a snow canopy (improves canopy-buried-by-snow physics). |
| `frac_snow_subl_melt` | 0 | Use snow-cover fraction in sublimation and melt calculations (0=off, 1=on). |
| `graupel_options` | 0 | 0=include graupel as snowfall, 1=ignore graupel, 2=treat separately. Only 0 is meaningful in standalone JULES (no separate graupel forcing); the others exist for the UM-coupled case. |

The multi-layer scheme tracks per-layer density, grain size, layer
thickness, and temperature. Compaction and densification are explicit.
The composite scheme is faster and simpler but cannot represent layered
snow processes (refrozen rain, depth hoar, percolation).

References embedded in `JULES_SNOW`:
- HCTN30: Hadley Centre technical note 30 (Met Office Library).
- Best et al. 2011 GMD energy and water paper.

---

## Hydrology and runoff

`jules_hydrology.nml::JULES_HYDROLOGY`

| Switch | Effect |
|--------|--------|
| `l_top` | TOPMODEL-based subsurface runoff with explicit water table |
| `l_pdm` | PDM (Probability Distributed Model) saturation-excess runoff |
| `nfita` | TOPMODEL fitting iterations |

`l_top` and `l_pdm` are alternative parameterisations of saturation excess
runoff and are not normally used together. `l_pdm` is faster and uses
fewer ancillaries; `l_top` is more physically grounded but requires a
topographic index field.

---

## River routing: TRIP and RFM

`jules_rivers.nml::JULES_RIVERS`

| Switch / value | Effect |
|----------------|--------|
| `l_rivers` | Master switch |
| `i_river_vn=1` | UM-coupled TRIP (not allowed in standalone JULES) |
| `i_river_vn=2` | RFM kinematic wave (Dadson & Bell 2010, Bell et al. 2007). Standalone-capable. |
| `i_river_vn=3` | Standalone TRIP (Oki et al. 1999). Standalone-capable. |
| `nstep_rivers` | Number of model timesteps per routing timestep |
| `l_riv_overbank` | Compute overbank inundation fraction |
| `a_thresh` (RFM) | Catchment-area threshold for routing |

The `JULES_OVERBANK` namelist becomes required when `l_riv_overbank=T`.

The user guide warns that the river routing code is still in active
development. Always sanity-check discharge against observations or a
reference simulation before using results downstream.

---

## Urban: MORUSES two-tile

`urban.nml::JULES_URBAN`

| Switch | Effect |
|--------|--------|
| `l_urban2t` | Master switch. Requires both `urban_canyon` and `urban_roof` surface types in `jules_surface_types.nml`. |
| `l_moruses_albedo` | MORUSES snow-free canyon albedo |
| `l_moruses_emissivity` | MORUSES canyon emissivity |
| `l_moruses_rough` | MORUSES heat roughness length |
| `l_moruses_macdonald` | MORUSES momentum roughness length (Macdonald 1998) |
| `l_moruses_storage` | MORUSES thermal inertia for canyon and roof |

Anything MORUSES does not override falls through to the values in
`nveg_params.nml`. Canyon geometry (height-to-width ratio, building plan
area fraction) is read from `URBAN_PROPERTIES` ancillary data.

---

## Radiation: two-stream canopy

`jules_radiation.nml::JULES_RADIATION` selects the canopy radiative
transfer model. The standard is a two-stream scheme with PFT-specific
leaf optical properties. Important switches include those for the
photo-acclimation model (`photo_acclim_model`) and for the trait-based
photosynthesis variant (`l_albedo_obs`, `l_dolr_land_black`, etc.).

---

## Surface flux scheme

`jules_surface.nml::JULES_SURFACE` controls the surface energy balance
scheme: aerodynamic resistance, surface drag, dust, and the choice of
photosynthesis model (Collatz vs Farquhar style). Key switches include
`i_aggregate_opt`, `cor_mo_iter`, `iscrntdiag`, and the various stomatal
conductance options driven through `pft_params.nml`.

---

## Soil: hydraulic and thermal

`jules_soil.nml::JULES_SOIL`

| Switch / variable | Effect |
|-------------------|--------|
| `sm_levels` | Number of soil layers |
| `l_vg_soil` | Use van Genuchten soil hydraulics (default is Brooks-Corey via Cosby) |
| `l_dpsids_dsdz` | Account for soil-water-potential gradients in heat conduction |
| `dzsoil_io` | Per-layer thicknesses (m), array of length `sm_levels` |
| `confrac` | Convective rainfall fraction for canopy interception |

Permafrost work commonly uses 8 to 16 soil layers with thicker deep
layers (1 m or more) to span the active layer and the permafrost top.
The default 4-layer column is too coarse for cryosphere science.

---

## Atmospheric deposition

`jules_deposition.nml` enables interactive dry deposition for a
configurable list of trace gases (`JULES_DEPOSITION_SPECIES`). This is
mostly used in coupled chemistry runs.

---

## Fire (INFERNO)

`fire.nml` enables the INFERNO interactive fire model, which couples
biomass burning emissions to vegetation cover, soil moisture, and
lightning frequency.

---

## IMOGEN

`imogen.nml` enables the IMOGEN climate emulator: pattern-scaled climate
forcings driven by global-mean temperature trajectories from one of a
library of CMIP-class GCMs. Used for low-cost scenario sweeps without
running the full UM.

---

## CABLE coupled mode

`cable_*.nml` switches JULES into a mode where the CABLE land surface
model is run inside the JULES driver framework (used for inter-model
comparison). This is an alternative physics path, not an addition.

---

## Science fixes

`science_fixes.nml::JULES_TEMP_FIXES` is a holding pen for temporary
flags that turn on bug fixes in a backwards-compatible way. New fixes
must be added off-by-default and migrated to default-on in a later
release. Reading this namelist for the version you run is a good way to
discover known bugs and their fixes.

---

## Putting it together: a "GL" (UK Earth System Model) style run

A representative configuration used in CMIP-class JULES runs has:

```
l_phenol           = T
l_triffid          = T
l_veg_compete      = T
l_ht_compete       = T
l_nitrogen         = T
l_trif_crop        = T
soil_bgc_model     = 2
l_layeredc         = T
nsmax              = 3
l_snowdep_surf     = T
l_top              = T
l_pdm              = F
l_rivers           = T
i_river_vn         = 2
sm_levels          = 4
l_urban2t          = F
```

Plus the matching `pft_params.nml`, `triffid_params.nml`,
`nveg_params.nml`, and ancillary files. The Met Office "GL9" series of
configurations and the corresponding rose-stem apps are the primary
worked examples to copy from.

---

## Where to next

- Run a standard configuration: `running-jules.md`
- Walk through the tutorial cases: `tutorial-walkthrough.md`
- Add or remove output variables: `output-files.md`
- Diagnose physics misbehaviour: `debugging.md`
