# automated-meetings

A lab tool that turns a recording of a meeting (English or Norwegian) into a written record of what was agreed, planned, and scheduled, with graphics rendered as code, and emails it to the owner for approval.

Status: design phase. Nothing runs yet.

- Design: `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md`
- Glossary: `CONTEXT.md`
- Open decisions (wayfinder map): `.scratch/meeting-record/map.md`
- Research notes: `.scratch/meeting-record/research/`

Planned shape: self-hosted NB-Whisper transcription with diarization, Claude for cleanup, extraction, and composition against JSON schemas, a fixed graphics catalog rendered by code to HTML/PDF/SVG, human approval before any email leaves.

Meeting recordings, transcripts, and records are never committed to this repository.
