# How are meetings recorded today, and how does a recording reach the tool?

Type: grilling
Status: resolved
Blocked by: 

## Question

Grilling. Is there a recording today at all (phone voice memo, laptop, a room mic, Zoom/Teams cloud recording)? What formats result? Who presses record and who uploads? Is a watched folder on the cluster storage, a CLI command, or a simple upload page the least-friction path for the lab? The answer fixes the ingest stage and rules the metadata source (attendee list, meeting title, date) in or out.

## Answer

Owner decision 2026-09-11: meetings are recorded on a phone or laptop. The file plus a small sidecar note (title, date, optional attendees) is dropped into a watched folder on cluster storage. No upload page or CLI in v1. Ingest stage = folder watcher that starts a Job when an audio file and its note both exist.

## Amendment 2026-09-11

No watched folder. Recording on the laptop, manual transfer to the lab server, then the owner explicitly declares the meeting concluded through an interface and starts the pipeline. The interface type is decided in the hosting-shape ticket.
