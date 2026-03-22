# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenVault is an open-source hardware design project built with KiCAD 10.0. It is licensed under the CERN Open Hardware License v2 - Strongly Reciprocal (CERN-OHL-S v2).

The primary component is the **nPM1300** (Nordic Semiconductor PMIC). Its datasheet is at `Hardware/Datasheets/nPM1300_datasheet.pdf`.

## Design Validation (KiCAD "Build & Test")

There are no CLI build commands. Validation is done inside KiCAD:

- **ERC (Electrical Rules Check)**: Run via Schematic Editor → Inspect → Electrical Rules Checker. Errors block manufacturing; warnings should be reviewed.
- **DRC (Design Rules Check)**: Run via PCB Editor → Inspect → Design Rules Checker. Catches routing violations before Gerber export.
- **Gerber export**: PCB Editor → File → Fabrication Outputs → Gerbers. This is the "build artifact."
- **BOM export**: Schematic Editor → Tools → Generate BOM → exports to `${PROJECTNAME}.csv`.

## PCB Design Rules (Default Net Class)

| Parameter       | Value   |
|-----------------|---------|
| Track width     | 0.2 mm  |
| Clearance       | 0.2 mm  |
| Via diameter    | 0.6 mm  |
| Via drill       | 0.3 mm  |
| Diff pair width | 0.2 mm  |
| Diff pair gap   | 0.25 mm |

## ERC Severity Reference

Key rules configured as **error** (must fix before sign-off):
`pin_not_connected`, `pin_not_driven`, `power_pin_not_driven`, `missing_power_pin`, `unannotated`, `label_dangling`, `wire_dangling`, `duplicate_reference`, `unit_value_mismatch`, `unresolved_variable`, `undefined_netclass`

Rules set to **warning** (review, don't necessarily block):
`endpoint_off_grid`, `footprint_link_issues`, `lib_symbol_issues`, `missing_bidi_pin`, `missing_input_pin`, `missing_unit`

## Version Control

Git-tracked files: `.kicad_sch`, `.kicad_pcb`, `.kicad_pro`

Git-ignored (per `.gitignore`): `.kicad_prl`, `*-backups/`, `_autosave-*`, `*.lck`, `*.net`, `*.xml` — do not commit these.

## File Layout

```
Hardware/
  OpenVault.kicad_sch   # Schematic
  OpenVault.kicad_pcb   # PCB layout
  OpenVault.kicad_pro   # Project config (design rules, ERC/DRC settings, BOM config)
  Datasheets/           # Component datasheets
  Footprints/           # Custom PCB footprint libraries
  Symbols/              # Custom schematic symbol libraries
  3D/                   # Custom 3D models
```
