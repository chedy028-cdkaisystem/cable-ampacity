# Cable-ampacity tools: field cheat sheet

Derived from the server's live `tools/list`. Field names are exact. Where this page and a tool's own
description differ, the tool's description wins. Engineering defaults are deliberately not repeated here:
when you leave a field out, report the value the tool used (from `normalized_input` or the result), and say
it was filled in by the tool.

Most tools take one object argument, `request`. The three engine tools take `design_input`. Every tool returns
an `error` object instead of a result when the input is invalid; quote it.

## Steady-state rating

### rate_cable — rate one cable (schema `cable-server/v1`)

Argument `request`, either the structured form or `design_input` alone.

- `method`: `iec60287` (default) or `neher_mcgrath`.
- `cable`: exactly one of `library_id` (from `list_cable_library`) or `construction`; optional
  `max_system_voltage_um_kv` (Um of the cable's voltage rating, recorded and checked by the engine).
- `construction` (engine field names): required `conductor_material` (`copper` or `aluminum`),
  `conductor_size_mm2`, `insulation_type`; typed optional `conductor_diameter_mm`, `conductor_r20_ohm_per_km`,
  `insulation_thickness_mm`, `sheath_material`, `sheath_thickness_mm`, `oversheath_material`,
  `oversheath_thickness_mm`, `max_conductor_temp_c`. Other cable-construction fields the engine accepts
  (screens, wires, armour, ks/kp, resistivities) may be added; installation fields may not.
- `installation`: `installation_method` (`direct_buried`, `buried_ducts`, `sand_filled_trough`, `in_air`,
  `unfilled_trough`, `ventilated_tunnel`), `cable_formation` (`trefoil`, `flat`, `isolated`, `two_flat`),
  `burial_depth_m`, `axial_spacing_mm`, `sheath_bonding` (`solid`, `single_point`, `cross_bonded`), `extra`
  (further engine installation fields: ducts, duct bank, circuits, crossings, heat sources, soil drying, as
  for `compute_ampacity_iec`).
- `site`: `ambient_temp_c` (required), `soil_thermal_resistivity_k_m_per_w`, optional `location`
  (`latitude`, `longitude` required; optional `elevation_m`, `record_start_year`, `record_end_year`) to compare
  the ambient with the measured US site value. The check never changes `ambient_temp_c`; when it flags the
  ambient it blocks design-ready, and the verified-envelope status is unchanged.
- `system`: `frequency_hz` (required), `system_voltage_kv` (defaults to the library cable's),
  `max_conductor_temp_c` (overrides the cable's).
- Optional: `loss_load_factor` (IEC: adds the IEC 60853 cyclic peak for a flat-topped cycle, buried cables;
  Neher-McGrath: the paper's loss factor), `load_factor` (Neher-McGrath only), `equation_options`.
- `design_input` alone: a complete engine input as for `compute_ampacity_iec`; its `calculation_method` is used
  when `method` is not given, and a `method` that disagrees is refused.
- Returns amps, conductor temperature, losses, thermal resistances, method and standard, a verified-envelope
  flag, warnings, snapshot hash and ledger hash. Errors: `{"error": {"code", "message", "details"}}`.

### compute_ampacity_iec — full IEC 60287 rating

Argument `design_input` (`calculation_method` is set to `iec60287` for you).

- Required: `conductor_size_mm2`, `ambient_temp_c`, `max_conductor_temp_c`, `system_voltage_kv`,
  `frequency_hz`, `conductor_r20_ohm_per_km`, `conductor_diameter_mm`, `insulation_thickness_mm`,
  `sheath_material`, `sheath_thickness_mm`, `oversheath_thickness_mm`, `cable_formation` (`trefoil` or
  `flat`), `sheath_bonding`; for buried cables `burial_depth_m` and `soil_thermal_resistivity_k_m_per_w` (in a
  sand-filled trough, `trough_sand_rho_k_m_per_w` instead; in a ventilated tunnel, a `tunnel` object instead of
  `burial_depth_m`).
- Optional: `installation_method`, `equation_options` (named alternatives to the default equations), armour,
  duct, duct bank, air, soil drying, `circuits` (each optionally with `load_ratio`, a fixed `current_a`, or its
  own cable), crossings, `crossing_circuit` (with its own cable, `burial_depth_m`, `crossing_angle_deg`),
  `heat_sources`, `surface_heat_transfer_w_per_m2k`, `soil_zones`, `numerical_ground`,
  `conductor_construction`, `flat_transposition`, `spacing_sections`, `spacing_variation_unknown`, wire screens
  (`screen_type` `wires`).
- Three-core cable: `cable_type` `three_core` and `core_screen` (`individual` or `common`) instead of
  `cable_formation` and `sheath_bonding`.
- Pipe-type cable: `cable_type` `pipe_type`, `pipe_inner_diameter_mm`, `pipe_wall_thickness_mm`, `pipe_filling`
  (`oil` or `gas`).
- Two single-core cables: `cable_formation` `two_flat` (DC with `current_type` `DC`); one DC cable alone:
  `isolated`. One two-core belted cable: `cable_type` `two_core`.
- Returns `design_input` as sent, `normalized_input` (every value used, defaults included), outputs,
  intermediate values, snapshot hash and ledger hash.

### compute_ampacity_neher_mcgrath — Neher-McGrath (AIEE 1957)

Argument `design_input` (`calculation_method` is set to `neher_mcgrath` for you). An opt-in alternative with a
narrow scope; read the tool description for it.

- Required: `conductor_size_mm2`, `ambient_temp_c`, `max_conductor_temp_c`, `system_voltage_kv`,
  `frequency_hz`, `conductor_diameter_mm`, `insulation_thickness_mm`, `sheath_material`, `sheath_thickness_mm`,
  `sheath_bonding`, `axial_spacing_mm`, `proximity_effect_kp`, `skin_effect_ks` (or
  `conductor_inner_diameter_mm` with `proximity_conductor_diameter_mm` for an annular conductor),
  `insulation_rho_k_m_per_w`, `insulation_relative_permittivity`, `insulation_power_factor`,
  `soil_thermal_resistivity_k_m_per_w`, `cable_positions` (each `{horizontal_offset_m, depth_m}`).
- Ducts: `installation_method` `buried_ducts` with `duct_material`, `duct_inner_diameter_mm`,
  `duct_outer_diameter_mm` and the four `duct_bank_*` fields.
- Optional: `loss_factor` or `load_factor`, `soil_diffusivity_m2_per_s`, `conductor_r20_ohm_per_km`,
  `oversheath_thickness_mm`, `oversheath_rho_k_m_per_w`, `duct_rho_k_m_per_w`, `duct_bank_*` as a backfill.
- Returns the same shape as `compute_ampacity_iec`; intermediate values are in the paper's units.

### compute_ampacity — screening estimate only

Flat arguments: `conductor_size_mm2`, `ambient_temp_c`, `max_conductor_temp_c`, `conductor_material`,
`insulation_type`, `installation_factor`, `soil_thermal_resistivity_k_m_per_w`, `number_of_loaded_cores`.
Every argument has a default; never let a default stand in for a value the engineer did not give without
saying so. The result is a preliminary estimate, not a rating.

## Selection and studies

### select_cable — smallest cable that carries the load

`request`: `load_a` (required), `candidates` (1 to 50, each `{library_id}` or `{construction}`),
`installation`, `site`, `system` (required, as for `rate_cable`), optional `method`, `loss_load_factor`
(the candidate must carry the load at its cyclic peak), `load_factor` (Neher-McGrath), `equation_options`.
Returns the selection, every candidate's rating, margin and envelope flag. If none carries the load,
`selected` is null and the largest rating is reported.

### run_study — cables × layouts × load cases

`request`: `cables` (each `{library_id}` or `{construction}`), `layouts` (each `{name, installation}`),
`load_cases` (each `{name, site, system}`, optional `loss_load_factor`, `load_factor`, `load_a`), optional
`method`, `equation_options`. At most 200 runs; above that it is refused with `study_too_large` before
anything runs. Returns one row per run (rating, envelope flag, margin to `load_a`, snapshot hash), the minimum
rating per layout and load case, and a summary.

## Cyclic, emergency and transient

### cyclic_rating_factor — cyclic factor M and peak current (IEC 60853-1/-2)

`request`: `design_input` (IEC input, `direct_buried` or `buried_ducts`), `load_cycle` (one of
`{"hourly": [24 loads], "peak_hour": h}`, `{"loss_load_factor": mu}`, or
`{"flat_top_hours": x, "fraction": p}`), optional `soil_diffusivity` (a value in m²/s, `table_ii`, or
`{dry_density_kg_per_m3, moisture_pct}`), `zone_diffusivity`, `method` (`auto`, `part1`, `part2`),
`soil_drying` (`{dry_soil_thermal_resistivity_k_m_per_w, soil_critical_temperature_c}`). Returns M, the rated
and peak currents, the terms, the cycle ordinates used, notes and a result hash.

### emergency_rating — emergency current (IEC 60853-2)

`request`: `design_input`, `duration_h`, `initial_current_a` (the steady load before), optional
`emergency_temperature_c`, `transient` (`uncorrected` or `corrected`), `circuit` (`long` or `short`),
`soil_diffusivity`, `zone_diffusivity`. Returns the emergency current with its terms, notes and a result hash.

### transient_temperature — temperature after a current step (IEC 60853-2)

`request`: `design_input` (buried directly, in buried ducts or in free air; single-core or three-core, not
armoured), optional `hours`, `current_a`, `circuit` (`long` or `short`), `soil_diffusivity`,
`zone_diffusivity`, or `steps` (up to 48 `{duration_h, current_a}`) with `initial_current_a`. Returns the
thermal circuit, capacitances, time constant and the conductor temperature at each time, notes and a result
hash.

## Short-circuit

### short_circuit_rating — permissible short-circuit currents (IEC 60949)

`request`: `duration_s`, `components` (1 to 20, each with `type` — `conductor`, `spaced_wires`, `sheath`,
`tape`, `touching_wires` or `braid` — `material`, `initial_temperature_c`, and `final_temperature_c` and/or
`fault_current_a`, plus the fields for its type, optional `name` and `thermal_contact_factor`), optional
`method` (`non_adiabatic` or `adiabatic`), `shared_fault` (`{current_a, components: [names]}`). The final
temperature limit is the engineer's input; never supply one yourself. Returns per component its area, K, the
non-adiabatic factor, the adiabatic and permissible currents and, with a fault current, the final temperature.

## Economic size

### economic_conductor_size — economic size (IEC 60287-3-2)

`request`: `sizes` (2 to 30, each `conductor_size_mm2`, `installed_cost_per_m`, and either
`resistance_ohm_per_m` or the size-dependent cable fields), `economics` (`max_current_a`, `length_m`,
`loss_hours_per_year`, `energy_price_per_wh`, `years`, `discount_rate_pct`; optional
`demand_charge_per_w_year`, `load_growth_pct`, `energy_price_growth_pct`, `loss_load_factor_growth_pct`,
`phases_per_circuit`, `circuits`, `mean_temperature`), optional `design_input` (rates each size). Returns each
size's costs and economic current range, the lowest-cost size, the economic size, notes and a result hash.

## Routes and circuit assignment

### rate_route — circuit rated section by section

`request`: `design_input` (the circuit and shared site defaults), `sections` (1 to 50, in chainage order, each
`name`, `from_m`, `to_m`, `inputs` that differ there; a section the engine cannot rate yet takes
`rated: false` and a reason), optional `name`, `axial_heat_flow` (`conductor` or `conductor_and_screen`).
Returns the circuit rating, the governing section, every section's rating and margin, `short_sections`,
whether every section is rated, and a route hash.

### route_study — cyclic, emergency or soil-drying study along a route

`request`: `route` (as for `rate_route`), `study` with `kind` `cyclic` (fields as for
`cyclic_rating_factor`), `emergency` (fields as for `emergency_rating`, or `initial_load_pct`), or
`soil_drying` (`mode` `partial` or `avoid`, `soil_critical_temperature_c`, and for partial
`dry_soil_thermal_resistivity_k_m_per_w`). Returns the circuit's value, the governing section and whether it
governs by a bound, every section's value, status and margin, bounded and open sections, notes and hashes.

### assign_circuits — which circuit in which position

`request`: `design_input` (site, installation and first circuit's cable, without `burial_depth_m` or
`circuits`), `positions` (2 to 8, each `horizontal_offset_m`, `burial_depth_m`), `circuits` (2 up to the number
of positions, each optional `name`, `load_ratio`, `cable`), optional `search` (`auto`, `exhaustive`, `local`).
Returns the best assignment with each circuit's position and current, the limiting cable, the best assignments
tried, the design input of the best one, notes and hashes.

## Site ambient

### site_ambient — design ambient soil temperature at a US site

`request`: `latitude`, `longitude`, `depth_m`, optional `elevation_m`, `record_start_year`, `record_end_year`,
`ambient_temp_c` (the ambient you plan to rate with, to compare). Returns the site value, the second-highest
and mean yearly maxima, stations with distance and record, cross-checks, excluded stations with reasons,
warnings, data credits and a table hash. Never changes the input.

## Capabilities, library, health and audit

- `list_iec_capabilities`: optional `status` (`verified`, `annex_checked`, `reference_checked`,
  `unit_tested`, `unverified`) and `group` (`installation`, `cable`, `bonding`, `group`, `site`, `sizing`).
  Returns each capability's inputs, clauses, verification status and source, what is not supported, and what
  each status means.
- `list_cable_library`: optional `rateable_only`. Each entry gives its id (use as `cable.library_id`), label,
  voltage, conductor, source document and page, whether it is rateable and, if not, what is missing.
- `list_standards_packs`: no arguments. Pack id, release version and integrity status of each pinned pack.
- `health`: no arguments. Standards packs availability and database connectivity.
- `get_run`: `run_id`. Run metadata, outputs and the full ledger event chain.
