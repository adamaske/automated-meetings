# Hosted or self-hosted speech-to-text, and which model?

Type: grilling
Status: resolved
Blocked by: 01, 02, 05

## Question

Grilling, informed by the Norwegian STT research, the cluster facts, and the data-handling rules. Decide the primary STT, the diarization method, and where it runs. Record the chosen model, expected turnaround for a 60-minute file, and the fallback.

## Answer

Decision 2026-09-11, following the Norwegian STT research, the data-handling rule, and the lab-server decision: NB-Whisper large through WhisperX with pyannote diarization, running on the lab server GPU. Deepgram Nova-3 remains the documented fallback if the self-hosted stack misbehaves. Audio never leaves the lab server.
