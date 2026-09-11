# Where does each stage run, and how does the owner interact with it?

Type: grilling
Status: resolved
Blocked by: 

## Question

Grilling. Given the cluster facts, the capture path, and the STT decision: does the whole pipeline run as one job on the cluster, or does STT run on the cluster and the rest on a small always-on machine? Where does the review page live? How is the email sent from that environment? The answer finalizes the deployment section of the design doc and completes the map.

## Amendment 2026-09-11

Narrowed: everything runs on the Windows 11 lab server. Two decisions remain. (1) Runtime: native Windows Python, or WSL2 with CUDA passthrough so the Linux toolchain (WhisperX, pyannote, WeasyPrint) is used unchanged. (2) Trigger and review interface: a CLI command, or a small local web page on the server where the owner registers a concluded meeting, uploads the file, starts the run, and later reviews and approves the Record.

## Answer

Owner decisions 2026-09-11:

- Everything runs on the lab server: Windows 11 with an RTX 3060 12 GB. Runtime is WSL2 (Ubuntu) with NVIDIA CUDA passthrough, so the Linux toolchain (WhisperX, pyannote, WeasyPrint, Playwright) is used unchanged.
- Trigger and review are one local web page served from WSL2 on the server, opened in a browser on the lab network: register a concluded meeting (title, date, attendees), pick the audio file, Run; later the same page shows the Record, verification flags, and Approve, which sends the Digest to the owner.
- Storage: a job directory per meeting on the server's disk, inside WSL2, not on the Windows side, for speed and file-permission sanity.
- 12 GB VRAM note: NB-Whisper large in float16 fits, but run WhisperX with a reduced batch size (8 or less) and let it unload the ASR model before pyannote diarization, which WhisperX does by default. If memory errors appear, fall back to int8 compute or NB-Whisper medium. Expect roughly 10 to 20 minutes per meeting hour on this card.
- Email from the server: to be confirmed at setup time. Default proposal is SMTP with a Gmail app password to the owner's address, since there is a single recipient.
