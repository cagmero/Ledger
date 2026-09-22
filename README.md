# Ledger

**Paper and panels into verified records — entirely on your phone.**

An on-device capture desk for the iQOO 15. Point the camera at paper (invoices, challans, forms, lab sheets) or at a panel (energy meters, multimeters, machine and instrument displays). Ledger reads it, structures it into named fields, checks it against deterministic rules, and hands a verified spreadsheet and run report to the laptop over Office Kit.

No cloud. No connectivity. No per-page cost. The app ships without the `INTERNET` permission, so it cannot upload a page, a reading or an image.

| | |
|---|---|
| **Event** | iQOO Hackathon 2026 · City Battles |
| **Battle** | Hyderabad · 26–27 September 2026 |
| **Track** | Open Innovation |
| **Bucket** | Students |
| **Team** | `[TEAM NAME]` · `[MEMBER]`, `[MEMBER]` |
| **Target hardware** | iQOO 15 (Snapdragon, Hexagon NPU) |

> **Repository status.** This repo holds **planning artifacts only** — spec, architecture, build plan and validation rules. Per the hackathon's original-work rule, the competition app is built in a clean repository from 11:00 on Saturday 26 September. Pre-event spikes live in `spikes/` and are throwaway probes, not product code.

---

## The problem

Two kinds of data never make it into a system cleanly:

1. **Paper.** Invoices, challans, admission forms and lab sheets arrive printed and get keyed in by hand — often twice, with transcription errors nobody catches until audit.
2. **Panels.** Energy meters, multimeters, machine displays and lab instruments have no data port. Readings are taken by eye onto a clipboard and typed up hours later from memory.

The obvious fix — a cloud document API — is blocked or distrusted for client, patient and factory data, and the places this data lives (plant rooms, basements, godowns, field sites) often have no signal anyway.

**The laptop can't see paper or panels, and isn't allowed to send that data anywhere. The phone can do both.**

## What Ledger does

One pipeline, two kinds of input, four stages:

| Stage | What happens |
|---|---|
| **Read** | CameraX capture with frame selection; on-device text recognition pulls every figure off the page or display. |
| **Structure** | A small quantised open model running on the Hexagon NPU turns loose text into named fields. It only organises text it was handed. |
| **Validate** | Fixed rules check totals, formats, ranges and checksums. Low-confidence or failing fields are flagged, never guessed. See [VALIDATION.md](VALIDATION.md). |
| **Deliver** | Office Kit file transfer and clipboard carry the verified `.xlsx`/`.csv` and a run report to the laptop. |

**Rounds mode** applies the same pipeline to a fixed capture list: the operator is walked stop by stop, warned on out-of-range readings, and the run closes with a report.

**Review queue.** Anything the validator rejects or the model returns with low confidence lands here for a human tap. Nothing uncertain reaches the output file.

## Why this must run on-device

| | |
|---|---|
| **Privacy by construction** | No `INTERNET` permission in the manifest. Not a policy promise — a build-time fact a judge can check in the APK. |
| **Works where the work is** | Plant rooms, basements, godowns and field sites have no signal. Ledger never needs one. |
| **No cost per page** | Cloud document APIs charge per page forever. The NPU charges nothing, so volume stops mattering. |
| **Fast enough to feel native** | On-device inference returns in seconds, so capture keeps the rhythm of flipping through a stack. |

## Why this device

Remove any one of these and the product stops working:

- **The NPU** — batch capture is the point; CPU-only inference makes it unusable.
- **The camera** — the only sensor that can read a printed invoice or a glowing display.
- **Office Kit** — carries verified records to the laptop with nothing installed there and no network in between.

## Architecture

```
  PAPER ─┐
         ├─▶ [1] CameraX ─▶ [2] On-device OCR ─▶ [3] Small LM on Hexagon NPU
 PANELS ─┘                                              │
                                                        ▼
                              laptop ◀── [5] Office Kit ◀── [4] Rule validator
                          (.xlsx + run report)                    │
                                                                  ▼
                                                            review queue
```

Full detail, including model selection, fallbacks and the privacy model: [ARCHITECTURE.md](ARCHITECTURE.md).

**Design rule:** the model does its easiest job — structuring text it was already given. It does not read, do arithmetic, or make judgement calls. Correctness comes from deterministic rules, which is why the output is trustworthy enough to import.

## Scope for 30 hours

**Ships**
- One document type, end to end
- One panel type — digital displays
- Rounds mode with range warnings
- Review queue for flagged fields
- Office Kit export to spreadsheet
- Live stats screen: latency, battery, bytes sent

**Explicitly out of scope**
- Handwriting recognition
- Analogue needle dials
- "Any document, any format"
- Accounts and sign-in
- Multi-device sync
- Anything that needs a network

A well-built narrow product beats a broken broad one. Build order and cut lines: [BUILD-PLAN.md](BUILD-PLAN.md).

## Measured results

Filled in from the spike and the build — measured on device, shown live in the demo, never estimated.

| Metric | Method | Result |
|---|---|---|
| Field accuracy | Against a hand-checked set of real pages | `__%` |
| Seconds per page | Capture → verified record, NPU backend | `__s` |
| NPU vs CPU | Same model, same page, both backends timed | `__×` |
| Battery per 50 pages | Measured from battery stats | `__%` |
| Bytes sent | OS network counter, airplane mode on | `0` |

## Demo script

1. Airplane mode on, byte counter visible on screen.
2. Scan a stack of pages, then a live meter display.
3. One page is deliberately wrong — validation catches it and routes it to review.
4. Corrected record is accepted; the spreadsheet lands on the laptop through Office Kit.
5. Counter still reads zero. Stats screen shows latency and battery for the run.

## Repo map

```
README.md          this file — what Ledger is and why
ARCHITECTURE.md    components, models, data flow, privacy and performance model
VALIDATION.md      the deterministic rule set and output schema
BUILD-PLAN.md      pre-event spikes, 30-hour plan, Red/Green split, risk register
spikes/            throwaway pre-event probes (not product code)
deck/              pitch deck
```

## Team

| Member | Owns |
|---|---|
| `[MEMBER]` | Android and capture layer — CameraX, OCR, review queue, Office Kit hand-off |
| `[MEMBER]` | Model and validation — NPU runtime, field extraction, rule engine, benchmarks |

Prior work: 1st place Algorand Hack Series 1 · 1st place Cardano Asia Hackathon 2025 · Top 10 Finalist Solana Stable Hacks 2026 · Top 7 Algorand Hack Series 3 · Top 10 Algorand Hack Series 2.

## License

Apache-2.0 (see `LICENSE`).
