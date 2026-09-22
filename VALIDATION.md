# Validation rules and output schema

This is where Ledger's trustworthiness comes from. Every rule here is plain code — no model involvement, no probability, no "usually right". A record is exported only if it passes.

## Confidence policy

Each field carries two numbers: OCR confidence and whether the validator's checks passed.

| OCR confidence | Validator | Outcome |
|---|---|---|
| High | Passes | Accepted, written to the record |
| High | Fails | **Review queue** — flagged with the failing rule |
| Low | Passes | **Review queue** — flagged as unverified read |
| Low | Fails | **Review queue** — flagged, both reasons shown |

There is no fourth state and no automatic correction. A human taps to confirm, edit, or discard. The review reason is always shown in plain language ("line items sum to 4,820, printed total is 4,280").

## Document rules (invoice as the shipped type)

| Field | Checks |
|---|---|
| Line items | Each line: quantity × unit price = line total, within rounding tolerance |
| Subtotal | Sum of line totals = printed subtotal |
| Tax | Subtotal × rate = printed tax amount, within tolerance; rate must be from the allowed set |
| Grand total | Subtotal + tax = printed total |
| Invoice number | Present, non-empty, matches the expected pattern for the document type |
| Invoice date | Parses as a real date; not in the future; not implausibly old (configurable window) |
| GSTIN | 15 characters, correct structural pattern, checksum verifies |
| Currency amounts | Parse as numbers; no stray separators; negative only where allowed |

Rounding tolerance is configurable per document type and recorded in the run report, so an auditor can see what was accepted.

## Panel rules (digital displays as the shipped type)

| Check | Rule |
|---|---|
| Range | Reading falls within the configured min/max for that instrument |
| Unit | Unit read from the display matches the expected unit for that stop |
| Delta | Change since the previous reading for the same stop is within the configured limit (catches a digit misread that passes the range check) |
| Monotonic | For cumulative meters, the reading cannot go backwards |
| Digit sanity | Decimal placement matches the instrument's known precision |
| Completeness | In rounds mode, every stop on the list has a reading before the run can close |

Out-of-range and out-of-delta readings warn immediately on screen at the point of capture, so the operator can re-read while still standing in front of the instrument. That is the whole point of doing this on the phone rather than at a desk later.

## Record schema

Every accepted capture is stored as one JSON record; the export flattens these into rows.

```json
{
  "record_id": "uuid",
  "type": "invoice | panel_reading",
  "captured_at": "ISO-8601 local timestamp",
  "source": {
    "capture_mode": "single | rounds",
    "run_id": "uuid, present in rounds mode",
    "stop_id": "string, present in rounds mode"
  },
  "fields": {
    "<field_name>": {
      "value": "normalised value",
      "raw_text": "text as read",
      "ocr_confidence": 0.0,
      "status": "accepted | reviewed | corrected"
    }
  },
  "validation": {
    "passed": true,
    "checks_run": ["line_item_math", "subtotal", "tax", "grand_total", "gstin_checksum"],
    "findings": []
  },
  "review": {
    "required": false,
    "reasons": [],
    "resolved_by_user": false
  },
  "runtime": {
    "backend": "npu | gpu | cpu",
    "ocr_ms": 0,
    "model_ms": 0,
    "validate_ms": 0
  }
}
```

## Exports

**Spreadsheet** (`.xlsx`, `.csv` fallback) — one row per record, columns being the document type's fields plus:

| Column | Meaning |
|---|---|
| `record_id` | Ties the row back to the stored record |
| `captured_at` | When it was captured |
| `status` | `accepted` or `corrected` |
| `flags` | Any findings that were reviewed and cleared |

**Run report** (Markdown, transferred alongside) — records captured, accepted, sent to review, corrected; the tolerance and range settings in force; per-stage timings; backend used; bytes sent (0). This is the artifact that makes the run auditable, and it is what turns Ledger from a scanner into a record-keeping tool.

## Masking

Fields marked sensitive in a document type's schema are stored in full on-device but written masked into exports (for example, all but the last four characters of an identifier). The masking rule applied is named in the run report.
