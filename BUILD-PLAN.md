# Build plan

Two phases: a pre-event spike week that removes unknowns, and a 30-hour build that only assembles known-good parts.

## Phase 1 — spike week (before 26 September)

Throwaway probes in `spikes/`, not product code. Each has one question and a yes/no answer. Anything still red by Thursday changes the plan, not the schedule.

| # | Spike | Question it answers | Done when |
|---|---|---|---|
| 1 | Model on NPU | Does a quantised small open model initialise and run on a Snapdragon Android device through the chosen runtime? | A prompt returns text, with the backend logged |
| 2 | Backend timing | How much faster is the NPU than the CPU for our prompt shape? | Same prompt timed on both, numbers recorded |
| 3 | Structured output | Does the model reliably return schema-matching JSON from OCR text? | 20 runs, parse-success rate recorded |
| 4 | OCR on paper | Does on-device OCR read a real printed invoice well enough to work with? | Field-level read accuracy on 10 real pages |
| 5 | OCR on displays | Can we read a seven-segment or LCD display, and is a dedicated path needed? | Verdict plus fallback decision |
| 6 | Office Kit | Pairing, file transfer, clipboard, remote control — all working, both directions | A file moves phone → laptop in under a minute, unaided |
| 7 | Cold start | How long from app launch to first verified record? | Timed on device |
| 8 | Battery | What does a 50-page run cost? | Battery delta recorded |

**Output of the week:** a one-page results table with real numbers, and a final decision on model, runtime and OCR path. Those numbers seed the README and the pitch.

## Phase 2 — the 30 hours

Clock starts Saturday 10:00; active build from 11:00. The competition repo is created clean at 11:00.

| Window | Light | Focus |
|---|---|---|
| Sat 10:00–11:00 | — | Opening, teach-in, device handover. Pair Office Kit immediately; confirm the loaner runs the spike build. |
| Sat 11:00–13:00 | Green | Skeleton: capture → OCR → model → JSON on screen. One document type only. |
| Sat 13:00–15:30 | Red | Prompt and schema tuning on the phone; build the seed page set; test captures under venue lighting. |
| Sat 15:30–16:30 | Green | Mentor round. Bring the sharpest question, not a status update. Validator work continues. |
| Sat 16:30–19:00 | Red | Review queue UI, field editing, capture flow polish on device. |
| Sat 19:00–22:00 | — | **Evaluation round 1.** Show the working read-to-record path end to end, and say what is coming. |
| Sat 22:00–00:00 | Green | Validation engine: arithmetic, formats, ranges, checksums. Findings surfaced in the UI. |
| Sun 00:00–01:00 | Green | Office Kit hand-off: spreadsheet and run report land on the laptop. **Core product complete by 01:00.** |
| Sun 01:00–06:30 | Red | Rounds mode, range warnings, stats screen, copy, empty and error states. Rotate sleep. |
| Sun 06:30–09:00 | Green | Panel capture path, measurement run for the numbers, bug fixes only. |
| Sun 09:00–12:00 | — | **Evaluation round 2.** Demo on the phone in airplane mode. |
| Sun 12:00–13:30 | Red | Rehearse the pitch on the phone, twice, timed. Freeze the build. |
| Sun 13:45 → | — | Final pitches, 3–5 minutes. |

### Cut lines, in the order things get dropped

1. Panel capture path (invoice path alone still demos the whole pipeline)
2. Rounds mode
3. Masking in exports
4. Stats screen detail beyond the byte counter
5. Anything cosmetic

Nothing on this list touches capture → validate → deliver. That path is protected.

### Red Light discipline

Phone-only hours are not downtime. They are for: prompt and schema tuning, seed data creation, capture testing under real lighting, UI copy, review-queue flows, demo rehearsal, and device telemetry that counts toward 15% of the score. Heavy code lands in Green Light windows only, which is why the schedule above front-loads architecture into the first two hours.

### Office Kit discipline

Office Kit usage is 10% of the score and read from device data, not self-reported. It is also genuinely the fastest way to work here: mirror the phone for development, move every build across by file transfer, use the shared clipboard for logs and prompts, and drive the phone by remote control during Red Light. Default to it rather than reaching for it.

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| NPU runtime fails on the loaner device | Medium | Backend ladder to GPU/CPU; spike 1 and 2 establish both sets of numbers in advance |
| OCR struggles with the venue's lighting | Medium | Quality gate with recapture prompt; test under fluorescent and low light during spike week |
| Model returns unusable JSON under time pressure | Medium | Strict schema, one retry, then review queue; measured in spike 3 |
| Seven-segment displays unreadable | Medium | Dedicated digit path as fallback; panel path is cut line 1 if it doesn't hold |
| Office Kit pairing problems at the venue | Low | Pair at 10:00 during the teach-in; local file export as a backup path |
| Scope creep into a second document type | High | Cut lines above are agreed before the event and not renegotiated during it |
| Exhaustion degrading Sunday morning work | High | Rotating sleep in the 01:00–06:30 window; Sunday morning is bug fixes only, no new features |
