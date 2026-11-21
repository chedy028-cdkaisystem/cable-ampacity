---
name: cable-ampacity
description: >-
  Rates power cables with the CDK AI System cable-ampacity MCP tools: steady-state ampacity (current rating,
  current-carrying capacity) by IEC 60287 or Neher-McGrath, the smallest cable for a load, studies over cables,
  layouts and load cases, cyclic, emergency and transient ratings, short-circuit limits, economic conductor size,
  ratings along a route, which circuit goes in which duct-bank position, and the design ambient soil temperature
  at a US site. Use when an engineer asks for a cable rating or derating, compares installations, sizes a cable
  for a load, or asks how soil, depth, spacing, bonding or ambient change the rating. Every number comes from the
  tools. Not for voltage drop, protection settings, building-wiring table lookups (NEC, BS 7671), or questions
  about the wording of IEC or CIGRE documents.
---

# Cable ampacity (CDK AI System)

These are MCP tools for Claude. The calculations run on the hosted CDK AI System server at
`cable-ampacity-mcp.onrender.com`. There is no account and no key. Your job is to turn the engineer's
description into a tool request, call the right tool, and report what it returns, faithfully.

The per-tool field cheat sheet is in [references/tools.md](references/tools.md). Read the section for a tool
before you build its request.

## 0. Check the tools are connected

Before anything else, check that the cable-ampacity tools are available (for example `rate_cable`,
`compute_ampacity_iec` and `list_iec_capabilities`). If they are not, say:

> The cable-ampacity tools aren't connected, so I can't rate this cable. Add the Cable Ampacity connector
> (from this plugin's Connectors tab, or `/mcp` in Claude Code) and ask again.

Then stop. Never estimate a rating yourself, from tables, rules of thumb, earlier results or memory.

If a call fails because the server cannot be reached, call `health` once. Report what it says, and do not
retry the calculation in a loop.

## 1. Gather the inputs (never invent them)

Collect what the engineer stated, with units, exactly as given. Compare it with the fields the chosen tool
needs (see [references/tools.md](references/tools.md)). Ask for everything that is missing in **one grouped
question**, naming each field in plain words with its unit. Do not ask one field at a time.

Typical groups for a rating:

- **Cable.** A `library_id` from `list_cable_library` (only entries marked rateable can be rated by id), or a
  full construction from the manufacturer's datasheet: conductor material and size, conductor diameter, DC
  resistance at 20 °C, insulation type and thickness, metallic sheath or screen, oversheath, armour where
  present, maximum conductor temperature.
- **Installation.** Method (direct buried, ducts, sand-filled trough, in air, unfilled trough, ventilated
  tunnel), formation (trefoil, flat, isolated, two flat), burial depth, spacing for flat formations, sheath
  bonding, and any ducts, duct bank, further circuits, crossings or heat sources.
- **Site.** Ambient temperature, soil thermal resistivity for buried cables, and optionally the site location
  (US latitude and longitude) for the site-ambient comparison.
- **System.** Voltage, frequency and, if different from the cable's, the maximum conductor temperature.
- **Method.** IEC 60287 (the default) or Neher-McGrath.

Rules:

- **Never fill in a value silently.** If the engineer says "use typical values" or leaves something out, name
  each value you propose, say it is your assumption, and wait for an explicit yes before calling a tool.
  Values from a catalogue or an earlier project are still assumptions.
- **Never rely on a tool's own defaults without saying so.** Some tools fill in defaults for missing fields.
  When you leave a field out, tell the engineer, and report the value the tool used from `normalized_input`.
- **Units as the field names say:** `_m` metres, `_mm` millimetres, `_mm2` mm², `_c` °C, `_kv` kV, `_hz` Hz,
  `_k_m_per_w` K·m/W. When you convert, write the conversion down in the answer.
- **Build each request from the engineer's own data.** Library cables are sources, not defaults for a new job.

## 2. Pick the tool

All 20 tools, grouped by job. When two fit, prefer the one named first in its group.

**Steady-state rating**

- `rate_cable`: one cable in one installation, IEC 60287 (default) or Neher-McGrath, with the verified-envelope
  flag and warnings. The default choice for "what is the rating of this cable".
- `compute_ampacity_iec`: the full IEC 60287 engine input (`design_input`), for cases `rate_cable`'s structured
  fields don't reach: three-core, two-core and pipe-type cables, crossings, soil zones, numerical ground,
  equation options.
- `compute_ampacity_neher_mcgrath`: the Neher-McGrath 1957 method, when the engineer asks for it and the case
  fits its scope (see the tool description).
- `compute_ampacity`: a preliminary screening estimate only. Use it only when the engineer asks for a quick
  screening figure, and label the result as a screening estimate, not a rating.

**Selection and studies**

- `select_cable`: the smallest of several candidate cables that carries `load_a`.
- `run_study`: every cable × every layout × every load case. It is refused above its run limit with
  `study_too_large`; offer to split the study.

**Cyclic, emergency and transient**

- `cyclic_rating_factor`: the cyclic rating factor and peak current for a daily load cycle (IEC 60853).
- `emergency_rating`: the emergency current for a period after a steady load (IEC 60853-2).
- `transient_temperature`: conductor temperature over time after a current step or a series of load steps.
- For a quick cyclic peak inside a steady-state rating, `rate_cable` and `select_cable` also take
  `loss_load_factor`.

**Short-circuit**

- `short_circuit_rating`: permissible short-circuit currents of conductors, sheaths, screens, tapes and
  armour (IEC 60949), or the final temperature for a given fault current.

**Economic size**

- `economic_conductor_size`: the economic conductor size (IEC 60287-3-2) from candidate sizes and costs. It is
  a sizing, not a rating; with a `design_input` it also rates each size.

**Routes and circuit assignment**

- `rate_route`: one circuit rated section by section along its route; the lowest section governs.
- `route_study`: a cyclic, emergency or soil-drying study on every section of a route.
- `assign_circuits`: which circuit goes in which duct-bank or trench position to carry the most.

**Site ambient**

- `site_ambient`: the design ambient soil temperature at cable depth for a US site, from measured station data.
  It never changes the engineer's input; compare and report.

**Capabilities, library, health and audit**

- `list_iec_capabilities`: what the IEC engine supports and how each capability is verified. Call it before
  stating any limit (section 5).
- `list_cable_library`: the cable library with each entry's source and whether it is rateable.
- `list_standards_packs`: the pinned standards packs and their integrity status.
- `health`: whether the server, its standards packs and its database are ready.
- `get_run`: a previous run's details and its full ledger event trail, by `run_id`.

## 3. Present the result

Write the answer in this order. Report the tool's numbers; never recompute, scale or adjust them.

1. **Ampacity and temperature first.** `ampacity_a` (rounded to the nearest ampere in the headline, unrounded
   in the details) and `conductor_temperature_c`, and what they are the rating of (cable, installation,
   method).
2. **Losses** (`losses`, including λ1 and λ2 where given) and **thermal resistances** T1 to T4
   (`thermal_resistances_k_m_per_w`).
3. **Method.** `method.standard_basis`, `method.rating_status` and the equation options used, or "defaults
   throughout".
4. **Verified envelope.** Show `verified_envelope.inside` and `verified_envelope.status` exactly as returned.
   When `inside` is false, list every entry of `verified_envelope.reasons`.
   - Call a result a *tested combination* only when `inside` is true.
   - `untested_combination` means the run sits inside the range of tested cases but is not itself one of
     them. It is not verified.
5. **Warnings.** Every entry of `warnings`, quoted, not shortened and not summarised away.
6. **Site ambient.** If the result has `site_ambient` with `blocks_design_ready: true`, say the result is not
   design-ready at that ambient.
7. **Inputs.** Each input marked *stated*, *converted* (with the conversion), *assumed* (accepted by the
   engineer) or *filled in by the tool* (from `normalized_input`).
8. **Audit.** The snapshot hash and ledger hash or result hash, and `run_id` where returned.

For other tools, lead with their headline value, then the same discipline:

- `select_cable`: the selected cable, then each candidate's rating, margin and envelope flag. If `selected` is
  null, say no candidate carries the load and give the largest rating.
- `run_study`: a table of runs with rating, envelope flag, margin and warnings, then the minimum per layout
  and load case.
- `cyclic_rating_factor`, `emergency_rating`, `transient_temperature`, `route_study`: the factor, peak,
  emergency current or temperatures, the governing section where relevant, and every note.
- `short_circuit_rating`: per component, the permissible current or final temperature, and every note.
- `economic_conductor_size`: the economic size, the lowest-cost size, and each size's costs.
- `rate_route`, `assign_circuits`: the circuit rating, the governing section or best assignment, each section
  or circuit, and every section left unrated with its reason.
- `site_ambient`: the site value, the stations used and excluded, warnings and data credits.

## 4. Errors

Errors come back as an `error` object, for example `{"error": {"code", "message", "details"}}`. Quote the
`code` and `message` verbatim, and the `details` where they name fields.

- `invalid_request`, `engine_rejected_input`, or any refusal naming a field: ask the engineer to correct or
  supply exactly what it names. Never retry with values you guessed.
- `unknown_cable`: offer the entries from `list_cable_library`.
- `study_too_large`: offer to split the study into smaller runs.
- Never change the inputs, switch method or installation, or drop a field to get around a refusal. A refusal
  usually means the method does not cover that case, and a rating of a different case is not an answer.

## 5. Limits

Before saying what the tools can or can't do, call `list_iec_capabilities`. Its `unsupported` list and each
capability's `status` are the source; don't state limits from memory. If the engineer's case is unsupported,
say it is not rated and do not give a number for it.

## 6. Every answer

- Never quote or paraphrase the text, tables, equations or figures of IEC standards or CIGRE brochures. Cite
  a standard, clause or case number at most.
- Never ask for, repeat or store a password, token or key in the conversation. These tools need none.
- End with: *"Screening result from an automated IEC 60287 or Neher-McGrath calculation."*
