# What is the meeting laptop's role, OS, and hardware?

Type: grilling
Status: open
Blocked by: 

## Question

The lab has a Windows 11 laptop that will be the "meeting laptop": it records meetings and, for now, runs the whole pipeline for demos. It can dual-boot CachyOS. Decide: (a) does the laptop run Linux or Windows for this tool; (b) does the laptop do everything, or does it hand transcription to the cluster GPU; (c) what GPU and RAM does it have (NB-Whisper large wants roughly 10 GB VRAM with WhisperX, or runs slowly on CPU); (d) is it always on and on the lab network, or only present during meetings. The answer reshapes the hosting-shape ticket and may make the cluster optional.
