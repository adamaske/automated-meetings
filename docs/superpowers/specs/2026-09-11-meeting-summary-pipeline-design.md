# Automated Meeting Summaries: Scope and Design

Status: draft for review. Written 2026-09-11. Nothing is implemented yet.

## 1. What this is

A pipeline that takes a recording of a meeting and, without human effort, produces:

1. A cleaned, speaker-attributed transcript.
2. A structured record of what was discussed: decisions, action items, open questions, disagreements, numbers and dates that were stated.
3. A report (HTML and PDF) with graphics generated as code (SVG, Mermaid, HTML tables), not raster image generation.
4. An email to the relevant people carrying the summary, the report, and the transcript.

Facts from the owner (2026-09-11) that replace the earlier assumptions:

- **Purpose: an internal research-lab tool.** Meetings happen without a written record, and disagreements follow about what was agreed, planned, or scheduled. The record exists to settle those.
- **Language: English or Norwegian**, sometimes mixed.
- **Content over attribution.** Who said what is secondary; what was agreed is what matters. Speaker-name mapping is best-effort and never blocks a record.
- **Delivery v1: one email to the owner**, who forwards after approval. No distribution logic.
- **Volume: at most about 60 minutes of meeting audio per week.** Cost is not a constraint if the output is good. The owner's research cluster can host servers, so self-hosted speech-to-text (better Norwegian models exist there) is a real option.
- **Language: Python.** Model defaults: Claude Opus 5 for reasoning stages, Claude Sonnet 5 for cleanup.

Open decisions are tracked on the wayfinder map at `.scratch/meeting-record/map.md`. The glossary is `CONTEXT.md`.

## 2. Approaches considered

**A. Glue existing products.** Zoom/Meet/Teams already produce transcripts and AI summaries. Forward those through a small LLM step to a mail merge. Cheapest and fastest, but you get their summary quality, no control over the extraction, and no path to custom graphics. Rejected because the graphics and the structured extraction are the point of the project.

**B. Code-orchestrated pipeline of single LLM calls (recommended).** Each stage is one deterministic step: a speech-to-text call, then three or four Claude calls with fixed prompts and JSON schemas, then a renderer, then a mailer. Code owns the control flow. Predictable cost, easy to test each stage in isolation, easy to swap a vendor. This is the "workflow" tier in Anthropic's own guidance and it fits because the task is fully specifiable in advance.

**C. One agent does everything.** A Managed Agent with tools for transcription, rendering, and email, told "summarize this meeting and send it." Most flexible, but cost and latency vary run to run, failures are harder to diagnose, and you cannot unit-test a stage. Nothing in the requirements needs open-ended exploration. Rejected for v1. Worth revisiting only if v2 adds things like "look up the Jira tickets mentioned."

Recommendation: B.

## 3. Architecture

```
 ingest ──► transcribe ──► clean ──► extract ──► compose ──► render ──► review ──► deliver
 (file,     (STT vendor,   (Sonnet)  (Opus,      (Opus,      (code:     (human    (email
  metadata)  diarized)               JSON        HTML +      SVG/PDF,    or        API)
                                     schema)     graphics    no LLM)    auto)
                                                 spec)
```

Every stage reads one artifact from a job directory and writes one artifact. A job is a folder (local disk in v1, object storage later) named by meeting ID:

```
jobs/<meeting-id>/
  meeting.json          # metadata: title, start time, attendees (name+email), language, audio path
  audio.<ext>
  transcript.raw.json   # STT output: segments with speaker label, start/end, text, confidence
  transcript.clean.json # speaker names resolved, disfluencies removed, paragraphs, same timestamps
  extraction.json       # decisions, actions, questions, data points, each with segment references
  report.spec.json      # what the report contains, which graphics, from which extraction items
  report.html / report.pdf / graphics/*.svg
  email.json            # to/cc, subject, body, attachments; awaiting approval or sent
  log.jsonl             # per-stage: model, tokens in/out, cost, duration, errors
```

A tiny orchestrator (a worker loop, not a framework) advances a job to the next stage and retries on transient failures. Stages are idempotent: re-running a stage overwrites its own output only.

### Stage 1: ingest

Input: an audio/video file plus a metadata JSON. Sources for v1: manual drop (CLI/upload), and a watched folder or webhook where Zoom/Meet/Teams cloud recordings land. Attendee list comes from the calendar event, never from the LLM. Video is stripped to audio with ffmpeg.

### Stage 2: transcribe

Send audio to a hosted speech-to-text vendor with speaker diarization enabled, pre-recorded mode. Output is normalized into one internal segment format so the vendor can be swapped.

Recommendation: Deepgram Nova-3 or AssemblyAI Universal, both roughly a quarter dollar per audio hour with diarization. Self-hosted WhisperX is about 10x cheaper per hour but adds a GPU to run and maintain; only worth it above roughly 500 meeting-hours a month or if audio must never leave your infrastructure.

### Stage 3: clean (Claude Sonnet 5)

Turn `Speaker 2: uh so we, we said the, the deadline is friday right` into `Maria: We agreed the deadline is Friday.` Tasks:

- Map diarization labels to attendee names. Inputs: the attendee list, and cues in the transcript (introductions, being addressed by name). Output includes a confidence per mapping. Unresolved speakers stay as "Speaker 2" rather than being guessed.
- Remove filler, false starts, repeated words. Keep meaning. Do not summarize.
- Fix obvious STT errors for names and terms using a per-organization glossary (product names, people, acronyms). The glossary is a text file the user maintains.
- Preserve segment timestamps so later stages can cite them.

Long transcripts are processed in chunks of about 15 minutes with 1 minute overlap, then stitched. Sonnet 5 is enough here; the task is mechanical.

### Stage 4: extract (Claude Opus 5, structured output)

One call with the full cleaned transcript and a JSON schema enforced via structured outputs. Schema, abbreviated:

```
decisions[]:   { statement, made_by[], segment_refs[], confidence }
actions[]:     { task, owner, due (if stated), segment_refs[], confidence }
questions[]:   { question, raised_by, resolved: bool, segment_refs[] }
data_points[]: { label, value, unit, stated_by, segment_refs[] }   # numbers, dates, amounts
topics[]:      { title, segment_range, one_line }
disagreements[]: { topic, positions[], resolved: bool, segment_refs[] }
```

Every item carries `segment_refs`, the transcript segment IDs it came from. This is the anti-hallucination mechanism: the report renderer refuses any item whose refs do not exist, and the review UI shows the quote next to the claim. Items the model is unsure about get low confidence and are shown as "possible" rather than dropped or asserted.

### Stage 5: compose (Claude Opus 5)

Input: extraction.json plus the meeting metadata plus a "graphics catalog" (below). Output: `report.spec.json`, a document outline that says which sections to include, in what order, with what prose, and which graphics to draw from which extraction items. The model writes prose and picks graphics. It does not write SVG.

### Stage 6: render (code, no LLM)

Renders `report.spec.json` to HTML and PDF using templates. The graphics catalog is a fixed set of chart and diagram types, each a small function that takes structured data and returns SVG:

- action item table with owner and due date
- decision log
- timeline of topics across the meeting (minutes on the x-axis)
- simple bar or line chart for data points that share a unit (only when the meeting actually stated the numbers)
- flow or dependency diagram via Mermaid, when the extraction found a sequence or dependency
- RACI-style owner matrix when there are 3+ actions and 3+ people

This is the main design choice relative to the original idea of having the LLM write the vector graphics directly. Reasons: consistent styling, zero tokens spent on SVG boilerplate, no risk of the model inventing a chart with numbers that were never said, and rendering is unit-testable. A free-form escape hatch stays available: the spec can include one Mermaid block the model authored, for a diagram the catalog does not cover. Mermaid is a compact text format, so the token cost is tiny.

PDF is produced from the HTML with a headless Chromium (Playwright) or WeasyPrint. SVG charts are also rasterized to PNG at this stage for email use (section 8, issue 5).

### Stage 7: verify (Claude Sonnet 5)

A second model reads the cleaned transcript and the rendered report text and flags: claims in the report with no support in the transcript, actions assigned to someone who never agreed to them, numbers that differ from what was said. Output is a list of flags attached to the job. In v1 flags are shown to the human reviewer. In v2 unattended mode, any flag above a threshold blocks sending.

### Stage 8: review and deliver

v1: a minimal web page (or even a CLI) shows the report, the flags, the proposed recipients, and a Send button. Recipients default to attendees plus action-item owners who were not present, restricted to an allowlist of known addresses from the calendar or directory. Nobody outside the allowlist ever receives mail from this system without a human typing the address.

Delivery via Amazon SES or Resend. The email body is a short plain summary (decisions, actions, next steps) with the PDF report and the cleaned transcript attached, and a link to the HTML report if hosting exists.

## 4. Skills and prompts

The pipeline uses fixed prompts per stage, not an open-ended agent, so "skills" here means two things.

**Runtime prompt packs** (one folder per stage, versioned in the repo): system prompt, the JSON schema, 2 to 3 worked examples, and the organization glossary. Changes to these are the main lever for quality and must be paired with the eval set in section 7.

**Development skills** for Claude Code working on this repo, written with the `superpowers:writing-skills` skill (the closest thing available here to skill-creator; it is the same idea): one skill describing the job artifact layout and stage contracts so any agent editing a stage knows the interfaces, and one for adding a graphic to the catalog (data shape, SVG function, template, test fixture). Those get written after this design is approved, not before.

Anthropic's Agent Skills (server-side skills used with code execution for docx/pptx output) are not needed in v1 because rendering is done in our own code. They become relevant if someone asks for a PowerPoint deck per meeting.

## 5. Cost model

Prices as of September 2026. Anthropic first-party API rates: Opus 5 $5 in / $25 out per million tokens, Sonnet 5 $2 / $10. Batch API halves those. Speech-to-text and email prices from vendor pages and third-party comparisons linked at the end.

Assumptions for one 60-minute meeting: about 9,000 spoken words, roughly 12k tokens plain, 18k tokens with speaker labels and timestamps. Thinking tokens estimated at moderate effort.

| Stage | Vendor / model | Tokens in / out | Cost |
|---|---|---|---|
| Transcribe | Deepgram Nova-3 batch, diarization included | 60 min audio | $0.26 |
| Clean | Sonnet 5 | 20k / 12k | $0.16 |
| Extract | Opus 5, structured output | 17k / 3k + ~5k thinking | $0.29 |
| Compose | Opus 5 | 10k / 8k + ~3k thinking | $0.33 |
| Render | code | none | $0.00 |
| Verify | Sonnet 5 | 22k / 1k | $0.05 |
| Email | SES | 1 to 10 emails | $0.00 |
| **Total** | | | **about $1.10** |

Variants:

| Configuration | Per 60-min meeting |
|---|---|
| Above, but all LLM stages through the Batch API (results within hours, fine for post-meeting) | about $0.70 |
| Sonnet 5 for every stage | about $0.60 |
| Opus 5 for every stage, interactive | about $1.50 |
| Self-hosted WhisperX instead of Deepgram (marginal cost, GPU excluded) | subtract about $0.23 |

Cost scales roughly linearly with meeting length. A 3-hour meeting is about $3.

Monthly examples:

| Volume | Variable cost | Fixed cost |
|---|---|---|
| 50 meetings, small team | about $55 | one small VM $20 to $40, email free tier |
| 400 meetings, one department | about $450 | VM $40 to $80 |
| 400 meetings, batch API | about $300 | same |

Prompt caching helps less than usual here because most input tokens are the transcript, which is unique per meeting. Cache the system prompts, schemas, examples, and glossary (together a few thousand tokens per stage); expect maybe 10 to 15 percent off the LLM line, not more.

The dominant cost is not the API. It is the review time if the summaries are not trusted, and the engineering time on audio capture if you go beyond "file arrives." Both are addressed below.

## 6. Latency

Batch STT on a 60-minute file: 1 to 3 minutes. Each Claude stage: 30 seconds to 3 minutes at moderate effort. Render: seconds. Total from "file arrived" to "ready for review": about 5 to 10 minutes. Batch API mode: up to a few hours, usually much less.

## 7. Testing and evaluation

- Unit tests per stage on fixtures: a fake STT output, a known cleaned transcript, a known extraction. Renderer tests compare SVG output to snapshots.
- Contract tests: every stage output validates against its JSON schema.
- **Eval set**: 10 to 20 real (consented) meeting recordings with a hand-written reference of decisions and actions. Score extraction on recall and precision of decisions and actions, and on owner attribution accuracy. Score cleanup on a small sample by reading it. No prompt change ships without running this. Anthropic's `claude-api build-eval` flow can scaffold the harness.
- Cost and token logging on every call, checked in CI against a per-meeting ceiling so a prompt change that triples thinking tokens is caught.

## 8. Issues found

These are ordered by how much they threaten the project, not by stage.

**1. Audio capture is the hard part, and it is not an LLM problem.** Getting audio out of Zoom, Google Meet, and Teams reliably means either a bot that joins calls (Recall.ai and similar charge roughly $0.50 to $1 per hour, and bots are visible to participants), relying on each platform's cloud recording plus its API and permissions, or a local recorder on one participant's machine. Each is a project. v1 scope: accept a file. Decide the capture path as its own sub-project once the pipeline produces summaries people trust.

**2. Diarization gives "Speaker 2", not "Maria".** Name mapping depends on people introducing themselves or being addressed by name. Expect 10 to 30 percent of speakers to stay unresolved in meetings where everyone knows each other. Mitigations: attendee list as input, glossary of names, per-speaker voice profiles later (some vendors support enrollment), and a one-click "who is Speaker 2" in the review page. Never let the model guess a name with low confidence, because a wrong owner on an action item is worse than no owner.

**3. Confident wrong summaries that get emailed are the main risk.** An action item attributed to the wrong person, a decision recorded that was only proposed, a number transposed. Once emailed it is the record. Mitigations built into the design: segment references on every claim, a verify stage, confidence levels surfaced as "possible" rather than hidden, and human approval before send in v1. The switch to unattended sending should be earned by eval numbers, not assumed.

**4. Recording consent and data handling.** Many jurisdictions need all-party consent to record; the EU and Norway fall under GDPR. Transcripts contain personal data. Audio and transcripts go to two third parties (STT vendor and Anthropic). Anthropic's API retains data 30 days by default and some models are not available under zero data retention. Needed: a consent notice at meeting start, a retention policy for the job folders (delete audio after N days, keep the report), and vendor data processing agreements. This is policy work, not code, but the code needs a delete-job command and a retention sweep from day one.

**5. Email clients do not render inline SVG.** Gmail strips it, Outlook desktop mangles it. This directly affects the "graphics as vector code" idea. The resolution: vector graphics live in the HTML and PDF report, and the email carries PNG rasterizations of the key chart(s) plus the PDF attachment. Mermaid diagrams likewise get rendered to SVG then PNG for email. The render stage handles both outputs from the same source.

**6. Free-form LLM graphics tend to invent data.** If the model is asked to "make a chart of what was discussed" it will happily draw bars for figures nobody said. The catalog-plus-schema design (section 3, stage 6) limits charts to data points with segment references. Keep the escape hatch (one Mermaid block) to diagrams, which describe structure, not quantities.

**7. Who receives the email.** "Relevant parties" is not something to infer from the transcript. A model reading "we should loop in legal" will try to email legal. Recipients come from the calendar attendee list and action-item owners resolved against a directory allowlist. Anything else requires a human to add it.

**8. Long meetings and cost creep.** A 3-hour all-hands with 30 speakers produces a 60k-token transcript and the extraction gets worse, not just more expensive. Chunked cleanup handles the mechanical part. For extraction, meetings over about 90 minutes should be split by topic segment and extracted per segment, then merged. Add a per-job cost ceiling that fails the job rather than running up a bill.

**9. Multilingual and mixed-language meetings.** Nova-3 and Universal both support many languages, and code-switching mid-meeting (Norwegian with English terms, say) is a known weak spot for all STT vendors. Set the language per meeting from metadata, and put technical English terms in the glossary. Test with a real mixed-language sample before promising it works.

**10. Vendor drift.** STT vendor formats and Claude model IDs change. The normalized segment format and a single `llm.py` wrapper with model IDs in config are the two seams that make swaps cheap. Do not let vendor types leak into stage code.

## 9. Out of scope for v1

- Live captioning or real-time summaries during the meeting. Doubles the STT cost, forces streaming everything, and the summary is better with the whole meeting available anyway.
- Meeting bots that join calls (see issue 1).
- Slides, screen-share OCR, or video understanding.
- Slack/Teams posting, Notion/Jira write-back. Easy to add as delivery targets once the report exists.
- PowerPoint output.
- Fully unattended sending.

## 10. Build order

Each is a separate plan and can be demoed on its own.

1. Job layout, orchestrator skeleton, ingest from file, transcribe with one vendor, normalized transcript. Demo: drop a file, get a diarized transcript.
2. Clean and extract stages with schemas, glossary, and the eval set with 5 recordings. Demo: extraction JSON with segment refs you can spot-check.
3. Compose, graphics catalog with 3 types (actions table, decision log, topic timeline), render to HTML and PDF. Demo: a report.
4. Verify stage, review page, email delivery with approval. Demo: end to end with a human click.
5. Grow the catalog, grow the eval set, tune prompts, decide on unattended mode and audio capture.

## 11. Open questions for the owner

1. Where do recordings come from today? This decides whether issue 1 is a week or a quarter.
2. Which meeting platform, and is its cloud recording already enabled?
3. Language(s) of the meetings.
4. Is there a hard requirement that audio never leaves your infrastructure? That changes the STT choice and the Anthropic retention discussion.
5. Roughly how many meetings per month, to size batch vs interactive.
6. Is a short human approval step acceptable per meeting in v1?

## Pricing sources

- Deepgram Nova-3: https://brasstranscripts.com/blog/deepgram-pricing-per-minute-2025-real-time-vs-batch and https://convertaudiototext.com/blog/deepgram-nova-3-explained
- AssemblyAI: https://www.assemblyai.com/blog/speech-to-text-api-pricing
- OpenAI transcription: https://costgoat.com/pricing/openai-transcription
- Self-hosted Whisper: https://www.spheron.network/blog/faster-whisper-gpu-cloud-production-deployment-guide/
- Email: https://www.buildmvpfast.com/api-costs/email
- Claude API: bundled claude-api skill pricing table, cached 2026-06-24
