# Map: Meeting record pipeline (wayfinder:map)

Effort slug: `meeting-record`. Tickets live in `issues/`. Research findings land in `research/`.

## Destination

An implementation-ready v1 spec for a lab tool that turns a meeting recording (English or Norwegian) into a trusted written record of what was agreed, planned, and scheduled, and emails it to the owner for approval and forwarding. Every vendor, model, hosting, and data-handling decision is made; the design doc at `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md` is updated to match; the next step after the map is `superpowers:writing-plans` for build step 1.

## Status

Destination reached 2026-09-11. Every vendor, model, hosting, and data-handling decision is made and the design doc matches. Build step 1 plan: `docs/superpowers/plans/2026-09-11-build-step-1-transcription.md`.

## Notes

- Domain: internal research-lab meetings, 2 to 10 people, English or Norwegian, sometimes mixed. No written record exists today, which causes disagreements later. The record's job is to settle those.
- Content over attribution: what was agreed matters, who said it is secondary. Speaker-name mapping is best-effort, never a blocker.
- Volume: at most about 60 minutes of meeting audio per week. Cost is not a constraint if the output is good. The lab server has a strong GPU, which makes self-hosting speech-to-text realistic.
- A lab-owned Windows 11 laptop records meetings. The pipeline runs on the lab server (Windows 11 only, strong NVIDIA GPU). Files move by hand; the owner triggers each run explicitly. No cluster in v1. Public repo: https://github.com/adamaske/automated-meetings
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
- [Meeting laptop role](issues/11-meeting-laptop.md): records only; Windows 11; files transferred by hand.
- [Where the pipeline runs](issues/02-cluster-capabilities.md): the lab server, Windows 11 only, strong NVIDIA GPU. Cluster dropped from v1.
- [Capture path, amended](issues/03-capture-path.md): no watched folder; the owner declares a meeting concluded through an interface and starts the run.
- [Speech-to-text decision](issues/06-stt-decision.md): NB-Whisper large via WhisperX + pyannote on the lab server GPU; Deepgram Nova-3 fallback.
- [Hosting shape](issues/10-hosting-shape.md): lab server, Windows 11 + RTX 3060 12 GB, WSL2 with CUDA; one local web page for trigger and review; WhisperX batch size reduced for 12 GB.
- [What a Record looks like](issues/04-sample-report.md): A's header and flag box, C's topic timeline and chronological spine first, then A's reference sections. Graphics catalog v1 is topic timeline, dates chart, action table, figures table. Prototype: https://claude.ai/code/artifact/bd2c6e06-5dd0-47fe-bdcd-88cd551eb2c8

## Not yet specified

- Prompt pack content per stage (system prompts, examples, glossary format). Now specifiable; belongs to the implementation plan, not the map.
- Eval scoring: how to judge "the record would have settled the disagreement." Waits on the eval recordings.
- Criteria for eventually sending unattended or to more than the owner. Waits on months of use, not a ticket now.
- Web page detail: exact fields on the meeting form and how attendees are entered. Small enough for the implementation plan.
- Norwegian handling in the LLM stages: whether to normalize Bokmål/Nynorsk, how to treat English terms inside Norwegian speech. Waits on the first real transcripts from the eval recordings.

## Out of scope

- Meeting bots that join Zoom/Meet/Teams calls. The lab records locally; a bot is a separate effort if ever needed.
- Live captioning or in-meeting summaries. The record is produced after the meeting.
- Precise who-said-what attribution as a quality goal. Best-effort only, per the owner.
- Automatic distribution to attendees or third parties. v1 emails the owner only.
- Slack, Notion, Jira write-back; PowerPoint output; video or slide understanding.
