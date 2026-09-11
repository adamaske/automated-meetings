# Norwegian + English STT with diarization, self-hostable on a GPU cluster

Research date: 2026-09-11. Context: lab records at most ~60 min of meeting audio/week, cost is not a constraint, quality is, and a self-owned GPU cluster is available.

## 1. NB-Whisper (Norwegian National Library / NbAiLab)

- Model family on Hugging Face: `NbAiLab/nb-whisper-{tiny,base,small,medium,large}`, fine-tuned from OpenAI Whisper on Norwegian data (NCC_speech, NST, NPSC). [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- Languages: explicitly "Norwegian, Norwegian Bokmål, Norwegian Nynorsk, English" — trained to transcribe/translate both Bokmål and Nynorsk. [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- Published WER, NB-Whisper Large vs. OpenAI Whisper large-v3, from the paper "Whispering in Norwegian: Navigating Orthographic and Dialectic Challenges" (arXiv:2402.01917):
  - Fleurs (Bokmål): Whisper large-v3 10.4% WER → NB-Whisper Large 6.6% WER.
  - NST (Bokmål): Whisper large-v3 6.8% WER → NB-Whisper Large 2.2% WER.
  - Common Voice (Nynorsk): Whisper large-v3 30.0% WER → NB-Whisper Large 12.6% WER.
  (Note: the ticket asked about NPSC too; NPSC/NST/NCC_speech are used as *training* data per the model card, and the paper's published head-to-head eval numbers found are Fleurs-Bokmål, NST-Bokmål, and Common Voice-Nynorsk, not a separate NPSC test split.) [arXiv:2402.01917](https://arxiv.org/html/2402.01917v1), [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- English: supported (model can transcribe/translate to English), but the paper's English WER numbers on held-out English sets were not found in the abstract/body I retrieved — it's an English-Norwegian bilingual model architecturally (trained in part on data machine-translated to English) rather than a general multilingual model like base Whisper. [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- Mixed-language / code-switching: not explicitly addressed in the model card or paper excerpt.
- Diarization: **not native**. The model card explicitly says speaker diarization requires pairing with an external tool — WhisperX + pyannote-audio. [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- License: **Apache 2.0**, with a note that Norwegian copyright-act attribution requirements still apply for downloads made in Norway. [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- Compute: Large = 1550M params; docs recommend a GPU for Medium/Large (Tiny/Base/Small are CPU-optimized). [model card](https://huggingface.co/NbAiLab/nb-whisper-large)
- Self-hostable: **yes**, standard Hugging Face Transformers / CTranslate2-compatible weights, runs locally.

## 2. faster-whisper / WhisperX with Whisper large-v3 (and large-v3-turbo)

- **faster-whisper**: CTranslate2 reimplementation of Whisper, up to 4x faster than the reference implementation with comparable accuracy and lower memory use; supports large-v3, large-v3-turbo, distil-large-v3, and custom fine-tunes (so NB-Whisper weights can be run through it too). [GitHub](https://github.com/SYSTRAN/faster-whisper)
  - License: **MIT**.
  - GPU: needs cuBLAS/cuDNN for CUDA 12; benchmarked VRAM ~2.9GB (int8) to ~6.1GB (fp16, batch_size=8) on an RTX 3070 Ti-class GPU — trivially fits a research-cluster GPU.
  - Self-hosting: official Docker images provided (`nvidia/cuda:12.3.2-cudnn9-runtime-ubuntu22.04`).
  - No native diarization; commonly paired with WhisperX or `whisper-diarization`.
- **WhisperX**: wraps faster-whisper for batched, word-aligned transcription and integrates **pyannote-audio** for diarization out of the box (`--min_speakers`/`--max_speakers` flags; requires a Hugging Face token + accepting the pyannote gated-model terms). [GitHub](https://github.com/m-bain/whisperX)
  - License: **BSD-2-Clause**.
  - Claims **70x realtime** transcription with Whisper large-v2 via batched inference; "1st place at Ego4d transcription challenge," accepted at INTERSPEECH 2023.
  - VRAM: faster-whisper backend needs <8GB for large-v2 at beam_size=5; can be reduced further with smaller batch size, smaller model, or int8 compute type; CPU-only mode also supported.
- **Base Whisper large-v3** (OpenAI model card, for reference on the underlying ASR the above wrap): 1.55B params, Apache 2.0, trained on 99 languages (Norwegian among them, though not itemized in the excerpt fetched); open-asr-leaderboard mean WER 7.44 across benchmarked languages; OpenAI's own docs note hallucination and accuracy are **worse on lower-resource languages**, which is exactly why the NB-Whisper Norwegian fine-tune outperforms stock large-v3 (6.6% vs 10.4% Fleurs Bokmål, 12.6% vs 30.0% Nynorsk, per above). [model card](https://huggingface.co/openai/whisper-large-v3)
- **pyannote/speaker-diarization-3.1** (the diarization engine WhisperX plugs in): **MIT license**; gated on Hugging Face (requires accepting terms + token, for both this model and its dependency `pyannote/segmentation-3.0`); DER benchmarks range from 7.8% (REPERE) to 11.3% (VoxConverse) up to 50.0% on the hard AVA-AVD set; runs on CPU by default but can be moved to CUDA (`pipeline.to(torch.device("cuda"))`). [model card](https://huggingface.co/pyannote/speaker-diarization-3.1)
- Self-hostable: **yes**, entirely — all components (faster-whisper/WhisperX, pyannote, and NB-Whisper weights) are open weights with permissive-enough licenses (Apache-2.0 / MIT / BSD-2-Clause), runnable on a lab GPU cluster with no per-minute fee.
- Mixed-language/code-switching: Whisper models support language-per-segment behavior but are not specifically evaluated for Bokmål/Nynorsk/English code-switching in the sources checked.

## 3. Deepgram Nova-3

- Norwegian support: **added** — Deepgram announced Nova-3 expansion to include Norwegian (alongside Italian, Turkish, Indonesian), citing Norwegian's "tonal pitch accents and regional variation between Bokmål and Nynorsk" as a specific challenge addressed; claims "double-digit relative WER reductions" vs. Nova-2 for Norwegian, but no absolute WER numbers were published in the announcement I could retrieve. [Deepgram blog](https://deepgram.com/learn/deepgram-expands-nova-3-with-italian-turkish-norwegian-and-indonesian-support)
- English: fully supported (Nova-3 is Deepgram's flagship English/multilingual model).
- Diarization: included at no extra cost for pre-recorded audio ("Detect multiple speakers and label who spoke when," listed as "Included"); streaming diarization pricing wasn't clearly broken out on the pricing page. [pricing](https://deepgram.com/pricing)
- Pricing (pay-as-you-go, pre-recorded/async): Monolingual $0.0043/min, Multilingual $0.0052/min (Growth tier: $0.0036 / $0.0043 per min). Streaming: Monolingual $0.0048/min (was $0.0077), Multilingual $0.0058/min (was $0.0092), with lower Growth-tier rates. [pricing](https://deepgram.com/pricing)
- Self-hostable: **yes, as an exception among the cloud vendors** — Deepgram offers an on-premises/self-hosted deployment. Docs specify a minimum of 1 NVIDIA GPU with 16GB VRAM, 4 CPU cores, 32GB RAM per STT node; only NVIDIA GPUs supported, no MIG/fractional GPUs; newer models (Flux) require Ampere-or-later GPUs (A10/L4/L40S/A100/H100 — T4 not supported), and a single Engine container can scale to up to 8 GPUs. [self-hosted deployment docs](https://developers.deepgram.com/docs/self-hosted-deployment-environments), [self-hosted introduction](https://developers.deepgram.com/docs/self-hosted-introduction)
- License/terms: proprietary commercial product (cloud API or licensed self-hosted deployment), not open-weight.

## 4. AssemblyAI Universal

- Norwegian support: **yes**, but tier-dependent.
  - Universal-3.5 Pro (flagship, 18 languages with native code-switching): includes Norwegian (`no`). [supported languages docs](https://www.assemblyai.com/docs/pre-recorded-audio/supported-languages)
  - Universal-2 (broader, 99 languages): includes both Norwegian (`no`) and Norwegian Nynorsk (`nn`) as distinct codes, placing Norwegian in the "Good accuracy" band (>10–25% WER) rather than the top "High accuracy" band (≤10% WER). [supported languages docs](https://www.assemblyai.com/docs/pre-recorded-audio/supported-languages)
- English: fully supported at top-tier accuracy on all models.
- Mixed-language/code-switching: Universal-3.5 Pro is marketed specifically for "native code-switching" across its 18 supported languages, per AssemblyAI's own multilingual blog post. [blog](https://www.assemblyai.com/blog/multilingual-speech-to-text-api)
- Diarization: **not bundled** — it's a paid add-on on top of transcription: async standard +$0.02/hr, async "experimental" +$0.065/hr, streaming +$0.12/hr (the latter bundles "U3.5 Pro Realtime real-time inline diarization"). [pricing](https://www.assemblyai.com/pricing)
- Pricing: pre-recorded Universal-3.5 Pro $0.21/hr, Universal-2 $0.15/hr; streaming Universal-3.5 Pro Realtime $0.45/hr, Universal-Streaming (English or Multilingual) $0.15/hr; Voice Agent API $0.075/min all-inclusive. [pricing](https://www.assemblyai.com/pricing)
- Self-hostable: **no** — cloud API only; no self-hosting/on-prem option found in AssemblyAI's docs.
- License/terms: proprietary commercial API.

## 5. OpenAI gpt-4o-transcribe-diarize

- What it is: an ASR model with **built-in speaker diarization** ("who spoke when"), available only through OpenAI's Transcription API. [OpenAI docs](https://developers.openai.com/api/docs/models/gpt-4o-transcribe-diarize)
- Language support: **not documented** — neither the model docs page nor secondary sources found specify a language list or explicitly confirm Norwegian support; treat Norwegian coverage as unverified/unknown from primary sources.
- Diarization: native/integrated into the model output, but no technical detail (max speakers, DER benchmarks, language limits on diarization) was published in the docs I could retrieve.
- Pricing: token-based — $2.50 / 1M input (audio) tokens, $10.00 / 1M output tokens; no separate diarization surcharge noted. Max input 16K tokens, max output 2,000 tokens. [OpenAI docs](https://developers.openai.com/api/docs/models/gpt-4o-transcribe-diarize)
- Self-hostable: **no** — cloud API only, proprietary, closed-weight.

## Comparison summary

| Option | Norwegian (Bokmål/Nynorsk) | English | Diarization | Self-hostable | License/cost model |
|---|---|---|---|---|---|
| NB-Whisper large + WhisperX + pyannote | Best measured: 6.6% WER Fleurs-Bokmål, 12.6% WER Nynorsk (Common Voice) | Yes (bilingual training) | Via WhisperX+pyannote | Yes | Apache-2.0 / BSD-2-Clause / MIT — free, GPU compute only |
| faster-whisper/WhisperX + stock large-v3 | Weaker than NB-Whisper (10.4% Fleurs-Bokmål, 30.0% Nynorsk) | Yes, strong (99-lang model) | Via WhisperX+pyannote | Yes | Apache-2.0/MIT/BSD — free, GPU compute only |
| Deepgram Nova-3 | Yes, added recently; no absolute WER published | Yes, flagship-quality | Included (pre-recorded) | Yes (on-prem option, min 16GB VRAM GPU) | Proprietary, $0.0043–0.0052/min cloud, or licensed self-host |
| AssemblyAI Universal | Yes (Universal-2 explicitly lists `nn`); "Good accuracy" band (10–25% WER) | Yes, flagship-quality | Paid add-on | No | Proprietary, $0.15–0.21/hr + diarization add-on |
| OpenAI gpt-4o-transcribe-diarize | Undocumented/unverified | Presumed yes, unverified for this task | Native/built-in | No | Proprietary, token-priced |

## Recommendation

**Primary: NB-Whisper large run through WhisperX (for batching/alignment) with pyannote-audio for diarization, self-hosted on the lab's GPU cluster.**
Rationale: NB-Whisper is the only option with *published, primary-source WER numbers specifically on Norwegian Bokmål and Nynorsk test sets*, and it beats stock Whisper large-v3 by a wide margin on exactly those sets (6.6% vs 10.4% WER Bokmål/Fleurs; 12.6% vs 30.0% WER Nynorsk/Common Voice). It's Apache-2.0/open-weight, so it runs freely on the existing GPU cluster with no per-minute vendor cost and no data leaving the lab — appropriate given "cost is not a constraint, quality is" points toward maximizing accuracy and control, not toward paying for a slightly-less-Norwegian-optimized commercial API. Diarization via pyannote-audio (MIT, DER 7.8–11.3% on good conditions) integrates natively through WhisperX.

**Fallback: Deepgram Nova-3 (cloud or self-hosted on-prem).**
Rationale: it is the one commercial vendor confirmed to support Norwegian with built-in diarization included in pricing, and is the only commercial option that also offers a real self-hosted deployment (min. 16GB-VRAM GPU) if the lab wants to stay on-cluster without running the OSS stack. Use as a fallback if NB-Whisper/WhisperX proves harder to operate reliably (e.g., pyannote gated-model access friction, or if Nynorsk-heavy meetings need a second opinion) — no absolute Norwegian WER was published by Deepgram, so it should be spot-checked against NB-Whisper on real meeting audio before being trusted as primary.

AssemblyAI Universal is not recommended as primary or fallback for this use case: no self-hosting option, diarization costs extra, and its own docs place Norwegian in the "Good" (10–25% WER) rather than "High" accuracy band — worse published/self-reported accuracy tier than NB-Whisper's measured numbers. OpenAI gpt-4o-transcribe-diarize is not recommended: no documented Norwegian language support could be found in primary sources, so Norwegian quality is an unknown risk, and it is closed/cloud-only.

## Realistic turnaround for a 60-minute meeting file (self-hosted primary path)

- WhisperX with batched inference on Whisper large-v2/v3-class models is benchmarked by its authors at **up to ~70x realtime** on a suitable GPU, which would put raw transcription of a 60-minute file at well under 5 minutes of GPU time. [WhisperX GitHub](https://github.com/m-bain/whisperX)
- faster-whisper alone (no batching) is up to 4x faster than reference Whisper with comparable accuracy, i.e. a 60-minute file transcribes in well under realtime (roughly 15 minutes or less) even without WhisperX's batching. [faster-whisper GitHub](https://github.com/SYSTRAN/faster-whisper)
- pyannote diarization adds further processing time on top of ASR but is a comparatively lightweight pass (runs on CPU or GPU); no primary-source runtime benchmark was found for a 60-minute file specifically, so budget for the diarization pass to roughly double end-to-end time in a conservative estimate.
- **Practical estimate**: on a single modern research-cluster GPU (e.g., an A10/A100-class card), a 60-minute meeting file should realistically transcribe + diarize in on the order of **5–20 minutes of wall-clock/GPU time**, i.e. far faster than the weekly ~60-minute recording cadence requires — turnaround is not a bottleneck for this workload at this volume.
