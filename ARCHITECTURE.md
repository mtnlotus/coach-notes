# Architecture

## Why this exists

Health & Wellness Coaches document sessions as free-text progress notes in the EHR — there's no
structured field for "long-term goal" or "Well-Being Signs score." `coach-notes` bridges that gap:
it parses the coach's actual note format (not a purpose-built form), and produces two artifacts
downstream systems can use directly — a patient-facing PDF and a standards-compliant FHIR Bundle.

## Pipeline

```
DOCX / .txt notes ──▶ docx-reader.ts ──▶ paragraph text[]
                                              │
                                              ▼
                                    note-parser.ts (NoteParser)
                                    one RawNote per input file
                                              │
                                              ▼
                                    note-merger.ts (mergeNotes)
                                    sorted-by-date, most-recent-wins
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                                ▼
                    php-data.json (merged view)      php-notes.json (per-note array)
                              │                                │
                              ▼                                ▼
                    pdf-report.ts                    fhir-builder.ts
                    (generate-pdf)                    (generate-fhir)
                              │                                │
                              ▼                                ▼
                  personal-health-plan.pdf              fhir-bundle.json
                                                                │
                                                                ▼
                                                  fhir-reader.ts + pdf-report.ts
                                                  (generate-pdf-from-fhir)
```

Two JSON artifacts come out of parsing, not one, because they serve different consumers:

- **`php-data.json`** is the *merged* view — one current value per field — and is what the PDF
  renderer wants.
- **`php-notes.json`** is the *per-note* array, sorted by session date, and is what the FHIR
  builder wants, because the Bundle needs to preserve session history (one `Observation` per
  session, one `ServiceRequest` per action step actually recorded), not just the latest values.

## Parsing strategy: text landmarks, not a form schema

`NoteParser` (`src/lib/note-parser.ts`) extracts fields by matching literal landmark phrases the
coach's note template always contains — e.g. "What really matters to [Name]." for the patient
name, "Utilized Importance Ruler:" / "Confidence Ruler:" for readiness scores, "Goal N:" for
action step blocks. This is deliberately format-matching rather than NLP/LLM extraction: the
notes come from a small number of fixed EHR templates, matching is deterministic and auditable,
and a coach can trust that the same note always parses the same way.

The tradeoff: a note template change requires a parser change. `clinical-notes/` holds one
example note per stage (initial/middle/final visit) specifically so parser changes can be
verified against real formatting before being trusted on new data.

## Merge strategy: most-recent-wins, not accumulate

A patient may have several session notes. `mergeNotes()` (`src/lib/note-merger.ts`) sorts notes
by session date (then session number) and applies:

- **Scalar fields** (WBS scores, importance/confidence rulers, discharge plan) — the latest
  session's value replaces all earlier ones.
- **Long-term goals** — deduplicated by goal text. A goal stated in the initial visit and
  repeated verbatim in a later visit is one `Goal`, not two.
- **Action steps** — matched by position within a goal and updated with the latest status
  (e.g. "not met" → "in progress" → "completed") rather than appended as new steps.

This mirrors how a coach actually reads a patient's history: the current state of a goal matters
more than a full diff of every session, but the *history* of action steps taken toward it still
matters — which is why the FHIR Bundle (built from `php-notes.json`, not `php-data.json`) keeps
one resource per session rather than only the merged view.

## FHIR mapping (PCO IG)

`fhir-builder.ts` maps parsed fields onto the
[Person-Centered Outcomes IG](https://build.fhir.org/ig/HL7/pco-ig):

| PHP concept | FHIR resource | Profile / code |
|---|---|---|
| Patient identity | `Patient` | base R4 (name only — no MRN) |
| MAP / Sense of Purpose | `Observation` | base R4, SNOMED `247751003` — one per note when content changes |
| Well-Being Signs (Q1–3) | `Observation` | temporary Mountain Lotus CodeSystem (see below) — one per session |
| Long-term goal | `Goal` | `pco-gas-goal-profile` — deduplicated by text |
| Importance/confidence rulers | `Observation` | `pco-readiness-assessment` — one per session with scores, referencing the Goal |
| Action step(s) | `ServiceRequest` | base R4, linked via `pertainsToGoal` — one per session |

A resource is only emitted when the source note actually contains that section — the builder
does not synthesize empty/placeholder resources for missing data.

**Temporary coding:** Well-Being Signs LOINC codes have not yet been officially assigned, so the
WBS `Observation` uses a placeholder Mountain Lotus `CodeSystem`
(`http://mtnlotus.com/fhir/whole-health-cards/CodeSystem/well-being-signs`). This is the one
piece of the mapping expected to change post-launch, once HL7 assigns real codes.

## Reverse mapping: FHIR → PDF without the intermediate JSON

`fhir-reader.ts` (`bundleToPhpData`) lets `generate-pdf-from-fhir` render a PDF directly from a
Bundle someone already has (e.g. fetched from an EHR via `DocumentReference`), without needing
`php-data.json` at all. This is intentionally lossy in one direction: the Bundle doesn't encode
"Strengths & Values" or the discharge plan (there was no PCO-IG-appropriate place to put them),
so a PDF generated this way omits those sections. See the coverage table in
[SKILLS.md](SKILLS.md#generate-pdf-from-fhir) for exactly what survives the round trip.

## Known gaps

- No automated test suite — correctness is currently verified by running the pipeline against
  the example notes in `clinical-notes/` and inspecting output. A first test suite (parser
  landmark matching, merge precedence, FHIR resource shape) would be a good first contribution.
- WBS CodeSystem is a placeholder pending official LOINC assignment (see above).
- "Strengths & Values" and discharge plan are PDF-only — not represented in the FHIR Bundle.
