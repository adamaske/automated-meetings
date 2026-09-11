# What are the lab's rules for recording consent, storage, retention, and sending audio to third parties?

Type: grilling
Status: resolved
Blocked by: 

## Question

Grilling. Does the lab need explicit consent from attendees to record and transcribe? Where may audio and transcripts be stored, and for how long? Is sending audio to a hosted STT vendor acceptable, or must transcription stay on the cluster? Is sending transcripts to the Anthropic API acceptable given its default 30-day retention? Does the institution have a data-processing agreement with any vendor already? The answers gate ticket 06 and ticket 08.

## Answer

Owner decision 2026-09-11: audio never leaves the cluster. Transcription runs self-hosted (NB-Whisper via WhisperX, per ticket 01). Only the text transcript is sent to the Anthropic API. Attendees are told at the start that the meeting is recorded for the record. Retention of audio on the cluster to be set in the hosting-shape ticket (default proposal: delete audio 30 days after the Record is approved, keep transcript and Record).
