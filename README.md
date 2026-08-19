# Health and Wellness Coach Notes

Turn a Health & Wellness Coach's clinical progress notes into a **Personal Health Plan (PHP)** —
a patient-facing PDF and a [FHIR R4](https://hl7.org/fhir/R4) Bundle conforming to the
[Person-Centered Outcomes (PCO) IG](https://build.fhir.org/ig/HL7/pco-ig).

Part of the Whole Health platform.
Used both as a standalone CLI / Claude Code skill pipeline, and as a library
(`github:mtnlotus/coach-notes`) by [`wholehealth-dashboard`](https://github.com/mtnlotus/wholehealth-dashboard)
for in-app note parsing and PHP rendering.

```
DOCX or plain-text (.txt) progress notes
       │
       ▼  parse-notes
  php-data.json           ← structured intermediate JSON
       │
       ├──▶  generate-fhir  ──▶  fhir-bundle.json
       │                                │
       │                                ▼  generate-pdf-from-fhir
       │                          personal-health-plan.pdf
       │
       └──▶  generate-pdf   ──▶  personal-health-plan.pdf
```

## Prerequisites

- Node.js 20+
- [pnpm](https://pnpm.io/) — this repo uses pnpm exclusively; do not use npm or yarn

## Install

```bash
pnpm install
```

**macOS font note:** the PDF generator uses Arial from
`/System/Library/Fonts/Supplemental/`. On other platforms it falls back to built-in Helvetica
automatically.

## Usage

### Command line

```bash
# 1. Parse progress notes (DOCX or plain text) → output/php-data.json + php-notes.json
pnpm parse-notes clinical-notes/
pnpm parse-notes clinical-notes/plain-text/   # plain-text (.txt) notes

# 2a. Generate a FHIR R4 Bundle → output/fhir-bundle.json
pnpm generate-fhir

# 2b. Generate the patient-facing PDF from parsed JSON → output/personal-health-plan.pdf
pnpm generate-pdf

# 2c. Generate the patient-facing PDF directly from a FHIR bundle
pnpm generate-pdf-from-fhir output/fhir-bundle.json -o output/personal-health-plan.pdf
```

All scripts write to `output/` by default and create the directory automatically. Override any
output path with `-o <file>`.

### Via Claude Code

Open Claude Code in this repository and invoke the same pipeline as slash commands:
`/parse-notes`, `/generate-fhir`, `/generate-pdf`, `/generate-pdf-from-fhir`. See
[SKILLS.md](SKILLS.md) for the full skill reference, including every field the parser extracts
and every resource the FHIR bundle builder produces.

### As a library

`wholehealth-dashboard` and other TypeScript consumers can import the pipeline directly:

```ts
import { NoteParser, mergeNotes, buildBundleFromNotes, bundleToPhpData } from "coach-notes";
```

See [`src/index.ts`](src/index.ts) for the full public API.

## Scripts

```bash
pnpm parse-notes [path]              # DOCX/.txt → output/php-data.json + output/php-notes.json
pnpm generate-fhir                   # output/php-notes.json → output/fhir-bundle.json
pnpm generate-pdf                    # output/php-data.json → output/personal-health-plan.pdf
pnpm generate-pdf-from-fhir [file]   # fhir-bundle.json → PDF
pnpm build                           # compile TypeScript → dist/
pnpm typecheck                       # type check only, no emit
```

## Documentation

- [SKILLS.md](SKILLS.md) — full pipeline reference: field extraction rules, FHIR resource
  mapping, PDF sections, and the Claude Code skill definitions in `.claude/skills/`
- [ARCHITECTURE.md](ARCHITECTURE.md) — how notes are parsed and merged, and why
- [AGENTS.md](AGENTS.md) — context and constraints for AI coding agents working in this repo

## Examples

- [Example health coach progress notes](clinical-notes/) — de-identified real-world examples
- [FHIR Bundle resource templates](fhir-templates/) — reference PCO IG bundle

## Testing

No automated test suite yet — validate changes against the example notes in `clinical-notes/`
and `pnpm typecheck`. See [ARCHITECTURE.md](ARCHITECTURE.md#known-gaps) for open work.

## License

[Apache-2.0](LICENSE)
