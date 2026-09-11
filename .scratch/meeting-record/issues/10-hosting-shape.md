# Where does each stage run, and how does the owner interact with it?

Type: grilling
Status: open
Blocked by: 

## Question

Grilling. Given the cluster facts, the capture path, and the STT decision: does the whole pipeline run as one job on the cluster, or does STT run on the cluster and the rest on a small always-on machine? Where does the review page live? How is the email sent from that environment? The answer finalizes the deployment section of the design doc and completes the map.

## Amendment 2026-09-11

Narrowed: everything runs on the Windows 11 lab server. Two decisions remain. (1) Runtime: native Windows Python, or WSL2 with CUDA passthrough so the Linux toolchain (WhisperX, pyannote, WeasyPrint) is used unchanged. (2) Trigger and review interface: a CLI command, or a small local web page on the server where the owner registers a concluded meeting, uploads the file, starts the run, and later reviews and approves the Record.
