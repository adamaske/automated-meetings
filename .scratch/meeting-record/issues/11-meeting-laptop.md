# What is the meeting laptop's role, OS, and hardware?

Type: grilling
Status: resolved
Blocked by: 

## Question

The lab has a Windows 11 laptop that will be the "meeting laptop": it records meetings and, for now, runs the whole pipeline for demos. It can dual-boot CachyOS. Decide: (a) does the laptop run Linux or Windows for this tool; (b) does the laptop do everything, or does it hand transcription to the cluster GPU; (c) what GPU and RAM does it have (NB-Whisper large wants roughly 10 GB VRAM with WhisperX, or runs slowly on CPU); (d) is it always on and on the lab network, or only present during meetings. The answer reshapes the hosting-shape ticket and may make the cluster optional.

## Answer

Owner decision 2026-09-11: the laptop only records. It runs Windows 11 with no Linux option. Files are transferred by hand to the lab server. The laptop runs no part of the pipeline.
