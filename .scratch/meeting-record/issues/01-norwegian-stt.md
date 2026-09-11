# Which speech-to-text handles Norwegian and English well, with diarization, and can it run on the cluster?

Type: research
Status: resolved
Blocked by: 

## Question

Compare, from primary sources: NB-Whisper (Norwegian National Library, Hugging Face), faster-whisper/WhisperX with large-v3, Deepgram Nova-3, AssemblyAI Universal, OpenAI gpt-4o-transcribe-diarize. For each: Norwegian (Bokmål and Nynorsk) support and any published WER, English support, mixed-language behavior, speaker diarization availability, pricing or GPU requirements, and whether it can be self-hosted. Recommend a primary and a fallback for a lab that can run GPU jobs on its own cluster and does not care about cost. Write findings to research/norwegian-stt.md with sources.

## Answer

Recommended primary: **NB-Whisper large (NbAiLab, Apache-2.0) run through WhisperX + pyannote-audio, self-hosted on the lab GPU cluster.** It's the only option with published, primary-source WER on Norwegian test sets, and it clearly beats stock Whisper large-v3 on Norwegian: 6.6% vs 10.4% WER on Fleurs (Bokmål) and 12.6% vs 30.0% WER on Common Voice (Nynorsk). English is supported (bilingual training), diarization comes via pyannote-audio (MIT, DER 7.8–11.3% in good conditions), and the whole stack (Apache-2.0/MIT/BSD-2-Clause) runs free on-cluster with no per-minute cost or data leaving the lab.

Recommended fallback: **Deepgram Nova-3**, cloud or self-hosted (min. 16GB-VRAM NVIDIA GPU documented for on-prem). It's the only commercial vendor confirmed to support Norwegian with diarization included in pricing and a real self-host option — useful if the OSS stack proves fragile (e.g., pyannote gated-model access) or as a cross-check on Nynorsk-heavy recordings, though Deepgram hasn't published absolute Norwegian WER.

AssemblyAI Universal (no self-hosting, diarization is a paid add-on, Norwegian only in the "10–25% WER" accuracy band) and OpenAI gpt-4o-transcribe-diarize (no documented Norwegian support found, closed/cloud-only) are not recommended.

Turnaround for a 60-minute meeting file on the self-hosted path: WhisperX's batched inference is benchmarked at up to ~70x realtime, so raw ASR should take well under 5 minutes of GPU time; with diarization added, a conservative estimate is roughly 5–20 minutes of wall-clock time on a single research-cluster GPU — not a bottleneck at ~60 min/week volume.

Full findings with sources: [research/norwegian-stt.md](../research/norwegian-stt.md)
