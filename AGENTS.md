# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this package does

`coach-notes` turns a Health & Wellness Coach's clinical progress notes into a **Personal Health
Plan (PHP)**: a patient-facing PDF and a FHIR R4 Bundle following the
[Person-Centered Outcomes (PCO) IG](https://build.fhir.org/ig/HL7/pco-ig). You are acting on
behalf of an NBHWC Board Certified Health & Wellness Coach whose sessions produce a Why
Statement (Mission/Aspiration/Purpose), a Well-Being Signs (WBS) assessment, long-term goals, and
short-term action steps for each goal — the pipeline's job is to extract and structure that
content, not to interpret or coach.

It ships two ways:
- **A CLI / Claude Code skill pipeline** — `parse-notes → generate-fhir → generate-pdf(-from-fhir)`.
  See [SKILLS.md](SKILLS.md) for the full reference and `.claude/skills/*.md` for the skill
  definitions themselves.
- **A TypeScript library** (`github:mtnlotus/coach-notes`) — consumed by `wholehealth-dashboard`
  for in-app note parsing and PHP rendering. Public API is `src/index.ts`; anything not exported
  there is internal.

See [ARCHITECTURE.md](ARCHITECTURE.md) for how parsing and merging actually work.

## Examples

- [Example health coach progress notes](./clinical-notes/) — de-identified real-world examples
  reflecting actual EHR usage. Use these to validate parser changes.
- [FHIR Bundle resource templates](./fhir-templates/) — reference PCO IG bundle; treat as the
  authoritative shape for `generate-fhir` output.

## Resources

- [Introduction to Whole Health](https://www.va.gov/ann-arbor-health-care/programs/whole-health/)
- [How to set a SMART Goal](https://www.va.gov/WHOLEHEALTHLIBRARY/tools/how-to-set-a-smart-goal.asp)
- [VA Passport to Whole Health](https://www.va.gov/wholehealthlibrary/passport/)
- [Well-Being Signs (WBS)](https://www.va.gov/wholehealth/professional-resources/well-being-measurement.asp)

## Specifications

- [Person-Centered Outcomes (PCO)](https://build.fhir.org/ig/HL7/pco-ig)

## Core stack

- **Language:** TypeScript, strict mode, ES2022 target, **Node16** module resolution.
- **Package manager:** pnpm only. `pnpm-lock.yaml` is committed — run `pnpm install` after pulling.
- **Key dependencies:** `@smile-cdr/fhirts` (FHIR R4 types), `pizzip` + `fast-xml-parser` (DOCX
  parsing), `zod` (schema validation), `pdfkit` (PDF generation), `commander` (CLI args).
- **No enforced formatter/linter** — there is no Biome config in this package (unlike
  `shc-services` and `wholehealth-dashboard`). Match the style of surrounding code.
- **No automated test suite yet.** Validate parser/builder changes against the notes in
  `clinical-notes/` and run `pnpm typecheck`. See [ARCHITECTURE.md](ARCHITECTURE.md#known-gaps).

## Where things live

```
src/
├── models.ts                    # Zod schemas + TypeScript types (PhpData, Goal, WbsAssessment, ActionStep)
├── parse-notes.ts                # CLI: DOCX/.txt → php-data.json + php-notes.json
├── generate-fhir.ts               # CLI: php-notes.json → FHIR R4 Bundle
├── generate-pdf.ts                # CLI: php-data.json → patient PDF
├── generate-pdf-from-fhir.ts      # CLI: fhir-bundle.json → patient PDF
├── index.ts                       # Public library exports
└── lib/
    ├── docx-reader.ts              # DOCX ZIP → paragraph text[]; .txt → paragraph text[]
    ├── note-parser.ts              # NoteParser — extracts structured fields from one note
    ├── note-merger.ts              # mergeNotes() — most-recent-wins merge across sessions
    ├── fhir-builder.ts             # buildBundleFromNotes() and PCO resource builders
    ├── fhir-reader.ts              # bundleToPhpData() — FHIR Bundle → PhpData
    └── pdf-report.ts               # PHPReport — VA Whole Health styled PDF
```

## Constraints an agent must respect

- **Most-recent-wins, not accumulate.** When merging multiple session notes, scalar fields (WBS
  scores, goal ruler values, discharge plan) take the latest session's value. Goals are
  deduplicated by text; action steps are matched by position and updated in place — don't change
  this to naive concatenation.
- **Don't fabricate FHIR resources.** If a section is absent from a clinical note, omit the
  corresponding resource rather than emitting an empty one — this was an explicit prior
  correction (see [Enhancements.md](Enhancements.md)).
- **WBS uses a temporary CodeSystem.** Well-Being Signs LOINC codes are not yet officially
  assigned; the bundle uses `http://mtnlotus.com/fhir/whole-health-cards/CodeSystem/well-being-signs`
  until HL7 assigns real codes. Don't invent alternative interim codes — keep using this one so
  swapping to real LOINC codes later is a single find-and-replace.
- **De-identified example data only.** Never add real patient data to `clinical-notes/` or
  `fhir-templates/` — both are checked into a public repo.
- **`dev-notes/` is gitignored** and may contain non-deidentified working notes — never read it
  as a source of requirements for code you're about to commit; treat it as scratch space, not
  documentation.

## Making changes

- Run affected CLI scripts against `clinical-notes/` (and `clinical-notes/plain-text/`) after any
  change to `note-parser.ts`, `note-merger.ts`, or `fhir-builder.ts`, and spot-check the output
  JSON/PDF.
- Run `pnpm typecheck` before considering a change done.
- Keep `SKILLS.md`'s field-extraction and resource-mapping tables in sync with actual parser/
  builder behavior — agents (including future ones) rely on it being accurate, not aspirational.
- [Enhancements.md](Enhancements.md) is a running log of prior feature requests and corrections —
  skim it before re-implementing something that was already tried and adjusted.

## Open-source reference implementations

- [`@smile-cdr/fhirts`](https://github.com/smilecdr/FHIR.ts) — see
  [GETTINGSTARTED.md](https://github.com/smilecdr/FHIR.ts/blob/main/GETTINGSTARTED.md)
