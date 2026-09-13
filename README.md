# Nocta Hardware

Open hardware for the Nocta vault: an offline password and credential manager
with a 2.7 inch e-ink screen, optional Bluetooth LE, and no cloud service
behind it.

This repository is the design, not a product. The device is a prototype, it is
not for sale, and there is no ship date.

Built by [Nocta Labs](https://noctalabs.sh), Leuven, Belgium.

## What is in here

| Path | What it is |
| --- | --- |
| `Hardware/OpenVault.kicad_sch` | Schematic, KiCad |
| `Hardware/Schematic.pdf` | The same schematic, readable without KiCad |
| `Hardware/OpenVault.kicad_pcb` | Board layout, KiCad |
| `Hardware/production/` | Fabrication packages: gerbers, positions, BOMs |
| `Hardware/3D/` | STEP models for individual component packages |
| `Hardware/Datasheets/` | Datasheets for the parts used |
| `Hardware/Footprints/`, `Symbols/` | Project libraries |

## What is not in here yet

- **Firmware.** Still being written. It will be published when it is worth
  reading, and the licence for it is not decided.
- **The enclosure.** No named enclosure model is published yet.
- **Photographs.** Coming with the next write-up.


## Licence

[CERN Open Hardware Licence Version 2, Strongly Reciprocal](LICENSE)
(`CERN-OHL-S-2.0`). Strongly reciprocal means anything you make from this and
distribute has to stay open under the same terms. That is deliberate.

## Reading the files

Open `Hardware/OpenVault.kicad_pro` in KiCad 8 or later, or start with
`Hardware/Schematic.pdf` if you just want to read it. The fabrication packages
under `Hardware/production/` are what actually went to the board house.

## Use of AI

AI assistance, primarily Anthropic's Claude, is used in the development of this
hardware.

Its role is limited to review and analysis: reading datasheets and verifying pin
assignments and component values against them, reviewing the schematic and the
board layout, triaging ERC and DRC output, maintaining helper scripts, and
drafting documentation.

Component selection, the board stackup, the USB differential pair impedance and
all security related decisions are made by the maintainers. No AI system commits
to this repository, and every change is reviewed by a maintainer before it
reaches `main`.

No part of this design has been validated by an AI system, and maintainer review
does not amount to electrical or security qualification. The design should be
read against the component datasheets and verified independently before
fabrication or use. The warranty and liability disclaimers in the licence apply
in full.

Contributions developed with AI assistance are accepted, provided the use is
disclosed in the pull request and the contributor can account for the work
submitted.

## Telling us we are wrong

Open an issue. A mistake found in the schematic now is worth more to us than
a compliment later :)

For anything sensitive, mail hello@noctalabs.sh first and we will credit you
publicly once it is fixed.

## Related

- [noctalabs.sh](https://noctalabs.sh) and its [security page](https://noctalabs.sh/security)
- [The blog](https://noctalabs.sh/blog), where revisions get written up
