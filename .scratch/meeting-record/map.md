# Map: Meeting record pipeline (wayfinder:map)

Effort slug: `meeting-record`. Tickets live in `issues/`. Research findings land in `research/`.

## Destination

An implementation-ready v1 spec for a lab tool that turns a meeting recording (English or Norwegian) into a trusted written record of what was agreed, planned, and scheduled, and emails it to the owner for approval and forwarding. Every vendor, model, hosting, and data-handling decision is made; the design doc at `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md` is updated to match; the next step after the map is `superpowers:writing-plans` for build step 1.

## Notes

- Domain: internal research-lab meetings, 2 to 10 people, English or Norwegian, sometimes mixed. No written record exists today, which causes disagreements later. The record's job is to settle those.
- Content over attribution: what was agreed matters, who said it is secondary. Speaker-name mapping is best-effort, never a blocker.
- Volume: at most about 60 minutes of meeting audio per week. Cost is not a constraint if the output is good. The owner's research cluster can host servers, which makes self-hosting speech-to-text realistic.
- Delivery v1: one email to the owner only. The owner forwards after approval. No distribution logic.
- Agreed shape: code-orchestrated pipeline of single Claude calls with JSON schemas, graphics from a fixed catalog rendered in code, human approval before anything leaves the system. See the design doc.
- Skills every session should consult: `claude-api` for any model or pricing question, `superpowers:brainstorming` before restructuring, `mattpocock-skills:domain-modeling` when terms shift (glossary in `CONTEXT.md`).
- Standing preference: prefer the option that produces a better record over the cheaper one.

## Decisions so far

- [Which speech-to-text handles Norwegian and English well](issues/01-norwegian-stt.md): NB-Whisper large via WhisperX + pyannote, self-hosted on the cluster; Deepgram Nova-3 as fallback. Findings in [research/norwegian-stt.md](research/norwegian-stt.md).
- [Which Python rendering stack](issues/09-render-stack.md): Jinja2 to HTML, WeasyPrint to PDF, headless Playwright with bundled mermaid.js for Mermaid to SVG only, cairosvg for SVG to PNG. Findings in [research/render-stack.md](research/render-stack.md).
- [How are meetings recorded today](issues/03-capture-path.md): phone or laptop recording dropped with a sidecar note into a watched folder on cluster storage.
- [Data handling rules](issues/05-data-handling.md): audio never leaves the cluster; only transcripts go to Anthropic; attendees are told they are recorded.
- [Output language](issues/07-output-language.md): English prose, quotes kept in the original language.

## Not yet specified

- Prompt pack content per stage (system prompts, examples, glossary format). Waits on the sample report reaction and the output-language decision.
- Graphics catalog: which of the six proposed graphic types earn a place for lab meetings, and whether a schedule/Gantt-style view is needed since "scheduled" disagreements are a stated pain. Waits on the sample report.
- Eval scoring: how to judge "the record would have settled the disagreement." Waits on the eval recordings.
- Criteria for eventually sending unattended or to more than the owner. Waits on months of use, not a ticket now.
- Orchestration detail: sidecar note format, folder layout, where the review page lives. Waits on cluster answers.
- Norwegian handling in the LLM stages: whether to normalize Bokmål/Nynorsk, how to treat English terms inside Norwegian speech. Waits on the STT decision and sample transcripts.

## Out of scope

- Meeting bots that join Zoom/Meet/Teams calls. The lab records locally; a bot is a separate effort if ever needed.
- Live captioning or in-meeting summaries. The record is produced after the meeting.
- Precise who-said-what attribution as a quality goal. Best-effort only, per the owner.
- Automatic distribution to attendees or third parties. v1 emails the owner only.
- Slack, Notion, Jira write-back; PowerPoint output; video or slide understanding.
