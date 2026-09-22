# Architecture

Ledger is a five-stage pipeline with one hard rule: **the language model never decides whether something is correct.** It structures text; deterministic code validates it. Everything runs on the device.

## Components

| # | Component | Responsibility | Candidate implementation |
|---|---|---|---|
| 1 | **Capture** | Camera preview, frame selection (sharpness and glare scoring), crop to page or display, torch control for panels | CameraX, Kotlin |
| 2 | **Recognition** | Text and digit extraction with per-token confidence and bounding boxes | On-device OCR (ML Kit text recognition); seven-segment displays may need a dedicated path — see Risks |
| 3 | **Structuring** | Loose OCR text → named fields as strict JSON | Quantised small open model (Gemma-class 1–4B or equivalent) on the Hexagon NPU via an on-device LLM runtime (LiteRT-LM or ExecuTorch with the QNN backend) |
| 4 | **Validation** | Deterministic checks, confidence policy, review queue routing | Pure Kotlin, no model involvement — see [VALIDATION.md](VALIDATION.md) |
| 5 | **Delivery** | Spreadsheet and run report generation, hand-off to the laptop | Local `.xlsx`/`.csv` writer + Office Kit file transfer and shared clipboard |

Supporting: **local store** (encrypted app storage, no external directories), **review queue UI**, **stats screen** (inference latency, backend in use, battery delta, bytes sent).

## Data flow

```
frame ──▶ quality gate ──▶ OCR ──▶ candidate text + confidences + boxes
                                          │
                                          ▼
                            prompt assembly (schema + text only)
                                          │
                                          ▼
                             small LM on NPU ──▶ strict JSON fields
                                          │
                                          ▼
                     validator: arithmetic · format · range · checksum
                              │                          │
                          passes                       fails / low confidence
                              │                          │
                              ▼                          ▼
                     record store (encrypted)        review queue (human tap)
                              │                          │
                              └──────────┬───────────────┘
                                         ▼
                          .xlsx + run report ──▶ Office Kit ──▶ laptop
```

## Model policy

- **Structured output only.** The model is given a field schema and the OCR text, and must return JSON matching that schema. Nothing else.
- **Rejected on malformed output.** JSON that doesn't parse or doesn't match the schema is retried once with a stricter prompt, then the record goes to review. No partial acceptance.
- **The model never sees arithmetic as its job.** Totals, tax and deltas are computed by code from the extracted numbers, then compared against the number printed on the page. A mismatch is a finding, not an error to be smoothed over.
- **Backend ladder.** NPU → GPU → CPU, selected at runtime and displayed on the stats screen, so the demo can show the NPU advantage honestly by switching backends on the same page.
- **Load on demand.** The model is loaded when a capture session starts and released when it ends. No resident model, no background service.

## Privacy model

| Claim | How it's enforced |
|---|---|
| Nothing leaves the device | No `INTERNET` permission in `AndroidManifest.xml`. The process cannot open a socket. |
| Nothing leaks to other apps | Records stay in app-private encrypted storage; no external storage writes, no content providers exported. |
| Sensitive fields are masked in exports | Configurable per document type — full values stay on device, exports carry masked copies where the rule set says so. |
| The claim is checkable | Stats screen shows a live byte counter from the OS network stats; the demo runs in airplane mode. |

The Office Kit hand-off is a local device-to-device transfer, so the record's whole life is: camera sensor → phone memory → encrypted local store → user-initiated transfer to the paired laptop.

## Performance and thermal model

Batch capture on a phone fails in two ways: it gets slow, or it gets hot. Both are addressed by shape rather than tuning.

- **Event-driven, not continuous.** Work runs on capture events. No background loop, no polling, no screen observation, so idle cost is zero.
- **Cheap gates first.** A frame that fails the sharpness or glare check never reaches OCR. OCR text that contains no candidate fields never reaches the model. Most rejected work costs milliseconds.
- **Model last and least.** Only the few strings that survive the gates are sent for structuring, with short prompts and a bounded output length.
- **Batch, don't stream.** In rounds mode, captures queue and are processed between stops rather than competing with the camera preview.
- **Measured, not claimed.** The stats screen records per-stage latency and battery delta for each run; those numbers go in the README and the pitch.

## Failure modes and fallbacks

| Failure | Fallback |
|---|---|
| NPU runtime won't initialise on the loaner device | Automatic drop to GPU/CPU backend; stats screen shows which is active. Pitch still works, numbers change. |
| OCR misreads a seven-segment display | Digit-specific path (fixed-region crop + template or small classifier) as a documented fallback; rounds mode allows manual correction in the review queue. |
| Model returns malformed JSON | One stricter retry, then review queue. Never a partial record. |
| Office Kit pairing fails at the venue | Export to a file on the phone and show it; hand-off is one stage, not the product. Practise pairing before Saturday. |
| Glare or skew on glossy paper | Quality gate rejects the frame and prompts recapture rather than accepting a bad read. |

## What we are deliberately not building

Handwriting, analogue dials, arbitrary document types, sign-in, sync, and anything requiring a network. Each is a known gap, stated up front, with a reason: they trade demo reliability for surface area.
