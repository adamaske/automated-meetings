# Meeting Record

A lab tool that turns a recording of a meeting into a written record of what was agreed, planned, and scheduled, so later disagreements can be settled by reading instead of remembering.

## Language

**Meeting**:
One recorded gathering of lab members, in English, Norwegian, or both. Identified by title and start time.
_Avoid_: Call, session

**Recording**:
The audio (or video, stripped to audio) of one Meeting as it arrived. The raw input; never edited.
_Avoid_: File, upload

**Job**:
One run of the pipeline for one Recording, with all its intermediate and final artifacts kept together.
_Avoid_: Run, task, pipeline instance

**Transcript**:
Text of the Recording as segments with timestamps and speaker labels. The raw Transcript is what speech-to-text emitted; the clean Transcript has filler removed and speakers named where possible.
_Avoid_: Dictation, captions

**Speaker label**:
The anonymous tag ("Speaker 2") diarization assigns to a voice. Mapped to a name only when the evidence is good; otherwise left as is.
_Avoid_: Speaker ID, voice

**Extraction**:
The structured content of a Meeting pulled from the clean Transcript: Decisions, Action items, Schedule points, Open questions, Data points. Every item cites the Transcript segments it came from.
_Avoid_: Summary, analysis, notes

**Decision**:
Something the Meeting settled. Distinct from a proposal or a preference someone voiced.
_Avoid_: Agreement, outcome, conclusion

**Action item**:
A task someone took on in the Meeting, with an owner if one was named and a due date if one was stated.
_Avoid_: Todo, task, follow-up

**Schedule point**:
A date, deadline, or time window stated in the Meeting. Kept separate from Action items because scheduling disputes are the main reason this tool exists.
_Avoid_: Deadline, milestone, timeline

**Open question**:
Something raised in the Meeting and not settled by its end.
_Avoid_: Issue, todo

**Segment reference**:
A pointer from an Extraction item to the Transcript segments that support it. An item without one is not allowed into the Record.
_Avoid_: Citation, source, evidence

**Record**:
The final document for one Meeting: prose sections plus Graphics, in HTML and PDF. The thing the lab reads to settle a disagreement.
_Avoid_: Report, summary, minutes

**Graphic**:
A chart or diagram in the Record, drawn by code from Extraction data using one of a fixed set of Graphic types. Never free-drawn by the model, except a single optional Mermaid diagram.
_Avoid_: Image, figure, visualization

**Graphics catalog**:
The fixed set of Graphic types the Record may use.
_Avoid_: Templates, chart library

**Verification flag**:
A note from the verify stage that a claim in the Record is unsupported by, or contradicts, the clean Transcript.
_Avoid_: Warning, error, issue

**Digest**:
The email carrying the Record to the owner for approval. In v1 it goes to the owner only.
_Avoid_: Notification, summary email

**Owner**:
The one person who receives the Digest, approves it, and forwards it. Also the person who administers the tool.
_Avoid_: Admin, user, recipient

**Glossary (lab)**:
The lab-maintained list of names, project names, and acronyms used to correct the Transcript. Not this file.
_Avoid_: Dictionary, vocabulary
