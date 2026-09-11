# Build Step 1: Lab Server Setup and Diarized Transcription Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On the Windows 11 lab server, one elevated setup script prepares WSL2 with CUDA, and the owner can hand the pipeline a Recording and get back a diarized Transcript in a Job directory.

**Architecture:** A small Python package (`meeting_record`) owns the Job directory layout, ingest, a normalized Transcript format, a stage orchestrator, and a CLI. The only module that touches the GPU stack is `whisperx_engine.py`, which runs NB-Whisper large through WhisperX, aligns words, and diarizes with pyannote, then hands WhisperX's output to a pure normalization function so vendor types never leave that module. Setup is two scripts: `scripts/windows/setup.ps1` (elevated PowerShell, winget, WSL, NVIDIA driver check) calls `scripts/wsl/bootstrap.sh` (root inside Ubuntu: packages, app user, uv, code, models).

**Tech Stack:** Python 3.12, uv, pydantic 2, pytest, WhisperX 3.8.6 (faster-whisper/CTranslate2, pyannote-audio 4, torch 2.8 with CUDA 12.8 wheels), ffmpeg, Bash, Windows PowerShell 5.1, winget, WSL2 Ubuntu 24.04.

**Spec:** `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md` (section 1 decisions, section 3 job layout and stages 1–2, section 10 build step 1). Decisions behind it: `.scratch/meeting-record/map.md`, tickets 05, 06, 10 in `.scratch/meeting-record/issues/`. Glossary: `CONTEXT.md`.

## Global Constraints

- Language: Python. Pin `requires-python = ">=3.12,<3.13"`.
- Runs on the lab server: Windows 11, RTX 3060 12 GB, inside WSL2 (Ubuntu 24.04) with CUDA. The Meeting laptop only records; files are moved by hand. No cluster, no watched folder.
- Speech-to-text: NB-Whisper large (`NbAiLab/nb-whisper-large`) through WhisperX with pyannote diarization (`pyannote/speaker-diarization-community-1`) on the server GPU. WhisperX batch size 8 or less. Fall back to `int8` compute or NB-Whisper medium if memory runs short. Unload each model before loading the next.
- Audio never leaves the lab server. Only model downloads go out.
- Job directories live on the WSL2 ext4 disk, never under `/mnt/c`.
- Meeting Recordings, Transcripts, and Records are never committed. Test media is generated or downloaded at test time.
- Every stage reads artifacts from the Job directory and writes one artifact. Stages are idempotent: re-running a stage overwrites only its own output. Outputs are written atomically so a crash never leaves a half-written output that looks finished.
- The normalized Transcript format is the vendor seam: no WhisperX, faster-whisper, or pyannote type or dict shape appears outside `whisperx_engine.py` and `transcript.normalize_segments`.
- Segment IDs are stable strings `s0001`, `s0002`, … because later stages cite them as Segment references.
- Use the glossary terms from `CONTEXT.md` in code, docs, and CLI text: Meeting, Recording, Job, Transcript, Speaker label.
- NVIDIA driver on Windows must be 570.65 or newer (CUDA 12.8 GA minimum for Windows). No NVIDIA driver is installed inside WSL.

## Facts verified on 2026-09-11 (so the executor does not re-research)

- `NbAiLab/nb-whisper-large` has CTranslate2 weights at the repo root (`model.bin` 6.17 GB float16, `config.json` with `alignment_heads`, `tokenizer.json`, `preprocessor_config.json` with `feature_size: 128`). `whisperx.load_model("NbAiLab/nb-whisper-large", ...)` downloads exactly those files through faster-whisper.
- WhisperX 3.8.6 on PyPI: `requires_python <3.14,>=3.10`, depends on `torch~=2.8.0`, `pyannote-audio>=4.0.0`, `ctranslate2>=4.5.0`, `huggingface-hub<1.0.0`. Resolves on Linux x86_64 to torch 2.8.0, ctranslate2 4.8.2, pyannote-audio 4.0.7, nvidia-cudnn-cu12 9.10.2.21, nvidia-cublas-cu12 12.8.4.1.
- WhisperX API: `whisperx.load_model(whisper_arch, device, compute_type=..., language=...)`, `model.transcribe(audio, batch_size=..., language=...)` returns `{"segments", "language"}`, `whisperx.load_align_model(language_code=..., device=...)`, `whisperx.align(segments, model, metadata, audio, device, return_char_alignments=False)`, `from whisperx.diarize import DiarizationPipeline`, `DiarizationPipeline(model_name=None, token=None, device=None)`, `pipeline(audio, num_speakers=None, min_speakers=None, max_speakers=None)` returns a DataFrame, `whisperx.assign_word_speakers(diarize_df, result)` adds `speaker` to segments and words.
- WhisperX default alignment models: `no` → `NbAiLab/nb-wav2vec2-1b-bokmaal-v2`, `nn` → `NbAiLab/nb-wav2vec2-1b-nynorsk` (both not gated), `en` → torchaudio `WAV2VEC2_ASR_BASE_960H`.
- `pyannote/speaker-diarization-community-1` is gated: the owner must accept its terms on huggingface.co and use a read token from the same account.
- ctranslate2 does not find the pip-installed cuDNN and cuBLAS by itself. faster-whisper's documented fix is to put `site-packages/nvidia/*/lib` on `LD_LIBRARY_PATH` before Python starts. That is what `bin/env.sh` does.
- winget has `Microsoft.WSL` but no NVIDIA driver package. The setup script can only check the driver.
- Sample Norwegian audio for the GPU smoke test: `https://github.com/NbAiLab/nb-whisper/raw/main/audio/knuthamsun.mp3` (22 MB, single speaker).

## Known risks carried into this step

- **Mixed-language Meetings.** The language is forced per Meeting. With `language=no`, English stretches may come out translated into Norwegian instead of transcribed. Task 10 spot-checks this on a real mixed Recording and records the result; it does not fix it.
- **First launch of a `--no-launch` distro.** If `wsl --install --no-launch` leaves Ubuntu unregistered on this Windows build, the setup script stops with a one-line manual workaround. Task 10 records which way it went.

## Deviations from the spec, on purpose

- `transcript.raw.txt` is added next to `transcript.raw.json` so the demo is readable without tooling.
- A Recording with a video stream is stored as `audio.flac` (mono 16 kHz); audio-only Recordings are copied unchanged as `audio.<ext>`.
- Retries on transient failures are left out: the only stage is local GPU work, which should fail fast. Retries arrive with the first API-calling stage in build step 2.
- Delete-job and retention sweep are left out: the data-handling rule deletes audio 30 days after approval, and approval arrives in build step 4.

## File Structure

```
pyproject.toml                        project, extras (gpu), dev group, pytest config
uv.lock                               generated by uv
.python-version                       3.12
.gitattributes                        LF for .sh, CRLF for .ps1
.env.example                          documented settings; real .env is gitignored
bin/env.sh                            sourceable: loads .env, puts pip CUDA libs on LD_LIBRARY_PATH
bin/mr                                wrapper: sources env.sh, execs the CLI
src/meeting_record/__init__.py
src/meeting_record/config.py          Settings.from_env
src/meeting_record/io.py              atomic text/JSON writes
src/meeting_record/meeting.py         Language, Attendee, RecordingInfo, MeetingMeta
src/meeting_record/jobs.py            Job directory names, slugify, new_job_dir, Job, open_job, log.jsonl
src/meeting_record/ingest.py          ffprobe/ffmpeg, create_job
src/meeting_record/transcript.py      Word, Segment, EngineInfo, RawTranscript, normalize_segments, render_text
src/meeting_record/transcribe.py      Transcriber protocol, TranscriptionResult, transcribe_stage
src/meeting_record/orchestrator.py    Stage, StageOutcome, StageFailed, run_job
src/meeting_record/whisperx_engine.py WhisperXTranscriber, fetch_models (only GPU-stack importer)
src/meeting_record/doctor.py          environment checks and report
src/meeting_record/cli.py             mr new / run / show / doctor / fetch-models
scripts/wsl/bootstrap.sh              root inside WSL: packages, user, uv, code, .env, GPU check, models
scripts/windows/setup.ps1             elevated PowerShell: preflight, driver, winget, WSL, distro, bootstrap
tests/conftest.py                     make_job, media, fake_transcriber fixtures
tests/test_config.py
tests/test_meeting.py
tests/test_jobs.py
tests/test_ingest.py
tests/test_transcript.py
tests/test_transcribe_stage.py
tests/test_orchestrator.py
tests/test_whisperx_engine.py
tests/test_gpu_smoke.py               opt-in, lab server only
tests/test_doctor.py
tests/test_cli.py
tests/test_bin_wrapper.py
tests/test_setup_scripts.py
README.md                             setup and demo instructions
```

The Job directory after this step:

```
~/meeting-record/jobs/<YYYY-MM-DD>-<title-slug>/
  meeting.json          MeetingMeta
  audio.<ext>           the Recording as it arrived (audio.flac if it had video)
  transcript.raw.json   RawTranscript
  transcript.raw.txt    one line per segment: [HH:MM:SS] SPEAKER_00: text
  log.jsonl             one JSON object per stage event: ts, stage, status, duration_s, details or error
```

---

### Task 1: Project scaffold and settings

**Files:**
- Create: `pyproject.toml`, `.python-version`, `.gitattributes`, `src/meeting_record/__init__.py`, `src/meeting_record/config.py`
- Test: `tests/test_config.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `meeting_record.config.Settings`, a frozen dataclass with fields `jobs_dir: Path`, `hf_token: str | None`, `whisper_model: str`, `compute_type: str`, `batch_size: int`, `device: str`, `diarization_model: str`, and `Settings.from_env(env: Mapping[str, str] | None = None) -> Settings`. Environment variables: `MR_JOBS_DIR`, `HF_TOKEN`, `MR_WHISPER_MODEL`, `MR_COMPUTE_TYPE`, `MR_BATCH_SIZE`, `MR_DEVICE`, `MR_DIARIZATION_MODEL`.

- [ ] **Step 1: Create the project files**

`pyproject.toml`:

```toml
[project]
name = "meeting-record"
version = "0.1.0"
description = "Turns a meeting Recording into a written Record."
requires-python = ">=3.12,<3.13"
dependencies = ["pydantic>=2.9"]

[project.optional-dependencies]
gpu = ["whisperx==3.8.6"]

[project.scripts]
meeting-record = "meeting_record.cli:main"

[dependency-groups]
dev = ["pytest>=8"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/meeting_record"]

[tool.uv]
# Both the dev machine and the lab server (WSL2) are Linux x86_64.
environments = ["sys_platform == 'linux' and platform_machine == 'x86_64'"]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = ["gpu: needs the gpu extra, a CUDA GPU, and HF_TOKEN; run with MR_GPU_TESTS=1 on the lab server"]
```

`.python-version`:

```
3.12
```

`.gitattributes`:

```
* text=auto
*.sh text eol=lf
bin/mr text eol=lf
*.ps1 text eol=crlf
```

`src/meeting_record/__init__.py`:

```python
"""Meeting Record: turns a meeting Recording into a written Record."""
```

The `cli.py` entry point named in `pyproject.toml` is created in Task 7. Until then the `meeting-record` script exists but fails when run; nothing runs it before Task 7.

- [ ] **Step 2: Lock and install**

Run: `uv sync`
Expected: creates `.venv` and `uv.lock`. The lock resolves the `gpu` extra too (torch, whisperx), but `uv sync` without `--extra gpu` does not install it. Resolution can take a minute.

- [ ] **Step 3: Write the failing test**

`tests/test_config.py`:

```python
from pathlib import Path

import pytest

from meeting_record.config import Settings


def test_defaults():
    s = Settings.from_env({})
    assert s.jobs_dir == Path.home() / "meeting-record" / "jobs"
    assert s.hf_token is None
    assert s.whisper_model == "NbAiLab/nb-whisper-large"
    assert s.compute_type == "float16"
    assert s.batch_size == 8
    assert s.device == "cuda"
    assert s.diarization_model == "pyannote/speaker-diarization-community-1"


def test_overrides(tmp_path):
    s = Settings.from_env(
        {
            "MR_JOBS_DIR": str(tmp_path),
            "HF_TOKEN": "hf_abc",
            "MR_WHISPER_MODEL": "NbAiLab/nb-whisper-medium",
            "MR_COMPUTE_TYPE": "int8",
            "MR_BATCH_SIZE": "4",
            "MR_DEVICE": "cpu",
            "MR_DIARIZATION_MODEL": "pyannote/other",
        }
    )
    assert s.jobs_dir == tmp_path
    assert s.hf_token == "hf_abc"
    assert s.whisper_model == "NbAiLab/nb-whisper-medium"
    assert s.compute_type == "int8"
    assert s.batch_size == 4
    assert s.device == "cpu"
    assert s.diarization_model == "pyannote/other"


def test_empty_hf_token_counts_as_unset():
    assert Settings.from_env({"HF_TOKEN": ""}).hf_token is None


def test_jobs_dir_expands_home():
    assert Settings.from_env({"MR_JOBS_DIR": "~/jobs"}).jobs_dir == Path.home() / "jobs"


def test_non_integer_batch_size_is_rejected():
    with pytest.raises(ValueError):
        Settings.from_env({"MR_BATCH_SIZE": "eight"})
```

- [ ] **Step 4: Run test to verify it fails**

Run: `uv run pytest tests/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'meeting_record.config'`

- [ ] **Step 5: Write the implementation**

`src/meeting_record/config.py`:

```python
"""Settings read from the environment (bin/env.sh loads .env into it)."""

from __future__ import annotations

import os
from collections.abc import Mapping
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class Settings:
    jobs_dir: Path
    hf_token: str | None
    whisper_model: str
    compute_type: str
    batch_size: int
    device: str
    diarization_model: str

    @classmethod
    def from_env(cls, env: Mapping[str, str] | None = None) -> Settings:
        env = os.environ if env is None else env
        default_jobs = Path.home() / "meeting-record" / "jobs"
        return cls(
            jobs_dir=Path(env.get("MR_JOBS_DIR", str(default_jobs))).expanduser(),
            hf_token=env.get("HF_TOKEN") or None,
            whisper_model=env.get("MR_WHISPER_MODEL", "NbAiLab/nb-whisper-large"),
            compute_type=env.get("MR_COMPUTE_TYPE", "float16"),
            batch_size=int(env.get("MR_BATCH_SIZE", "8")),
            device=env.get("MR_DEVICE", "cuda"),
            diarization_model=env.get(
                "MR_DIARIZATION_MODEL", "pyannote/speaker-diarization-community-1"
            ),
        )
```

- [ ] **Step 6: Run test to verify it passes**

Run: `uv run pytest tests/test_config.py -v`
Expected: 5 passed

- [ ] **Step 7: Commit**

```bash
git add pyproject.toml uv.lock .python-version .gitattributes src/meeting_record/__init__.py src/meeting_record/config.py tests/test_config.py
git commit -m "Scaffold meeting_record package with settings from environment"
```

---

### Task 2: Meeting metadata and the Job directory

**Files:**
- Create: `src/meeting_record/io.py`, `src/meeting_record/meeting.py`, `src/meeting_record/jobs.py`, `tests/conftest.py`
- Test: `tests/test_meeting.py`, `tests/test_jobs.py`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `meeting_record.io.write_text_atomic(path: Path, text: str) -> None`
  - `meeting_record.io.write_json_atomic(path: Path, data: BaseModel | dict) -> None`
  - `meeting_record.meeting.Language = Literal["no", "nn", "en"]`
  - `meeting_record.meeting.Attendee(name: str, email: str | None = None)` with `Attendee.parse(raw: str) -> Attendee` accepting `"Name <email>"` or `"Name"`
  - `meeting_record.meeting.RecordingInfo(original_name: str, file: str, sha256: str, duration_s: float, had_video: bool)`
  - `meeting_record.meeting.MeetingMeta(schema_version=1, meeting_id: str, title: str, date: dt.date, language: Language, attendees: list[Attendee] = [], min_speakers: int | None = None, max_speakers: int | None = None, recording: RecordingInfo, created_at: dt.datetime)`
  - `meeting_record.jobs` constants `MEETING_FILE = "meeting.json"`, `RAW_TRANSCRIPT_FILE = "transcript.raw.json"`, `RAW_TRANSCRIPT_TEXT_FILE = "transcript.raw.txt"`, `LOG_FILE = "log.jsonl"`
  - `meeting_record.jobs.slugify(title: str) -> str`
  - `meeting_record.jobs.new_job_dir(jobs_root: Path, date: dt.date, title: str) -> Path`
  - `meeting_record.jobs.Job(dir: Path)` with `.id -> str`, `.path(name: str) -> Path`, `.meta() -> MeetingMeta`, `.append_log(stage: str, status: str, **fields) -> None`, `.read_log() -> list[dict]`
  - `meeting_record.jobs.JobNotFound(Exception)`, `meeting_record.jobs.open_job(jobs_root: Path, job_id: str) -> Job`
  - pytest fixture `make_job(job_id="2026-09-10-lab-meeting", language="no", min_speakers=None, max_speakers=None) -> Job` in `tests/conftest.py`, which writes `meeting.json` and a dummy `audio.wav` (duration 4.0 s) under `tmp_path / "jobs"`.

- [ ] **Step 1: Write the failing tests**

`tests/conftest.py`:

```python
import datetime as dt

import pytest

from meeting_record.io import write_json_atomic
from meeting_record.jobs import MEETING_FILE, Job
from meeting_record.meeting import MeetingMeta, RecordingInfo


@pytest.fixture
def make_job(tmp_path):
    """Build a registered Job without ffmpeg: meeting.json plus a dummy audio file."""

    def _make(job_id="2026-09-10-lab-meeting", language="no", min_speakers=None, max_speakers=None):
        job_dir = tmp_path / "jobs" / job_id
        job_dir.mkdir(parents=True)
        (job_dir / "audio.wav").write_bytes(b"not really audio")
        meta = MeetingMeta(
            meeting_id=job_id,
            title="Lab meeting",
            date=dt.date(2026, 9, 10),
            language=language,
            min_speakers=min_speakers,
            max_speakers=max_speakers,
            recording=RecordingInfo(
                original_name="rec.wav",
                file="audio.wav",
                sha256="0" * 64,
                duration_s=4.0,
                had_video=False,
            ),
            created_at=dt.datetime(2026, 9, 10, 12, 0, tzinfo=dt.timezone.utc),
        )
        write_json_atomic(job_dir / MEETING_FILE, meta)
        return Job(job_dir)

    return _make
```

`tests/test_meeting.py`:

```python
import datetime as dt

import pytest
from pydantic import ValidationError

from meeting_record.io import write_json_atomic
from meeting_record.meeting import Attendee, MeetingMeta, RecordingInfo


def _meta(**overrides):
    fields = dict(
        meeting_id="2026-09-10-lab-mote",
        title="Lab-møte",
        date=dt.date(2026, 9, 10),
        language="no",
        recording=RecordingInfo(
            original_name="opptak.m4a", file="audio.m4a", sha256="a" * 64, duration_s=60.0, had_video=False
        ),
        created_at=dt.datetime(2026, 9, 10, 12, 0, tzinfo=dt.timezone.utc),
    )
    fields.update(overrides)
    return MeetingMeta(**fields)


def test_attendee_parse_name_and_email():
    assert Attendee.parse(" Kari Nordmann <kari@example.org> ") == Attendee(
        name="Kari Nordmann", email="kari@example.org"
    )


def test_attendee_parse_name_only():
    assert Attendee.parse("Ola") == Attendee(name="Ola", email=None)


def test_attendee_parse_rejects_empty():
    with pytest.raises(ValidationError):
        Attendee.parse("   ")


def test_meeting_meta_rejects_inverted_speaker_bounds():
    with pytest.raises(ValidationError, match="min_speakers"):
        _meta(min_speakers=4, max_speakers=2)


def test_meeting_meta_rejects_unknown_language():
    with pytest.raises(ValidationError):
        _meta(language="sv")


def test_meeting_meta_round_trips_json_keeping_norwegian_letters(tmp_path):
    path = tmp_path / "meeting.json"
    meta = _meta(attendees=[Attendee(name="Åse")])
    write_json_atomic(path, meta)
    raw = path.read_text(encoding="utf-8")
    assert "Lab-møte" in raw
    assert MeetingMeta.model_validate_json(raw) == meta


def test_write_json_atomic_leaves_no_temp_file(tmp_path):
    write_json_atomic(tmp_path / "x.json", {"a": "ø"})
    assert sorted(p.name for p in tmp_path.iterdir()) == ["x.json"]
    assert (tmp_path / "x.json").read_text(encoding="utf-8") == '{\n  "a": "ø"\n}\n'
```

`tests/test_jobs.py`:

```python
import datetime as dt

import pytest

from meeting_record.jobs import JobNotFound, new_job_dir, open_job, slugify


def test_slugify_transliterates_norwegian_letters():
    assert slugify("Møte om Ærlig budsjett!") == "mote-om-aerlig-budsjett"


def test_slugify_strips_accents():
    assert slugify("Café planning") == "cafe-planning"


def test_slugify_falls_back_when_nothing_is_left():
    assert slugify("!!!") == "meeting"


def test_slugify_caps_length():
    assert len(slugify("a" * 100)) == 60


def test_new_job_dir_uses_date_and_slug_and_avoids_collisions(tmp_path):
    first = new_job_dir(tmp_path, dt.date(2026, 9, 10), "Lab meeting")
    second = new_job_dir(tmp_path, dt.date(2026, 9, 10), "Lab meeting")
    assert first.name == "2026-09-10-lab-meeting"
    assert second.name == "2026-09-10-lab-meeting-2"
    assert first.is_dir() and second.is_dir()


def test_open_job_finds_a_registered_job(make_job):
    job = make_job()
    opened = open_job(job.dir.parent, job.id)
    assert opened.id == "2026-09-10-lab-meeting"
    assert opened.meta().title == "Lab meeting"


@pytest.mark.parametrize("job_id", ["missing", "empty", "../jobs", ".."])
def test_open_job_rejects_unknown_empty_and_path_like_ids(tmp_path, job_id):
    (tmp_path / "empty").mkdir()
    with pytest.raises(JobNotFound):
        open_job(tmp_path, job_id)


def test_log_appends_json_lines(make_job):
    job = make_job()
    job.append_log("ingest", "ok", audio_duration_s=2.0)
    job.append_log("transcribe", "error", error="boom")
    log = job.read_log()
    assert [(e["stage"], e["status"]) for e in log] == [("ingest", "ok"), ("transcribe", "error")]
    assert log[0]["audio_duration_s"] == 2.0
    assert log[1]["error"] == "boom"
    assert "ts" in log[0]


def test_read_log_of_a_new_job_is_empty(make_job):
    assert make_job().read_log() == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_meeting.py tests/test_jobs.py -v`
Expected: errors with `ModuleNotFoundError: No module named 'meeting_record.io'`

- [ ] **Step 3: Write the implementation**

`src/meeting_record/io.py`:

```python
"""Atomic writes: a stage output either exists completely or not at all."""

from __future__ import annotations

import json
import os
from pathlib import Path

from pydantic import BaseModel


def write_text_atomic(path: Path, text: str) -> None:
    tmp = path.with_name(f".{path.name}.tmp")
    tmp.write_text(text, encoding="utf-8")
    os.replace(tmp, path)


def write_json_atomic(path: Path, data: BaseModel | dict) -> None:
    if isinstance(data, BaseModel):
        text = data.model_dump_json(indent=2)
    else:
        text = json.dumps(data, indent=2, ensure_ascii=False)
    write_text_atomic(path, text + "\n")
```

`src/meeting_record/meeting.py`:

```python
"""What the owner tells us about a Meeting, stored as meeting.json."""

from __future__ import annotations

import datetime as dt
import re
from typing import Literal

from pydantic import BaseModel, Field, model_validator

Language = Literal["no", "nn", "en"]

_NAME_EMAIL = re.compile(r"\s*(.+?)\s*<\s*([^<>\s]+@[^<>\s]+)\s*>\s*")


class Attendee(BaseModel):
    name: str = Field(min_length=1)
    email: str | None = None

    @classmethod
    def parse(cls, raw: str) -> Attendee:
        match = _NAME_EMAIL.fullmatch(raw)
        if match:
            return cls(name=match.group(1), email=match.group(2))
        return cls(name=raw.strip())


class RecordingInfo(BaseModel):
    original_name: str
    file: str
    sha256: str
    duration_s: float
    had_video: bool


class MeetingMeta(BaseModel):
    schema_version: Literal[1] = 1
    meeting_id: str
    title: str = Field(min_length=1)
    date: dt.date
    language: Language
    attendees: list[Attendee] = Field(default_factory=list)
    min_speakers: int | None = Field(default=None, ge=1)
    max_speakers: int | None = Field(default=None, ge=1)
    recording: RecordingInfo
    created_at: dt.datetime

    @model_validator(mode="after")
    def _speaker_bounds_in_order(self) -> MeetingMeta:
        if self.min_speakers and self.max_speakers and self.min_speakers > self.max_speakers:
            raise ValueError("min_speakers is greater than max_speakers")
        return self
```

`src/meeting_record/jobs.py`:

```python
"""The Job directory: one folder per Meeting holding every artifact of its run."""

from __future__ import annotations

import datetime as dt
import json
import re
import unicodedata
from dataclasses import dataclass
from pathlib import Path

from meeting_record.meeting import MeetingMeta

MEETING_FILE = "meeting.json"
RAW_TRANSCRIPT_FILE = "transcript.raw.json"
RAW_TRANSCRIPT_TEXT_FILE = "transcript.raw.txt"
LOG_FILE = "log.jsonl"

_TRANSLIT = str.maketrans({"æ": "ae", "ø": "o", "å": "a", "Æ": "ae", "Ø": "o", "Å": "a"})


class JobNotFound(Exception):
    pass


def slugify(title: str) -> str:
    text = unicodedata.normalize("NFKD", title.translate(_TRANSLIT))
    text = text.encode("ascii", "ignore").decode().lower()
    slug = re.sub(r"[^a-z0-9]+", "-", text).strip("-")
    return slug[:60].rstrip("-") or "meeting"


def new_job_dir(jobs_root: Path, date: dt.date, title: str) -> Path:
    jobs_root.mkdir(parents=True, exist_ok=True)
    base = f"{date.isoformat()}-{slugify(title)}"
    n = 1
    while True:
        candidate = jobs_root / (base if n == 1 else f"{base}-{n}")
        try:
            candidate.mkdir()
            return candidate
        except FileExistsError:
            n += 1


@dataclass(frozen=True)
class Job:
    dir: Path

    @property
    def id(self) -> str:
        return self.dir.name

    def path(self, name: str) -> Path:
        return self.dir / name

    def meta(self) -> MeetingMeta:
        return MeetingMeta.model_validate_json(self.path(MEETING_FILE).read_text(encoding="utf-8"))

    def append_log(self, stage: str, status: str, **fields) -> None:
        entry = {
            "ts": dt.datetime.now(dt.timezone.utc).isoformat(timespec="seconds"),
            "stage": stage,
            "status": status,
            **fields,
        }
        with self.path(LOG_FILE).open("a", encoding="utf-8") as f:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")

    def read_log(self) -> list[dict]:
        path = self.path(LOG_FILE)
        if not path.exists():
            return []
        lines = path.read_text(encoding="utf-8").splitlines()
        return [json.loads(line) for line in lines if line.strip()]


def open_job(jobs_root: Path, job_id: str) -> Job:
    job_dir = jobs_root / job_id
    if "/" in job_id or job_id in {".", ".."} or not (job_dir / MEETING_FILE).is_file():
        raise JobNotFound(f"no job {job_id!r} in {jobs_root}")
    return Job(job_dir)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_meeting.py tests/test_jobs.py -v`
Expected: all passed (19 tests)

- [ ] **Step 5: Commit**

```bash
git add src/meeting_record/io.py src/meeting_record/meeting.py src/meeting_record/jobs.py tests/conftest.py tests/test_meeting.py tests/test_jobs.py
git commit -m "Add Meeting metadata and Job directory with atomic writes and log"
```

---

### Task 3: Ingest a Recording

**Files:**
- Create: `src/meeting_record/ingest.py`
- Modify: `tests/conftest.py` (append the `media` fixture)
- Test: `tests/test_ingest.py`

**Interfaces:**
- Consumes: `new_job_dir`, `Job`, `MEETING_FILE` from `meeting_record.jobs`; `Attendee`, `Language`, `MeetingMeta`, `RecordingInfo` from `meeting_record.meeting`; `write_json_atomic` from `meeting_record.io`.
- Produces:
  - `meeting_record.ingest.IngestError(Exception)`
  - `meeting_record.ingest.Probe(duration_s: float, has_audio: bool, has_video: bool)` and `probe(path: Path) -> Probe`
  - `meeting_record.ingest.create_job(jobs_root: Path, source: Path, *, title: str, date: dt.date, language: Language, attendees: list[Attendee] | None = None, min_speakers: int | None = None, max_speakers: int | None = None, now: Callable[[], dt.datetime] = ...) -> Job`. On any failure it removes the half-made Job directory and re-raises. On success it logs `("ingest", "ok", source=<name>, audio_duration_s=<float>)`.
  - pytest fixture `media -> Path`, a directory with `tone.wav` (2 s mono sine), `clip.mp4` (2 s video with audio), `silent.mp4` (video, no audio), `notes.txt`. Skips the test if ffmpeg is missing.

- [ ] **Step 1: Append the media fixture to `tests/conftest.py`**

Add these imports at the top of `tests/conftest.py`:

```python
import shutil
import subprocess
```

Append at the end:

```python
def _ffmpeg(*args: str) -> None:
    subprocess.run(["ffmpeg", "-nostdin", "-v", "error", "-y", *args], check=True)


@pytest.fixture
def media(tmp_path):
    """Tiny generated media files; nothing recorded, nothing committed."""
    if shutil.which("ffmpeg") is None:
        pytest.skip("ffmpeg not installed")
    d = tmp_path / "media"
    d.mkdir()
    _ffmpeg("-f", "lavfi", "-i", "sine=frequency=440:duration=2", "-ac", "1", str(d / "tone.wav"))
    _ffmpeg(
        "-f", "lavfi", "-i", "testsrc=size=64x64:rate=5:duration=2",
        "-f", "lavfi", "-i", "sine=frequency=440:duration=2",
        "-shortest", "-c:v", "mpeg4", "-c:a", "aac", str(d / "clip.mp4"),
    )
    _ffmpeg("-f", "lavfi", "-i", "testsrc=size=64x64:rate=5:duration=2", "-c:v", "mpeg4", str(d / "silent.mp4"))
    (d / "notes.txt").write_text("not media", encoding="utf-8")
    return d
```

- [ ] **Step 2: Write the failing test**

`tests/test_ingest.py`:

```python
import datetime as dt
import hashlib

import pytest
from pydantic import ValidationError

from meeting_record.ingest import IngestError, create_job, probe
from meeting_record.jobs import open_job
from meeting_record.meeting import Attendee

FIXED_NOW = dt.datetime(2026, 9, 11, 9, 30, tzinfo=dt.timezone.utc)


def _job_dirs(root):
    return sorted(p.name for p in root.iterdir()) if root.exists() else []


def _create(root, source, **overrides):
    kwargs = dict(title="Lab møte", date=dt.date(2026, 9, 10), language="no", now=lambda: FIXED_NOW)
    kwargs.update(overrides)
    return create_job(root, source, **kwargs)


def test_audio_recording_is_copied_unchanged_and_registered(tmp_path, media):
    root = tmp_path / "jobs"
    src = media / "tone.wav"
    job = _create(root, src, attendees=[Attendee.parse("Kari <kari@example.org>")], max_speakers=3)

    assert job.id == "2026-09-10-lab-mote"
    assert job.path("audio.wav").read_bytes() == src.read_bytes()
    meta = open_job(root, job.id).meta()
    assert meta.title == "Lab møte"
    assert meta.language == "no"
    assert meta.attendees[0].email == "kari@example.org"
    assert meta.max_speakers == 3
    assert meta.created_at == FIXED_NOW
    assert meta.recording.file == "audio.wav"
    assert meta.recording.original_name == "tone.wav"
    assert meta.recording.sha256 == hashlib.sha256(src.read_bytes()).hexdigest()
    assert meta.recording.duration_s == pytest.approx(2.0, abs=0.1)
    assert meta.recording.had_video is False
    assert [(e["stage"], e["status"]) for e in job.read_log()] == [("ingest", "ok")]


def test_video_recording_is_stored_as_mono_16k_flac(tmp_path, media):
    job = _create(tmp_path / "jobs", media / "clip.mp4")
    meta = job.meta()
    assert meta.recording.file == "audio.flac"
    assert meta.recording.had_video is True
    extracted = probe(job.path("audio.flac"))
    assert extracted.has_audio and not extracted.has_video
    assert extracted.duration_s == pytest.approx(2.0, abs=0.2)


def test_video_without_audio_is_rejected_and_leaves_nothing(tmp_path, media):
    root = tmp_path / "jobs"
    with pytest.raises(IngestError, match="no audio"):
        _create(root, media / "silent.mp4")
    assert _job_dirs(root) == []


def test_missing_file_is_rejected(tmp_path):
    with pytest.raises(IngestError, match="not found"):
        _create(tmp_path / "jobs", tmp_path / "nope.m4a")


def test_non_media_file_is_rejected_and_leaves_nothing(tmp_path, media):
    root = tmp_path / "jobs"
    with pytest.raises(IngestError):
        _create(root, media / "notes.txt")
    assert _job_dirs(root) == []


def test_invalid_metadata_leaves_nothing(tmp_path, media):
    root = tmp_path / "jobs"
    with pytest.raises(ValidationError):
        _create(root, media / "tone.wav", min_speakers=5, max_speakers=2)
    assert _job_dirs(root) == []


def test_same_title_and_date_gets_a_new_job(tmp_path, media):
    root = tmp_path / "jobs"
    first = _create(root, media / "tone.wav")
    second = _create(root, media / "tone.wav")
    assert (first.id, second.id) == ("2026-09-10-lab-mote", "2026-09-10-lab-mote-2")
```

- [ ] **Step 3: Run test to verify it fails**

Run: `uv run pytest tests/test_ingest.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'meeting_record.ingest'`

- [ ] **Step 4: Write the implementation**

`src/meeting_record/ingest.py`:

```python
"""Stage 1, ingest: register a Meeting and copy its Recording into a new Job."""

from __future__ import annotations

import datetime as dt
import hashlib
import json
import shutil
import subprocess
from collections.abc import Callable
from dataclasses import dataclass
from pathlib import Path

from meeting_record.io import write_json_atomic
from meeting_record.jobs import MEETING_FILE, Job, new_job_dir
from meeting_record.meeting import Attendee, Language, MeetingMeta, RecordingInfo


class IngestError(Exception):
    pass


@dataclass(frozen=True)
class Probe:
    duration_s: float
    has_audio: bool
    has_video: bool


def probe(path: Path) -> Probe:
    proc = subprocess.run(
        ["ffprobe", "-v", "error", "-print_format", "json", "-show_format", "-show_streams", str(path)],
        capture_output=True,
        text=True,
    )
    if proc.returncode != 0:
        raise IngestError(f"ffprobe could not read {path.name}: {proc.stderr.strip()}")
    info = json.loads(proc.stdout)
    streams = info.get("streams", [])
    has_audio = any(s.get("codec_type") == "audio" for s in streams)
    # Cover art in m4a/mp3 files shows up as a video stream flagged attached_pic.
    has_video = any(
        s.get("codec_type") == "video" and not s.get("disposition", {}).get("attached_pic")
        for s in streams
    )
    duration = float(info.get("format", {}).get("duration", 0.0))
    return Probe(duration_s=duration, has_audio=has_audio, has_video=has_video)


def _sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1 << 20), b""):
            digest.update(chunk)
    return digest.hexdigest()


def _extract_audio(source: Path, dest: Path) -> None:
    proc = subprocess.run(
        ["ffmpeg", "-nostdin", "-v", "error", "-y", "-i", str(source),
         "-vn", "-ac", "1", "-ar", "16000", "-c:a", "flac", str(dest)],
        capture_output=True,
        text=True,
    )
    if proc.returncode != 0:
        raise IngestError(f"ffmpeg could not extract audio from {source.name}: {proc.stderr.strip()}")


def _utc_now() -> dt.datetime:
    return dt.datetime.now(dt.timezone.utc)


def create_job(
    jobs_root: Path,
    source: Path,
    *,
    title: str,
    date: dt.date,
    language: Language,
    attendees: list[Attendee] | None = None,
    min_speakers: int | None = None,
    max_speakers: int | None = None,
    now: Callable[[], dt.datetime] = _utc_now,
) -> Job:
    if not source.is_file():
        raise IngestError(f"recording not found: {source}")
    info = probe(source)
    if not info.has_audio:
        raise IngestError(f"{source.name} has no audio stream")
    if info.duration_s <= 0:
        raise IngestError(f"{source.name} has no measurable duration")

    job = Job(new_job_dir(jobs_root, date, title))
    try:
        if info.has_video:
            audio_name = "audio.flac"
            _extract_audio(source, job.path(audio_name))
        else:
            audio_name = f"audio{source.suffix.lower()}"
            shutil.copyfile(source, job.path(audio_name))
        meta = MeetingMeta(
            meeting_id=job.id,
            title=title,
            date=date,
            language=language,
            attendees=attendees or [],
            min_speakers=min_speakers,
            max_speakers=max_speakers,
            recording=RecordingInfo(
                original_name=source.name,
                file=audio_name,
                sha256=_sha256(source),
                duration_s=round(info.duration_s, 3),
                had_video=info.has_video,
            ),
            created_at=now(),
        )
        write_json_atomic(job.path(MEETING_FILE), meta)
    except BaseException:
        shutil.rmtree(job.dir, ignore_errors=True)
        raise
    job.append_log("ingest", "ok", source=source.name, audio_duration_s=meta.recording.duration_s)
    return job
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_ingest.py -v`
Expected: 7 passed

- [ ] **Step 6: Commit**

```bash
git add src/meeting_record/ingest.py tests/conftest.py tests/test_ingest.py
git commit -m "Ingest a Recording into a new Job, extracting audio from video"
```

---

### Task 4: Normalized Transcript format

**Files:**
- Create: `src/meeting_record/transcript.py`
- Test: `tests/test_transcript.py`

**Interfaces:**
- Consumes: `Language` from `meeting_record.meeting`.
- Produces:
  - `Word(text: str, start: float | None = None, end: float | None = None, score: float | None = None, speaker: str | None = None)`
  - `Segment(id: str, start: float, end: float, speaker: str | None, text: str, confidence: float | None = None, words: list[Word] = [])`. `confidence` is the mean word alignment score, not an ASR probability.
  - `EngineInfo(name: str, version: str, asr_model: str, compute_type: str, batch_size: int, diarization_model: str)`
  - `RawTranscript(schema_version=1, meeting_id: str, language: Language, duration_s: float, speakers: list[str], engine: EngineInfo, segments: list[Segment])`
  - `normalize_segments(raw_segments: list[dict]) -> list[Segment]`: input is WhisperX's `result["segments"]` after `assign_word_speakers`. Splits a segment wherever the word speaker changes. Words without a speaker inherit the previous word's speaker, starting from the segment's speaker. Words without timestamps stay in place with `start`/`end` `None`. Drops empty segments. Numbers segments `s0001`, `s0002`, … in order.
  - `speakers_in_order(segments: list[Segment]) -> list[str]`
  - `format_timestamp(seconds: float) -> str` → `"HH:MM:SS"`
  - `render_text(transcript: RawTranscript) -> str`: one line per segment, `[HH:MM:SS] SPEAKER_00: text`, `UNKNOWN` when the speaker is `None`, trailing newline.

- [ ] **Step 1: Write the failing test**

`tests/test_transcript.py`:

```python
import pytest

from meeting_record.transcript import (
    EngineInfo,
    RawTranscript,
    format_timestamp,
    normalize_segments,
    render_text,
    speakers_in_order,
)


def w(text, start=None, end=None, score=None, speaker=None):
    word = {"word": text}
    if start is not None:
        word.update(start=start, end=end, score=score)
    if speaker is not None:
        word["speaker"] = speaker
    return word


ONE_SPEAKER = {
    "start": 0.5, "end": 4.2, "text": " Vi flytter fristen til fredag.", "speaker": "SPEAKER_00",
    "words": [
        w("Vi", 0.5, 0.7, 0.9, "SPEAKER_00"),
        w("flytter", 0.8, 1.2, 0.8, "SPEAKER_00"),
        w("fristen", 1.3, 1.8, 0.7, "SPEAKER_00"),
        w("til", 1.9, 2.0, 0.9, "SPEAKER_00"),
        w("fredag.", 2.1, 2.6, 0.7, "SPEAKER_00"),
    ],
}

SPEAKER_CHANGE = {
    "start": 5.0, "end": 8.0, "text": "Greit. Men hva med budsjettet?", "speaker": "SPEAKER_01",
    "words": [
        w("Greit.", 5.0, 5.4, 0.9, "SPEAKER_00"),
        w("Men", 6.0, 6.2, 0.9, "SPEAKER_01"),
        w("hva", 6.3, 6.5, 0.9, "SPEAKER_01"),
        w("med", 6.6, 6.7, 0.9, "SPEAKER_01"),
        w("budsjettet?", 6.8, 8.0, 0.6, "SPEAKER_01"),
    ],
}


def test_single_speaker_segment_becomes_one_segment():
    [seg] = normalize_segments([ONE_SPEAKER])
    assert seg.id == "s0001"
    assert seg.text == "Vi flytter fristen til fredag."
    assert seg.speaker == "SPEAKER_00"
    assert (seg.start, seg.end) == (0.5, 2.6)
    assert seg.confidence == pytest.approx(0.8)
    assert [x.text for x in seg.words] == ["Vi", "flytter", "fristen", "til", "fredag."]


def test_segment_is_split_where_the_speaker_changes():
    first, second = normalize_segments([SPEAKER_CHANGE])
    assert (first.id, first.speaker, first.text, first.start, first.end) == (
        "s0001", "SPEAKER_00", "Greit.", 5.0, 5.4
    )
    assert (second.id, second.speaker, second.text, second.start, second.end) == (
        "s0002", "SPEAKER_01", "Men hva med budsjettet?", 6.0, 8.0
    )
    assert second.confidence == pytest.approx(0.825)


def test_unaligned_word_inherits_speaker_and_does_not_split():
    raw = {
        "start": 10.0, "end": 12.0, "text": "Frist 15. oktober", "speaker": "SPEAKER_00",
        "words": [w("Frist", 10.0, 10.4, 0.9, "SPEAKER_00"), w("15."), w("oktober", 11.0, 12.0, 0.8, "SPEAKER_00")],
    }
    [seg] = normalize_segments([raw])
    assert seg.text == "Frist 15. oktober"
    assert seg.words[1].speaker == "SPEAKER_00"
    assert seg.words[1].start is None
    assert seg.confidence == pytest.approx(0.85)


def test_unaligned_last_word_uses_last_known_end():
    raw = {
        "start": 1.0, "end": 3.0, "text": "Rom 214", "speaker": "SPEAKER_02",
        "words": [w("Rom", 1.0, 1.5, 0.9, "SPEAKER_02"), w("214")],
    }
    [seg] = normalize_segments([raw])
    assert seg.end == 1.5


def test_leading_word_without_speaker_takes_the_segment_speaker():
    raw = {
        "start": 0.0, "end": 1.0, "text": "Ja, ok", "speaker": "SPEAKER_03",
        "words": [w("Ja,", 0.0, 0.3, 0.9), w("ok", 0.4, 1.0, 0.9, "SPEAKER_03")],
    }
    [seg] = normalize_segments([raw])
    assert seg.speaker == "SPEAKER_03"
    assert seg.words[0].speaker == "SPEAKER_03"


def test_segment_without_words_keeps_its_text_and_empty_ones_are_dropped():
    raw = [
        {"start": 20.0, "end": 21.0, "text": " Mhm. ", "words": []},
        {"start": 22.0, "end": 23.0, "text": "   ", "words": []},
        ONE_SPEAKER,
    ]
    segs = normalize_segments(raw)
    assert [(s.id, s.text, s.speaker) for s in segs] == [
        ("s0001", "Mhm.", None),
        ("s0002", "Vi flytter fristen til fredag.", "SPEAKER_00"),
    ]
    assert segs[0].confidence is None


def test_ids_run_across_segments_in_order():
    segs = normalize_segments([ONE_SPEAKER, SPEAKER_CHANGE])
    assert [s.id for s in segs] == ["s0001", "s0002", "s0003"]
    assert speakers_in_order(segs) == ["SPEAKER_00", "SPEAKER_01"]


def test_format_timestamp():
    assert format_timestamp(0) == "00:00:00"
    assert format_timestamp(3723.9) == "01:02:03"


def _transcript(segments):
    return RawTranscript(
        meeting_id="2026-09-10-lab-meeting",
        language="no",
        duration_s=8.0,
        speakers=speakers_in_order(segments),
        engine=EngineInfo(
            name="whisperx", version="3.8.6", asr_model="NbAiLab/nb-whisper-large",
            compute_type="float16", batch_size=8, diarization_model="pyannote/speaker-diarization-community-1",
        ),
        segments=segments,
    )


def test_render_text_one_line_per_segment():
    segs = normalize_segments([{"start": 62.0, "end": 63.0, "text": "Mhm.", "words": []}, SPEAKER_CHANGE])
    assert render_text(_transcript(segs)) == (
        "[00:01:02] UNKNOWN: Mhm.\n"
        "[00:00:05] SPEAKER_00: Greit.\n"
        "[00:00:06] SPEAKER_01: Men hva med budsjettet?\n"
    )


def test_raw_transcript_round_trips_json():
    transcript = _transcript(normalize_segments([ONE_SPEAKER, SPEAKER_CHANGE]))
    assert RawTranscript.model_validate_json(transcript.model_dump_json()) == transcript
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_transcript.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'meeting_record.transcript'`

- [ ] **Step 3: Write the implementation**

`src/meeting_record/transcript.py`:

```python
"""The normalized Transcript format. This is the speech-to-text vendor seam:
nothing downstream sees WhisperX's shapes, only these models."""

from __future__ import annotations

from typing import Literal

from pydantic import BaseModel, Field

from meeting_record.meeting import Language


class Word(BaseModel):
    text: str
    start: float | None = None
    end: float | None = None
    score: float | None = None
    speaker: str | None = None


class Segment(BaseModel):
    id: str
    start: float
    end: float
    speaker: str | None
    text: str
    confidence: float | None = None  # mean word alignment score
    words: list[Word] = Field(default_factory=list)


class EngineInfo(BaseModel):
    name: str
    version: str
    asr_model: str
    compute_type: str
    batch_size: int
    diarization_model: str


class RawTranscript(BaseModel):
    schema_version: Literal[1] = 1
    meeting_id: str
    language: Language
    duration_s: float
    speakers: list[str]
    engine: EngineInfo
    segments: list[Segment]


def _speaker_runs(words: list[Word], segment_speaker: str | None) -> list[list[Word]]:
    runs: list[list[Word]] = []
    current = segment_speaker
    for word in words:
        speaker = word.speaker or current
        word = word.model_copy(update={"speaker": speaker})
        if runs and runs[-1][0].speaker == speaker:
            runs[-1].append(word)
        else:
            runs.append([word])
        current = speaker
    return runs


def _mean_score(words: list[Word]) -> float | None:
    scores = [w.score for w in words if w.score is not None]
    return round(sum(scores) / len(scores), 3) if scores else None


def normalize_segments(raw_segments: list[dict]) -> list[Segment]:
    drafts: list[dict] = []
    for raw in raw_segments:
        segment_speaker = raw.get("speaker")
        words = [
            Word(
                text=str(w["word"]).strip(),
                start=w.get("start"),
                end=w.get("end"),
                score=w.get("score"),
                speaker=w.get("speaker"),
            )
            for w in raw.get("words", [])
            if str(w.get("word", "")).strip()
        ]
        if not words:
            text = str(raw.get("text", "")).strip()
            if text:
                drafts.append(
                    dict(start=raw["start"], end=raw["end"], speaker=segment_speaker, text=text, words=[])
                )
            continue
        # A new run only starts at a word that carries its own speaker, and WhisperX
        # only assigns speakers to timed words, so the fallbacks below apply to the
        # first run of a segment only.
        for run in _speaker_runs(words, segment_speaker):
            starts = [w.start for w in run if w.start is not None]
            ends = [w.end for w in run if w.end is not None]
            drafts.append(
                dict(
                    start=starts[0] if starts else raw["start"],
                    end=ends[-1] if ends else raw["end"],
                    speaker=run[0].speaker,
                    text=" ".join(w.text for w in run),
                    words=run,
                )
            )
    return [
        Segment(
            id=f"s{i:04d}",
            start=round(d["start"], 3),
            end=round(d["end"], 3),
            speaker=d["speaker"],
            text=d["text"],
            confidence=_mean_score(d["words"]),
            words=d["words"],
        )
        for i, d in enumerate(drafts, start=1)
    ]


def speakers_in_order(segments: list[Segment]) -> list[str]:
    seen: list[str] = []
    for segment in segments:
        if segment.speaker and segment.speaker not in seen:
            seen.append(segment.speaker)
    return seen


def format_timestamp(seconds: float) -> str:
    total = int(seconds)
    return f"{total // 3600:02d}:{total % 3600 // 60:02d}:{total % 60:02d}"


def render_text(transcript: RawTranscript) -> str:
    lines = [
        f"[{format_timestamp(s.start)}] {s.speaker or 'UNKNOWN'}: {s.text}" for s in transcript.segments
    ]
    return "\n".join(lines) + "\n"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/test_transcript.py -v`
Expected: 10 passed

- [ ] **Step 5: Commit**

```bash
git add src/meeting_record/transcript.py tests/test_transcript.py
git commit -m "Add normalized Transcript format with speaker-change splitting"
```

---

### Task 5: Transcribe stage and orchestrator

**Files:**
- Create: `src/meeting_record/transcribe.py`, `src/meeting_record/orchestrator.py`
- Modify: `tests/conftest.py` (append the `fake_transcriber` fixture)
- Test: `tests/test_transcribe_stage.py`, `tests/test_orchestrator.py`

**Interfaces:**
- Consumes: `Job`, `RAW_TRANSCRIPT_FILE`, `RAW_TRANSCRIPT_TEXT_FILE` from `meeting_record.jobs`; `write_json_atomic`, `write_text_atomic` from `meeting_record.io`; `Segment`, `EngineInfo`, `RawTranscript`, `render_text`, `speakers_in_order` from `meeting_record.transcript`; `Language` from `meeting_record.meeting`.
- Produces:
  - `meeting_record.transcribe.TranscriptionResult(segments: list[Segment], engine: EngineInfo, timings_s: dict[str, float] = {})`
  - `meeting_record.transcribe.Transcriber` protocol: `transcribe(self, audio_path: Path, *, language: Language, min_speakers: int | None, max_speakers: int | None) -> TranscriptionResult`
  - `meeting_record.transcribe.transcribe_stage(job: Job, transcriber: Transcriber) -> dict`. Writes `transcript.raw.txt` then `transcript.raw.json` (the JSON file is the completion marker). Returns `{"segments": int, "speakers": int, "asr_model": str, "timings_s": dict}`. Raises `ValueError` if the transcriber returns no segments.
  - `meeting_record.orchestrator.Stage(name: str, output: str, run: Callable[[Job], dict | None])`
  - `meeting_record.orchestrator.StageOutcome(stage: str, status: Literal["ok", "skipped"], duration_s: float = 0.0)`
  - `meeting_record.orchestrator.StageFailed(Exception)` with `.stage: str` and `.cause: BaseException`
  - `meeting_record.orchestrator.run_job(job: Job, stages: Sequence[Stage], *, force: Collection[str] = (), clock: Callable[[], float] = time.monotonic) -> list[StageOutcome]`. Skips a stage whose output exists unless it is forced or an earlier stage ran in this call. Logs every outcome to `log.jsonl`. Raises `ValueError` for unknown names in `force`.
  - pytest fixture `fake_transcriber` returning an object with `.calls: list[dict]` and `.segments: list[Segment]` (two default segments: `s0001` SPEAKER_00 "Vi flytter fristen til fredag." 0.0–2.0 and `s0002` SPEAKER_01 "Greit." 2.5–4.0), and a `.transcribe(...)` that records the call and returns those segments with engine name `"fake"` and `timings_s={"asr": 1.0}`.

- [ ] **Step 1: Append the fake transcriber to `tests/conftest.py`**

Append at the end of `tests/conftest.py`:

```python
class FakeTranscriber:
    """Stands in for the GPU engine: records what it was asked, returns fixed Segments."""

    def __init__(self):
        from meeting_record.transcript import Segment

        self.calls = []
        self.segments = [
            Segment(id="s0001", start=0.0, end=2.0, speaker="SPEAKER_00",
                    text="Vi flytter fristen til fredag.", confidence=0.9),
            Segment(id="s0002", start=2.5, end=4.0, speaker="SPEAKER_01", text="Greit.", confidence=0.8),
        ]

    def transcribe(self, audio_path, *, language, min_speakers, max_speakers):
        from meeting_record.transcribe import TranscriptionResult
        from meeting_record.transcript import EngineInfo

        self.calls.append(
            {"audio_path": audio_path, "language": language,
             "min_speakers": min_speakers, "max_speakers": max_speakers}
        )
        return TranscriptionResult(
            segments=self.segments,
            engine=EngineInfo(name="fake", version="0", asr_model="fake-asr", compute_type="none",
                              batch_size=1, diarization_model="fake-diarization"),
            timings_s={"asr": 1.0},
        )


@pytest.fixture
def fake_transcriber():
    return FakeTranscriber()
```

- [ ] **Step 2: Write the failing tests**

`tests/test_transcribe_stage.py`:

```python
import pytest

from meeting_record.jobs import RAW_TRANSCRIPT_FILE, RAW_TRANSCRIPT_TEXT_FILE
from meeting_record.transcribe import transcribe_stage
from meeting_record.transcript import RawTranscript


def test_writes_json_and_text_transcripts(make_job, fake_transcriber):
    job = make_job()
    details = transcribe_stage(job, fake_transcriber)

    transcript = RawTranscript.model_validate_json(job.path(RAW_TRANSCRIPT_FILE).read_text(encoding="utf-8"))
    assert transcript.meeting_id == job.id
    assert transcript.language == "no"
    assert transcript.duration_s == 4.0
    assert transcript.speakers == ["SPEAKER_00", "SPEAKER_01"]
    assert transcript.engine.name == "fake"
    assert [s.id for s in transcript.segments] == ["s0001", "s0002"]
    assert job.path(RAW_TRANSCRIPT_TEXT_FILE).read_text(encoding="utf-8").splitlines() == [
        "[00:00:00] SPEAKER_00: Vi flytter fristen til fredag.",
        "[00:00:02] SPEAKER_01: Greit.",
    ]
    assert details == {"segments": 2, "speakers": 2, "asr_model": "fake-asr", "timings_s": {"asr": 1.0}}


def test_passes_meeting_language_speaker_bounds_and_audio_path(make_job, fake_transcriber):
    job = make_job(language="nn", min_speakers=2, max_speakers=5)
    transcribe_stage(job, fake_transcriber)
    assert fake_transcriber.calls == [
        {"audio_path": job.path("audio.wav"), "language": "nn", "min_speakers": 2, "max_speakers": 5}
    ]


def test_no_segments_is_an_error_and_writes_no_transcript(make_job, fake_transcriber):
    job = make_job()
    fake_transcriber.segments = []
    with pytest.raises(ValueError, match="no segments"):
        transcribe_stage(job, fake_transcriber)
    assert not job.path(RAW_TRANSCRIPT_FILE).exists()
```

`tests/test_orchestrator.py`:

```python
import pytest

from meeting_record.orchestrator import Stage, StageFailed, run_job


def writer(name, output, calls, fail=False):
    def run(job):
        calls.append(name)
        if fail:
            raise RuntimeError("boom")
        job.path(output).write_text(name, encoding="utf-8")
        return {"wrote": output}

    return Stage(name=name, output=output, run=run)


def fake_clock(*times):
    return iter(times).__next__


def test_runs_stages_in_order_and_logs_duration_and_details(make_job):
    job = make_job()
    calls = []
    stages = [writer("a", "a.out", calls), writer("b", "b.out", calls)]
    outcomes = run_job(job, stages, clock=fake_clock(10.0, 12.5, 20.0, 21.0))

    assert calls == ["a", "b"]
    assert [(o.stage, o.status, o.duration_s) for o in outcomes] == [("a", "ok", 2.5), ("b", "ok", 1.0)]
    log = job.read_log()
    assert [(e["stage"], e["status"], e["duration_s"], e["wrote"]) for e in log] == [
        ("a", "ok", 2.5, "a.out"),
        ("b", "ok", 1.0, "b.out"),
    ]


def test_skips_a_stage_whose_output_exists(make_job):
    job = make_job()
    job.path("a.out").write_text("earlier", encoding="utf-8")
    calls = []
    outcomes = run_job(job, [writer("a", "a.out", calls), writer("b", "b.out", calls)])

    assert calls == ["b"]
    assert [(o.stage, o.status) for o in outcomes] == [("a", "skipped"), ("b", "ok")]
    assert job.path("a.out").read_text(encoding="utf-8") == "earlier"
    assert [(e["stage"], e["status"]) for e in job.read_log()] == [("a", "skipped"), ("b", "ok")]


def test_forcing_a_stage_reruns_it_and_everything_after(make_job):
    job = make_job()
    job.path("a.out").write_text("old", encoding="utf-8")
    job.path("b.out").write_text("old", encoding="utf-8")
    calls = []
    outcomes = run_job(job, [writer("a", "a.out", calls), writer("b", "b.out", calls)], force=["a"])

    assert calls == ["a", "b"]
    assert [o.status for o in outcomes] == ["ok", "ok"]


def test_failure_is_logged_raised_and_stops_later_stages(make_job):
    job = make_job()
    calls = []
    stages = [writer("a", "a.out", calls, fail=True), writer("b", "b.out", calls)]
    with pytest.raises(StageFailed) as caught:
        run_job(job, stages)

    assert caught.value.stage == "a"
    assert isinstance(caught.value.cause, RuntimeError)
    assert calls == ["a"]
    [entry] = job.read_log()
    assert (entry["stage"], entry["status"], entry["error"]) == ("a", "error", "RuntimeError: boom")


def test_unknown_forced_stage_is_rejected_before_anything_runs(make_job):
    job = make_job()
    calls = []
    with pytest.raises(ValueError, match="nope"):
        run_job(job, [writer("a", "a.out", calls)], force=["nope"])
    assert calls == []
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `uv run pytest tests/test_transcribe_stage.py tests/test_orchestrator.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'meeting_record.transcribe'` and `'meeting_record.orchestrator'`

- [ ] **Step 4: Write the implementation**

`src/meeting_record/transcribe.py`:

```python
"""Stage 2, transcribe: Recording to raw Transcript through a swappable Transcriber."""

from __future__ import annotations

from pathlib import Path
from typing import Protocol

from pydantic import BaseModel, Field

from meeting_record.io import write_json_atomic, write_text_atomic
from meeting_record.jobs import RAW_TRANSCRIPT_FILE, RAW_TRANSCRIPT_TEXT_FILE, Job
from meeting_record.meeting import Language
from meeting_record.transcript import EngineInfo, RawTranscript, Segment, render_text, speakers_in_order


class TranscriptionResult(BaseModel):
    segments: list[Segment]
    engine: EngineInfo
    timings_s: dict[str, float] = Field(default_factory=dict)


class Transcriber(Protocol):
    def transcribe(
        self,
        audio_path: Path,
        *,
        language: Language,
        min_speakers: int | None,
        max_speakers: int | None,
    ) -> TranscriptionResult: ...


def transcribe_stage(job: Job, transcriber: Transcriber) -> dict:
    meta = job.meta()
    result = transcriber.transcribe(
        job.path(meta.recording.file),
        language=meta.language,
        min_speakers=meta.min_speakers,
        max_speakers=meta.max_speakers,
    )
    if not result.segments:
        raise ValueError("speech-to-text returned no segments; is there speech in the Recording?")
    transcript = RawTranscript(
        meeting_id=meta.meeting_id,
        language=meta.language,
        duration_s=meta.recording.duration_s,
        speakers=speakers_in_order(result.segments),
        engine=result.engine,
        segments=result.segments,
    )
    write_text_atomic(job.path(RAW_TRANSCRIPT_TEXT_FILE), render_text(transcript))
    write_json_atomic(job.path(RAW_TRANSCRIPT_FILE), transcript)  # written last: marks the stage done
    return {
        "segments": len(transcript.segments),
        "speakers": len(transcript.speakers),
        "asr_model": result.engine.asr_model,
        "timings_s": result.timings_s,
    }
```

`src/meeting_record/orchestrator.py`:

```python
"""Advances a Job through its stages. A loop, not a framework."""

from __future__ import annotations

import time
from collections.abc import Callable, Collection, Sequence
from dataclasses import dataclass
from typing import Literal

from meeting_record.jobs import Job


@dataclass(frozen=True)
class Stage:
    name: str
    output: str
    run: Callable[[Job], dict | None]


@dataclass(frozen=True)
class StageOutcome:
    stage: str
    status: Literal["ok", "skipped"]
    duration_s: float = 0.0


class StageFailed(Exception):
    def __init__(self, stage: str, cause: BaseException):
        super().__init__(f"stage {stage!r} failed: {type(cause).__name__}: {cause}")
        self.stage = stage
        self.cause = cause


def run_job(
    job: Job,
    stages: Sequence[Stage],
    *,
    force: Collection[str] = (),
    clock: Callable[[], float] = time.monotonic,
) -> list[StageOutcome]:
    unknown = set(force) - {s.name for s in stages}
    if unknown:
        raise ValueError(f"unknown stage(s): {', '.join(sorted(unknown))}")

    outcomes: list[StageOutcome] = []
    upstream_ran = False  # once a stage runs, every later output is stale
    for stage in stages:
        if job.path(stage.output).exists() and stage.name not in force and not upstream_ran:
            job.append_log(stage.name, "skipped")
            outcomes.append(StageOutcome(stage.name, "skipped"))
            continue
        started = clock()
        try:
            details = stage.run(job) or {}
        except Exception as exc:
            job.append_log(
                stage.name, "error",
                duration_s=round(clock() - started, 3),
                error=f"{type(exc).__name__}: {exc}",
            )
            raise StageFailed(stage.name, exc) from exc
        duration = round(clock() - started, 3)
        job.append_log(stage.name, "ok", duration_s=duration, **details)
        outcomes.append(StageOutcome(stage.name, "ok", duration))
        upstream_ran = True
    return outcomes
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_transcribe_stage.py tests/test_orchestrator.py -v`
Expected: 8 passed

- [ ] **Step 6: Run the whole suite**

Run: `uv run pytest -v`
Expected: all passed

- [ ] **Step 7: Commit**

```bash
git add src/meeting_record/transcribe.py src/meeting_record/orchestrator.py tests/conftest.py tests/test_transcribe_stage.py tests/test_orchestrator.py
git commit -m "Add transcribe stage behind a Transcriber protocol and the stage orchestrator"
```

---

### Task 6: WhisperX engine with NB-Whisper and pyannote

**Files:**
- Create: `src/meeting_record/whisperx_engine.py`
- Test: `tests/test_whisperx_engine.py`, `tests/test_gpu_smoke.py`

**Interfaces:**
- Consumes: `Settings` from `meeting_record.config`; `EngineInfo`, `normalize_segments` from `meeting_record.transcript`; `TranscriptionResult` from `meeting_record.transcribe`; `Language` from `meeting_record.meeting`.
- Produces:
  - `meeting_record.whisperx_engine.WhisperXTranscriber(settings: Settings)`, which satisfies `Transcriber`. The constructor raises `RuntimeError` mentioning `HF_TOKEN` if the token is missing. `transcribe(...)` returns `timings_s` with keys `asr`, `align`, `diarize`.
  - `meeting_record.whisperx_engine.fetch_models(settings: Settings, languages: tuple[str, ...] = ("no", "en")) -> list[str]`, which downloads the ASR model, the alignment models for `languages`, and the diarization pipeline, and returns a name for each. Raises `RuntimeError` if `HF_TOKEN` is missing.
  - Importing this module must not import `torch` or `whisperx`; they are imported inside the functions.

This module is the only place WhisperX is imported. Its GPU path cannot run on the dev machine; the unit tests cover the import boundary and the token check, and `tests/test_gpu_smoke.py` covers the real path on the lab server in Task 10.

- [ ] **Step 1: Write the failing tests**

`tests/test_whisperx_engine.py`:

```python
import subprocess
import sys

import pytest

from meeting_record.config import Settings


def test_importing_the_engine_does_not_load_the_gpu_stack():
    code = (
        "import sys, meeting_record.whisperx_engine; "
        "print(any(m in sys.modules for m in ('torch', 'whisperx', 'pyannote')))"
    )
    out = subprocess.run([sys.executable, "-c", code], capture_output=True, text=True, check=True)
    assert out.stdout.strip() == "False"


def test_transcriber_requires_hf_token():
    from meeting_record.whisperx_engine import WhisperXTranscriber

    with pytest.raises(RuntimeError, match="HF_TOKEN"):
        WhisperXTranscriber(Settings.from_env({}))


def test_fetch_models_requires_hf_token():
    from meeting_record.whisperx_engine import fetch_models

    with pytest.raises(RuntimeError, match="HF_TOKEN"):
        fetch_models(Settings.from_env({}))
```

`tests/test_gpu_smoke.py`:

```python
"""Real NB-Whisper + pyannote run. Lab server only:

    source bin/env.sh && MR_GPU_TESTS=1 .venv/bin/pytest -m gpu -v
"""

import os
import urllib.request
from pathlib import Path

import pytest

from meeting_record.config import Settings

pytestmark = [
    pytest.mark.gpu,
    pytest.mark.skipif(os.environ.get("MR_GPU_TESTS") != "1", reason="set MR_GPU_TESTS=1 on the lab server"),
]

SAMPLE_URL = "https://github.com/NbAiLab/nb-whisper/raw/main/audio/knuthamsun.mp3"


@pytest.fixture(scope="module")
def sample_audio() -> Path:
    cache = Path.home() / ".cache" / "meeting-record-tests"
    cache.mkdir(parents=True, exist_ok=True)
    path = cache / "knuthamsun.mp3"
    if not path.exists():
        partial = path.with_suffix(".part")
        urllib.request.urlretrieve(SAMPLE_URL, partial)
        partial.replace(path)
    return path


def test_nb_whisper_transcribes_and_diarizes_the_norwegian_sample(sample_audio):
    from meeting_record.whisperx_engine import WhisperXTranscriber

    result = WhisperXTranscriber(Settings.from_env()).transcribe(
        sample_audio, language="no", min_speakers=1, max_speakers=2
    )

    assert result.segments
    with_speaker = sum(1 for s in result.segments if s.speaker)
    assert with_speaker / len(result.segments) >= 0.9
    assert sum(len(s.text.split()) for s in result.segments) > 50
    assert set(result.timings_s) == {"asr", "align", "diarize"}
    print("timings_s:", result.timings_s)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_whisperx_engine.py tests/test_gpu_smoke.py -v`
Expected: the three engine tests FAIL with `ModuleNotFoundError: No module named 'meeting_record.whisperx_engine'`; the smoke test is SKIPPED.

- [ ] **Step 3: Write the implementation**

`src/meeting_record/whisperx_engine.py`:

```python
"""NB-Whisper large through WhisperX, word alignment, pyannote diarization.

The only module that imports the GPU stack, and it does so inside functions so the
rest of the package (and its tests) run without the gpu extra. Models are loaded one
at a time and released before the next, because the RTX 3060 has 12 GB.
"""

from __future__ import annotations

import gc
import time
from importlib.metadata import version
from pathlib import Path

from meeting_record.config import Settings
from meeting_record.meeting import Language
from meeting_record.transcribe import TranscriptionResult
from meeting_record.transcript import EngineInfo, normalize_segments

_TOKEN_HELP = (
    "HF_TOKEN is not set. pyannote diarization needs a Hugging Face read token from an account "
    "that accepted the terms of the diarization model. Put HF_TOKEN=... in .env (see README, Setup)."
)


def _release_gpu(torch) -> None:
    gc.collect()
    if torch.cuda.is_available():
        torch.cuda.empty_cache()


class WhisperXTranscriber:
    def __init__(self, settings: Settings) -> None:
        if not settings.hf_token:
            raise RuntimeError(_TOKEN_HELP)
        self.settings = settings

    def transcribe(
        self,
        audio_path: Path,
        *,
        language: Language,
        min_speakers: int | None,
        max_speakers: int | None,
    ) -> TranscriptionResult:
        import torch
        import whisperx
        from whisperx.diarize import DiarizationPipeline

        s = self.settings
        timings: dict[str, float] = {}
        audio = whisperx.load_audio(str(audio_path))

        started = time.monotonic()
        asr = whisperx.load_model(s.whisper_model, s.device, compute_type=s.compute_type, language=language)
        result = asr.transcribe(audio, batch_size=s.batch_size, language=language)
        timings["asr"] = round(time.monotonic() - started, 1)
        del asr
        _release_gpu(torch)

        started = time.monotonic()
        align_model, align_metadata = whisperx.load_align_model(language_code=language, device=s.device)
        result = whisperx.align(
            result["segments"], align_model, align_metadata, audio, s.device, return_char_alignments=False
        )
        timings["align"] = round(time.monotonic() - started, 1)
        del align_model
        _release_gpu(torch)

        started = time.monotonic()
        diarizer = DiarizationPipeline(model_name=s.diarization_model, token=s.hf_token, device=s.device)
        turns = diarizer(audio, min_speakers=min_speakers, max_speakers=max_speakers)
        timings["diarize"] = round(time.monotonic() - started, 1)
        del diarizer
        _release_gpu(torch)

        result = whisperx.assign_word_speakers(turns, result)
        return TranscriptionResult(
            segments=normalize_segments(result["segments"]),
            engine=EngineInfo(
                name="whisperx",
                version=version("whisperx"),
                asr_model=s.whisper_model,
                compute_type=s.compute_type,
                batch_size=s.batch_size,
                diarization_model=s.diarization_model,
            ),
            timings_s=timings,
        )


def fetch_models(settings: Settings, languages: tuple[str, ...] = ("no", "en")) -> list[str]:
    """Download everything a run needs (about 11 GB) so the first Meeting doesn't wait on it."""
    if not settings.hf_token:
        raise RuntimeError(_TOKEN_HELP)
    import whisperx
    from faster_whisper.utils import download_model
    from whisperx.diarize import DiarizationPipeline

    ready = []
    download_model(settings.whisper_model)
    ready.append(settings.whisper_model)
    for language in languages:
        whisperx.load_align_model(language_code=language, device="cpu")
        ready.append(f"alignment model for {language!r}")
    DiarizationPipeline(model_name=settings.diarization_model, token=settings.hf_token, device="cpu")
    ready.append(settings.diarization_model)
    return ready
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_whisperx_engine.py tests/test_gpu_smoke.py -v`
Expected: 3 passed, 1 skipped

- [ ] **Step 5: Commit**

```bash
git add src/meeting_record/whisperx_engine.py tests/test_whisperx_engine.py tests/test_gpu_smoke.py
git commit -m "Add WhisperX engine running NB-Whisper large with pyannote diarization"
```

---

### Task 7: CLI, doctor, and the `bin/mr` wrapper

**Files:**
- Create: `src/meeting_record/doctor.py`, `src/meeting_record/cli.py`, `bin/env.sh`, `bin/mr`, `.env.example`
- Test: `tests/test_doctor.py`, `tests/test_cli.py`, `tests/test_bin_wrapper.py`

**Interfaces:**
- Consumes: everything above. Specifically `Settings`; `create_job`, `IngestError`; `open_job`, `JobNotFound`, `Job`, `RAW_TRANSCRIPT_FILE`, `RAW_TRANSCRIPT_TEXT_FILE`; `Attendee`; `Stage`, `StageFailed`, `run_job`; `Transcriber`, `transcribe_stage`; `WhisperXTranscriber`, `fetch_models` (imported lazily); fixtures `media`, `fake_transcriber`.
- Produces:
  - `meeting_record.doctor.Check(name: str, ok: bool, detail: str)`
  - `meeting_record.doctor.check_binaries(which=shutil.which) -> Check`, `check_jobs_dir(settings) -> Check`, `check_hf_token(settings) -> Check`, `check_gpu_stack(settings) -> list[Check]`, `run_checks(settings) -> list[Check]`
  - `meeting_record.doctor.report(checks: list[Check]) -> tuple[str, int]`: lines `ok    name: detail` or `FAIL  name: detail`, then a summary line; exit code 1 if any failed.
  - `meeting_record.cli.main(argv: list[str] | None = None, *, settings: Settings | None = None, make_transcriber: Callable[[Settings], Transcriber] = ..., out: TextIO | None = None) -> int`
  - `meeting_record.cli.build_stages(settings, make_transcriber) -> list[Stage]`
  - CLI commands (program name `mr`):
    - `mr new RECORDING --title T --date YYYY-MM-DD --language {no,nn,en} [--attendee "Name <email>"]... [--min-speakers N] [--max-speakers N] [--run]`
    - `mr run JOB_ID [--force STAGE]...`
    - `mr show JOB_ID`
    - `mr doctor`
    - `mr fetch-models`
  - `bin/env.sh` (sourceable in bash): exports `MR_ROOT`, loads `${MR_ENV_FILE:-$MR_ROOT/.env}`, prepends `site-packages/nvidia/*/lib` to `LD_LIBRARY_PATH`. `bin/mr` sources it and execs `.venv/bin/meeting-record`.

- [ ] **Step 1: Write the failing tests**

`tests/test_doctor.py`:

```python
import importlib.util
from pathlib import Path

import pytest

from meeting_record.config import Settings
from meeting_record.doctor import Check, check_binaries, check_gpu_stack, check_hf_token, check_jobs_dir, report


def test_report_formats_checks_and_fails_if_any_failed():
    text, code = report([Check("a", True, "fine"), Check("b", False, "broken")])
    assert text == "ok    a: fine\nFAIL  b: broken\n1 check(s) failed\n"
    assert code == 1


def test_report_passes_when_all_pass():
    text, code = report([Check("a", True, "fine")])
    assert text.endswith("all checks passed\n")
    assert code == 0


def test_check_binaries_names_what_is_missing():
    check = check_binaries(which=lambda name: None if name == "ffprobe" else f"/usr/bin/{name}")
    assert not check.ok
    assert "ffprobe" in check.detail


def test_check_jobs_dir_creates_a_writable_dir(tmp_path):
    settings = Settings.from_env({"MR_JOBS_DIR": str(tmp_path / "jobs")})
    check = check_jobs_dir(settings)
    assert check.ok
    assert (tmp_path / "jobs").is_dir()
    assert list((tmp_path / "jobs").iterdir()) == []


def test_check_jobs_dir_rejects_the_windows_drive():
    check = check_jobs_dir(Settings.from_env({"MR_JOBS_DIR": "/mnt/c/Users/lab/jobs"}))
    assert not check.ok
    assert "inside WSL" in check.detail


def test_check_hf_token():
    assert not check_hf_token(Settings.from_env({})).ok
    assert check_hf_token(Settings.from_env({"HF_TOKEN": "hf_x"})).ok


def test_gpu_stack_check_explains_a_missing_gpu_extra():
    if importlib.util.find_spec("torch") is not None:
        pytest.skip("gpu extra is installed here")
    [check] = check_gpu_stack(Settings.from_env({}))
    assert not check.ok
    assert "uv sync --extra gpu" in check.detail
```

`tests/test_cli.py`:

```python
import io

from meeting_record.cli import main
from meeting_record.config import Settings
from meeting_record.doctor import Check
from meeting_record.jobs import open_job


def cli(argv, tmp_path, transcriber):
    out = io.StringIO()
    settings = Settings.from_env({"MR_JOBS_DIR": str(tmp_path / "jobs")})
    code = main(argv, settings=settings, make_transcriber=lambda s: transcriber, out=out)
    return code, out.getvalue()


def new_args(media, *extra):
    return [
        "new", str(media / "tone.wav"),
        "--title", "Lab møte", "--date", "2026-09-10", "--language", "no",
        "--attendee", "Kari Nordmann <kari@example.org>", "--attendee", "Ola",
        "--max-speakers", "3", *extra,
    ]


def test_new_with_run_produces_a_diarized_transcript(tmp_path, media, fake_transcriber):
    code, text = cli(new_args(media, "--run"), tmp_path, fake_transcriber)

    assert code == 0
    assert "job 2026-09-10-lab-mote" in text
    assert "transcribe: ok" in text
    job = open_job(tmp_path / "jobs", "2026-09-10-lab-mote")
    assert [a.email for a in job.meta().attendees] == ["kari@example.org", None]
    assert fake_transcriber.calls[0]["max_speakers"] == 3

    code, text = cli(["show", job.id], tmp_path, fake_transcriber)
    assert code == 0
    assert "[00:00:00] SPEAKER_00: Vi flytter fristen til fredag." in text


def test_new_without_run_does_not_transcribe(tmp_path, media, fake_transcriber, capsys):
    code, _ = cli(new_args(media), tmp_path, fake_transcriber)
    assert code == 0
    assert fake_transcriber.calls == []

    code, _ = cli(["show", "2026-09-10-lab-mote"], tmp_path, fake_transcriber)
    assert code == 1
    assert "mr run 2026-09-10-lab-mote" in capsys.readouterr().err


def test_run_skips_finished_stage_unless_forced(tmp_path, media, fake_transcriber):
    cli(new_args(media, "--run"), tmp_path, fake_transcriber)

    code, text = cli(["run", "2026-09-10-lab-mote"], tmp_path, fake_transcriber)
    assert code == 0
    assert "transcribe: skipped" in text
    assert len(fake_transcriber.calls) == 1

    code, text = cli(["run", "2026-09-10-lab-mote", "--force", "transcribe"], tmp_path, fake_transcriber)
    assert code == 0
    assert "transcribe: ok" in text
    assert len(fake_transcriber.calls) == 2


def test_unknown_job_is_an_error(tmp_path, fake_transcriber, capsys):
    code, _ = cli(["run", "no-such-job"], tmp_path, fake_transcriber)
    assert code == 1
    assert "no job" in capsys.readouterr().err


def test_bad_recording_is_an_error(tmp_path, fake_transcriber, capsys):
    code, _ = cli(
        ["new", str(tmp_path / "missing.m4a"), "--title", "x", "--date", "2026-09-10", "--language", "en"],
        tmp_path, fake_transcriber,
    )
    assert code == 1
    assert "not found" in capsys.readouterr().err


def test_gpu_out_of_memory_prints_what_to_try(tmp_path, media, capsys):
    class OutOfMemory:
        def transcribe(self, *args, **kwargs):
            raise RuntimeError("CUDA failed with error out of memory")

    code, _ = cli(new_args(media, "--run"), tmp_path, OutOfMemory())
    assert code == 1
    err = capsys.readouterr().err
    assert "stage 'transcribe' failed" in err
    assert "MR_BATCH_SIZE=4" in err


def test_doctor_prints_report_and_exit_code(tmp_path, fake_transcriber, monkeypatch):
    monkeypatch.setattr("meeting_record.cli.run_checks", lambda s: [Check("a", True, "fine"), Check("b", False, "broken")])
    code, text = cli(["doctor"], tmp_path, fake_transcriber)
    assert code == 1
    assert "FAIL  b: broken" in text
```

`tests/test_bin_wrapper.py`:

```python
import os
import subprocess
from pathlib import Path

ROOT = Path(__file__).resolve().parents[1]


def test_mr_wrapper_runs_the_cli():
    proc = subprocess.run([str(ROOT / "bin" / "mr"), "--help"], capture_output=True, text=True)
    assert proc.returncode == 0
    assert "doctor" in proc.stdout


def test_mr_wrapper_loads_the_env_file(tmp_path):
    jobs = tmp_path / "wrapper-jobs"
    env_file = tmp_path / "test.env"
    env_file.write_text(f"MR_JOBS_DIR={jobs}\n", encoding="utf-8")
    env = {**os.environ, "MR_ENV_FILE": str(env_file)}
    env.pop("MR_JOBS_DIR", None)

    proc = subprocess.run([str(ROOT / "bin" / "mr"), "doctor"], capture_output=True, text=True, env=env)

    assert str(jobs) in proc.stdout
    assert jobs.is_dir()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_doctor.py tests/test_cli.py tests/test_bin_wrapper.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'meeting_record.doctor'` / `'meeting_record.cli'`, and the wrapper tests fail because `bin/mr` does not exist.

- [ ] **Step 3: Write `doctor.py`**

`src/meeting_record/doctor.py`:

```python
"""`mr doctor`: is this machine ready to transcribe? Each failure says what to do."""

from __future__ import annotations

import ctypes
import shutil
from collections.abc import Callable
from dataclasses import dataclass

from meeting_record.config import Settings

MIN_FREE_VRAM_BYTES = 8 * 2**30


@dataclass(frozen=True)
class Check:
    name: str
    ok: bool
    detail: str


def check_binaries(which: Callable[[str], str | None] = shutil.which) -> Check:
    missing = [name for name in ("ffmpeg", "ffprobe") if which(name) is None]
    if missing:
        return Check("ffmpeg", False, f"missing: {', '.join(missing)} (sudo apt install ffmpeg)")
    return Check("ffmpeg", True, "found")


def check_jobs_dir(settings: Settings) -> Check:
    path = settings.jobs_dir
    if str(path).startswith("/mnt/"):
        return Check("jobs dir", False, f"{path} is on the Windows drive; use a path inside WSL (MR_JOBS_DIR)")
    try:
        path.mkdir(parents=True, exist_ok=True)
        probe = path / ".write-test"
        probe.write_text("ok", encoding="utf-8")
        probe.unlink()
    except OSError as exc:
        return Check("jobs dir", False, f"{path}: {exc}")
    return Check("jobs dir", True, str(path))


def check_hf_token(settings: Settings) -> Check:
    if settings.hf_token:
        return Check("HF_TOKEN", True, "set")
    return Check("HF_TOKEN", False, "not set: add HF_TOKEN=... to .env")


def _check_diarization_access(settings: Settings) -> Check:
    name = "pyannote access"
    if not settings.hf_token:
        return Check(name, False, "skipped: HF_TOKEN not set")
    from huggingface_hub import auth_check
    from huggingface_hub.errors import GatedRepoError, RepositoryNotFoundError

    try:
        auth_check(settings.diarization_model, token=settings.hf_token)
    except GatedRepoError:
        return Check(
            name, False,
            f"accept the terms at https://huggingface.co/{settings.diarization_model} "
            "with the account that owns HF_TOKEN",
        )
    except RepositoryNotFoundError:
        return Check(name, False, f"{settings.diarization_model} not found, or HF_TOKEN is invalid")
    except OSError as exc:
        return Check(name, False, f"could not reach huggingface.co: {exc}")
    return Check(name, True, settings.diarization_model)


def check_gpu_stack(settings: Settings) -> list[Check]:
    try:
        import ctranslate2
        import torch
    except ImportError as exc:
        return [Check("gpu extra", False, f"{exc.name} is not installed: run 'uv sync --extra gpu'")]

    if not torch.cuda.is_available():
        return [
            Check("torch CUDA", False,
                  "no CUDA device visible: update the Windows NVIDIA driver, then run 'wsl --shutdown' in Windows")
        ]
    free, total = torch.cuda.mem_get_info()
    gib = 2**30
    checks = [
        Check("torch CUDA", True, f"{torch.cuda.get_device_name(0)}, {free / gib:.1f} of {total / gib:.1f} GiB free"),
        Check(
            "GPU memory",
            free >= MIN_FREE_VRAM_BYTES,
            "enough free for NB-Whisper large" if free >= MIN_FREE_VRAM_BYTES
            else f"only {free / gib:.1f} GiB free: close GPU apps on Windows, or set MR_COMPUTE_TYPE=int8",
        ),
    ]
    devices = ctranslate2.get_cuda_device_count()
    checks.append(Check("ctranslate2 CUDA", devices > 0, f"{devices} device(s)"))
    try:
        ctypes.CDLL("libcudnn_ops.so.9")
        checks.append(Check("cuDNN 9", True, "libcudnn_ops.so.9 loads"))
    except OSError as exc:
        checks.append(Check("cuDNN 9", False, f"{exc}: start the tool through bin/mr, which sets LD_LIBRARY_PATH"))
    checks.append(_check_diarization_access(settings))
    return checks


def run_checks(settings: Settings) -> list[Check]:
    return [check_binaries(), check_jobs_dir(settings), check_hf_token(settings), *check_gpu_stack(settings)]


def report(checks: list[Check]) -> tuple[str, int]:
    lines = [f"{'ok  ' if c.ok else 'FAIL'}  {c.name}: {c.detail}" for c in checks]
    failed = sum(1 for c in checks if not c.ok)
    lines.append(f"{failed} check(s) failed" if failed else "all checks passed")
    return "\n".join(lines) + "\n", (1 if failed else 0)
```

- [ ] **Step 4: Write `cli.py`**

`src/meeting_record/cli.py`:

```python
"""Command line for build step 1: register a Meeting, run its Job, read the Transcript."""

from __future__ import annotations

import argparse
import datetime as dt
import sys
from collections.abc import Callable, Collection
from pathlib import Path
from typing import TextIO

from meeting_record.config import Settings
from meeting_record.doctor import report, run_checks
from meeting_record.ingest import IngestError, create_job
from meeting_record.jobs import RAW_TRANSCRIPT_FILE, RAW_TRANSCRIPT_TEXT_FILE, Job, JobNotFound, open_job
from meeting_record.meeting import Attendee
from meeting_record.orchestrator import Stage, StageFailed, run_job
from meeting_record.transcribe import Transcriber, transcribe_stage

TranscriberFactory = Callable[[Settings], Transcriber]

OOM_HINT = (
    "The GPU ran out of memory. In .env, try these one at a time: MR_BATCH_SIZE=4, then "
    "MR_COMPUTE_TYPE=int8, then MR_WHISPER_MODEL=NbAiLab/nb-whisper-medium. "
    "Then re-run with: mr run JOB_ID --force transcribe"
)


def _whisperx_transcriber(settings: Settings) -> Transcriber:
    from meeting_record.whisperx_engine import WhisperXTranscriber

    return WhisperXTranscriber(settings)


def build_stages(settings: Settings, make_transcriber: TranscriberFactory) -> list[Stage]:
    def transcribe(job: Job) -> dict:
        return transcribe_stage(job, make_transcriber(settings))

    return [Stage(name="transcribe", output=RAW_TRANSCRIPT_FILE, run=transcribe)]


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="mr", description="Meeting Record: Recording in, diarized Transcript out.")
    sub = parser.add_subparsers(dest="command", required=True)

    new = sub.add_parser("new", help="register a concluded Meeting from its Recording")
    new.add_argument("recording", type=Path, help="audio or video file, e.g. /mnt/c/Users/lab/Desktop/meeting.m4a")
    new.add_argument("--title", required=True)
    new.add_argument("--date", required=True, type=dt.date.fromisoformat, help="YYYY-MM-DD")
    new.add_argument("--language", required=True, choices=["no", "nn", "en"], help="main spoken language")
    new.add_argument("--attendee", action="append", default=[], metavar='"NAME <EMAIL>"')
    new.add_argument("--min-speakers", type=int)
    new.add_argument("--max-speakers", type=int)
    new.add_argument("--run", action="store_true", help="transcribe right away")

    run = sub.add_parser("run", help="run the Job's remaining stages")
    run.add_argument("job_id")
    run.add_argument("--force", action="append", default=[], metavar="STAGE", help="re-run STAGE and what follows")

    show = sub.add_parser("show", help="print the Job's Transcript")
    show.add_argument("job_id")

    sub.add_parser("doctor", help="check that this machine can transcribe")
    sub.add_parser("fetch-models", help="download all models ahead of the first Meeting")
    return parser


def _run(job: Job, settings: Settings, make_transcriber: TranscriberFactory,
         force: Collection[str], out: TextIO) -> int:
    for outcome in run_job(job, build_stages(settings, make_transcriber), force=force):
        suffix = f" ({outcome.duration_s:.1f}s)" if outcome.status == "ok" else ""
        print(f"{outcome.stage}: {outcome.status}{suffix}", file=out)
    print(f"transcript: {job.path(RAW_TRANSCRIPT_TEXT_FILE)}", file=out)
    return 0


def _dispatch(args: argparse.Namespace, settings: Settings, make_transcriber: TranscriberFactory,
              out: TextIO) -> int:
    if args.command == "new":
        job = create_job(
            settings.jobs_dir,
            args.recording,
            title=args.title,
            date=args.date,
            language=args.language,
            attendees=[Attendee.parse(a) for a in args.attendee],
            min_speakers=args.min_speakers,
            max_speakers=args.max_speakers,
        )
        print(f"job {job.id}", file=out)
        print(f"  {job.dir}", file=out)
        return _run(job, settings, make_transcriber, (), out) if args.run else 0

    if args.command == "run":
        job = open_job(settings.jobs_dir, args.job_id)
        return _run(job, settings, make_transcriber, args.force, out)

    if args.command == "show":
        job = open_job(settings.jobs_dir, args.job_id)
        text_path = job.path(RAW_TRANSCRIPT_TEXT_FILE)
        if not text_path.is_file():
            print(f"error: job {job.id} has no Transcript yet; run: mr run {job.id}", file=sys.stderr)
            return 1
        out.write(text_path.read_text(encoding="utf-8"))
        return 0

    if args.command == "doctor":
        text, code = report(run_checks(settings))
        out.write(text)
        return code

    if args.command == "fetch-models":
        from meeting_record.whisperx_engine import fetch_models

        for name in fetch_models(settings):
            print(f"ready: {name}", file=out)
        return 0

    raise AssertionError(f"unhandled command {args.command!r}")


def main(
    argv: list[str] | None = None,
    *,
    settings: Settings | None = None,
    make_transcriber: TranscriberFactory = _whisperx_transcriber,
    out: TextIO | None = None,
) -> int:
    out = out or sys.stdout
    args = build_parser().parse_args(argv)
    settings = settings or Settings.from_env()
    try:
        return _dispatch(args, settings, make_transcriber, out)
    except StageFailed as exc:
        print(f"error: {exc}", file=sys.stderr)
        if "out of memory" in str(exc.cause).lower():
            print(OOM_HINT, file=sys.stderr)
        return 1
    except (IngestError, JobNotFound, ValueError, RuntimeError) as exc:
        print(f"error: {exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
```

`project.scripts` expects `main` to return the exit code; the console-script shim calls `sys.exit(main())`, so returning an int is correct.

- [ ] **Step 5: Write the wrapper scripts and `.env.example`**

`bin/env.sh`:

```bash
# Source this file from bash to get the environment the pipeline needs:
#   - settings from .env (or from $MR_ENV_FILE)
#   - the pip-installed CUDA libraries (cuDNN, cuBLAS) on the loader path;
#     ctranslate2 does not find them inside the venv on its own.
# Example: source bin/env.sh && MR_GPU_TESTS=1 .venv/bin/pytest -m gpu

MR_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
export MR_ROOT

if [[ ! -x "$MR_ROOT/.venv/bin/python" ]]; then
  echo "error: $MR_ROOT/.venv is missing; run 'uv sync --extra gpu' in $MR_ROOT" >&2
  return 1
fi

_mr_env_file="${MR_ENV_FILE:-$MR_ROOT/.env}"
if [[ -f "$_mr_env_file" ]]; then
  set -a
  # shellcheck disable=SC1090
  source "$_mr_env_file"
  set +a
fi

_mr_site="$("$MR_ROOT/.venv/bin/python" -c 'import sysconfig; print(sysconfig.get_path("purelib"))')"
_mr_libs=""
for _mr_dir in "$_mr_site"/nvidia/*/lib; do
  if [[ -d "$_mr_dir" ]]; then
    _mr_libs="${_mr_libs:+$_mr_libs:}$_mr_dir"
  fi
done
if [[ -n "$_mr_libs" ]]; then
  export LD_LIBRARY_PATH="$_mr_libs${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
fi
unset _mr_env_file _mr_site _mr_libs _mr_dir
```

`bin/mr`:

```bash
#!/usr/bin/env bash
# Meeting Record CLI with the right environment. Usage: bin/mr --help
set -euo pipefail
# shellcheck source=bin/env.sh
source "$(dirname "${BASH_SOURCE[0]}")/env.sh"
exec "$MR_ROOT/.venv/bin/meeting-record" "$@"
```

`.env.example`:

```bash
# Copy to .env next to this file. Never commit .env. bin/mr loads it.

# Hugging Face read token from the account that accepted the terms at
# https://huggingface.co/pyannote/speaker-diarization-community-1
HF_TOKEN=

# Where Job directories live. Keep it inside WSL, never under /mnt/c.
# MR_JOBS_DIR=/home/meeting/meeting-record/jobs

# If the GPU runs out of memory, change these one at a time.
# MR_BATCH_SIZE=8
# MR_COMPUTE_TYPE=float16
# MR_WHISPER_MODEL=NbAiLab/nb-whisper-large
```

Run: `chmod +x bin/mr`

- [ ] **Step 6: Run tests to verify they pass**

Run: `uv run pytest tests/test_doctor.py tests/test_cli.py tests/test_bin_wrapper.py -v`
Expected: all passed (7 doctor, 7 CLI, 2 wrapper)

- [ ] **Step 7: Run the whole suite**

Run: `uv run pytest -v`
Expected: all passed, 1 skipped (the GPU smoke test)

- [ ] **Step 8: Commit**

```bash
git add src/meeting_record/doctor.py src/meeting_record/cli.py bin/env.sh bin/mr .env.example tests/test_doctor.py tests/test_cli.py tests/test_bin_wrapper.py
git commit -m "Add mr CLI (new, run, show, doctor, fetch-models) and CUDA-aware wrapper"
```

---

### Task 8: WSL bootstrap script

**Files:**
- Create: `scripts/wsl/bootstrap.sh`
- Test: `tests/test_setup_scripts.py`

**Interfaces:**
- Consumes: `bin/mr doctor` and `bin/mr fetch-models` from Task 7; `uv.lock` and the `gpu` extra from Task 1; `.env` format from `.env.example`.
- Produces: `scripts/wsl/bootstrap.sh [--user NAME] [--repo URL] [--branch NAME] [--source DIR] [--skip-gpu-check] [--skip-models]`, run as root inside the WSL distro. Reads `HF_TOKEN` from its environment. Exit codes: 0 success, 1 failure (including not root), 2 bad arguments. Task 9 calls it with `--user`, `--repo`, `--branch`, and optionally `--skip-models`, passing `HF_TOKEN` via `WSLENV`.

What it does, in order, each step safe to repeat:

1. Install apt packages: `ffmpeg git curl ca-certificates rsync sudo`.
2. Create the app user if missing (no sudo rights; the owner uses `wsl -u root` for admin).
3. Under WSL, add `[user] default=<user>` to `/etc/wsl.conf` if no `[user]` section exists, keeping Ubuntu's existing `[boot] systemd=true`.
4. Install uv for the app user if missing.
5. Get the code into `/home/<user>/automated-meetings`: copy from `--source` if given, else `git pull --ff-only` an existing clone, else clone.
6. `uv sync --frozen --extra gpu` as the app user.
7. Write `HF_TOKEN` into `.env` (mode 600) when provided, keeping other lines.
8. Unless `--skip-gpu-check`: confirm `nvidia-smi` works inside WSL, then run `bin/mr doctor`.
9. Unless `--skip-models`: run `bin/mr fetch-models`.

- [ ] **Step 1: Write the failing test**

`tests/test_setup_scripts.py`:

```python
import os
import subprocess
from pathlib import Path

import pytest

ROOT = Path(__file__).resolve().parents[1]
BOOTSTRAP = ROOT / "scripts" / "wsl" / "bootstrap.sh"


def bootstrap(*args):
    return subprocess.run(["bash", str(BOOTSTRAP), *args], capture_output=True, text=True)


def test_bootstrap_has_unix_line_endings():
    assert b"\r" not in BOOTSTRAP.read_bytes()


def test_bootstrap_help_exits_zero():
    proc = bootstrap("--help")
    assert proc.returncode == 0
    assert "--skip-models" in proc.stdout


def test_bootstrap_rejects_unknown_arguments():
    proc = bootstrap("--bogus")
    assert proc.returncode == 2
    assert "unknown argument" in proc.stderr


@pytest.mark.skipif(os.geteuid() == 0, reason="checks the non-root refusal")
def test_bootstrap_refuses_to_run_without_root():
    proc = bootstrap("--skip-gpu-check", "--skip-models")
    assert proc.returncode == 1
    assert "run as root" in proc.stderr
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_setup_scripts.py -v`
Expected: FAIL (`bootstrap.sh` does not exist: `FileNotFoundError` on `read_bytes`, and bash exits 127 for the others)

- [ ] **Step 3: Write the implementation**

`scripts/wsl/bootstrap.sh`:

```bash
#!/usr/bin/env bash
# Prepares Ubuntu inside WSL2 on the lab server to run the Meeting Record pipeline.
# Run as root; scripts/windows/setup.ps1 does this for you.
# Safe to re-run: every step checks the current state before changing it.
set -euo pipefail

LINUX_USER="meeting"
REPO_URL="https://github.com/adamaske/automated-meetings.git"
BRANCH="master"
SOURCE_DIR=""
SKIP_GPU_CHECK=0
SKIP_MODELS=0

usage() {
  cat <<'USAGE'
usage: bootstrap.sh [--user NAME] [--repo URL] [--branch NAME] [--source DIR]
                    [--skip-gpu-check] [--skip-models]

  --user NAME        Linux user that owns and runs the app (default: meeting)
  --repo URL         git repository to clone (default: the public GitHub repo)
  --branch NAME      branch to check out (default: master)
  --source DIR       copy code from DIR instead of cloning (for testing a local checkout)
  --skip-gpu-check   do not require a GPU, and skip 'mr doctor'
  --skip-models      do not download models (about 11 GB)

If HF_TOKEN is set in the environment it is written to the app's .env file.
USAGE
}

while [[ $# -gt 0 ]]; do
  case "$1" in
    --user) LINUX_USER="$2"; shift 2 ;;
    --repo) REPO_URL="$2"; shift 2 ;;
    --branch) BRANCH="$2"; shift 2 ;;
    --source) SOURCE_DIR="$2"; shift 2 ;;
    --skip-gpu-check) SKIP_GPU_CHECK=1; shift ;;
    --skip-models) SKIP_MODELS=1; shift ;;
    -h|--help) usage; exit 0 ;;
    *) echo "unknown argument: $1" >&2; usage >&2; exit 2 ;;
  esac
done

log() { printf '\n==> %s\n' "$*"; }
die() { printf 'error: %s\n' "$*" >&2; exit 1; }

[[ $EUID -eq 0 ]] || die "run as root (the Windows setup script does this for you)"

USER_HOME="/home/$LINUX_USER"
APP_DIR="$USER_HOME/automated-meetings"
UV="$USER_HOME/.local/bin/uv"

as_user() { sudo -u "$LINUX_USER" -H bash -c "$1"; }

log "Installing system packages"
export DEBIAN_FRONTEND=noninteractive
apt-get update -q
apt-get install -y -q ffmpeg git curl ca-certificates rsync sudo

log "Ensuring user '$LINUX_USER'"
if ! id -u "$LINUX_USER" >/dev/null 2>&1; then
  useradd --create-home --shell /bin/bash "$LINUX_USER"
fi

if grep -qi microsoft /proc/version 2>/dev/null; then
  log "Setting WSL default user"
  if grep -q '^\[user\]' /etc/wsl.conf 2>/dev/null; then
    if ! grep -qx "default=$LINUX_USER" /etc/wsl.conf; then
      echo "warning: /etc/wsl.conf already has a [user] section; set default=$LINUX_USER there by hand" >&2
    fi
  else
    printf '\n[user]\ndefault=%s\n' "$LINUX_USER" >> /etc/wsl.conf
  fi
fi

log "Ensuring uv"
if [[ ! -x "$UV" ]]; then
  as_user 'curl -LsSf https://astral.sh/uv/install.sh | sh'
fi

log "Getting the code into $APP_DIR"
if [[ -n "$SOURCE_DIR" ]]; then
  [[ -d "$SOURCE_DIR" ]] || die "--source $SOURCE_DIR is not a directory"
  mkdir -p "$APP_DIR"
  rsync -a --delete --exclude .venv --exclude .env --exclude jobs "$SOURCE_DIR"/ "$APP_DIR"/
  chown -R "$LINUX_USER:$LINUX_USER" "$APP_DIR"
elif [[ -d "$APP_DIR/.git" ]]; then
  as_user "git -C '$APP_DIR' fetch origin '$BRANCH' && git -C '$APP_DIR' checkout '$BRANCH' && git -C '$APP_DIR' pull --ff-only origin '$BRANCH'"
else
  as_user "git clone --branch '$BRANCH' '$REPO_URL' '$APP_DIR'"
fi

log "Installing Python dependencies (first run downloads several GB)"
as_user "cd '$APP_DIR' && '$UV' sync --frozen --extra gpu"

ENV_FILE="$APP_DIR/.env"
if [[ -n "${HF_TOKEN:-}" ]]; then
  log "Writing HF_TOKEN to $ENV_FILE"
  (
    umask 077
    touch "$ENV_FILE"
    grep -v '^HF_TOKEN=' "$ENV_FILE" > "$ENV_FILE.tmp" || true
    printf 'HF_TOKEN=%s\n' "$HF_TOKEN" >> "$ENV_FILE.tmp"
    mv "$ENV_FILE.tmp" "$ENV_FILE"
  )
  chown "$LINUX_USER:$LINUX_USER" "$ENV_FILE"
  chmod 600 "$ENV_FILE"
elif [[ ! -f "$ENV_FILE" ]]; then
  echo "warning: no HF_TOKEN given and no $ENV_FILE; diarization will not run until you add one" >&2
fi

if [[ $SKIP_GPU_CHECK -eq 0 ]]; then
  log "Checking the GPU from inside WSL"
  NVIDIA_SMI="$(command -v nvidia-smi || echo /usr/lib/wsl/lib/nvidia-smi)"
  "$NVIDIA_SMI" --query-gpu=name,driver_version,memory.total --format=csv,noheader \
    || die "GPU not visible inside WSL: update the Windows NVIDIA driver, run 'wsl --shutdown' in Windows, re-run setup"
  log "Running mr doctor"
  as_user "'$APP_DIR/bin/mr' doctor" || die "mr doctor reported problems (see above)"
fi

if [[ $SKIP_MODELS -eq 0 ]]; then
  log "Downloading models (about 11 GB, once)"
  as_user "'$APP_DIR/bin/mr' fetch-models"
fi

log "Done"
cat <<DONE
Open the app shell from Windows with:   wsl -d <distro> -u $LINUX_USER
Then try it:
  cd ~/automated-meetings
  bin/mr new /mnt/c/Users/<you>/Desktop/meeting.m4a --title "Lab meeting" --date $(date +%F) --language no --run
  bin/mr show <job id printed above>
DONE
```

The repo's default branch is `master`, so the default `BRANCH` is `master`.

- [ ] **Step 4: Run tests and shellcheck**

Run: `uv run pytest tests/test_setup_scripts.py -v`
Expected: 4 passed

Run: `uvx --from shellcheck-py shellcheck scripts/wsl/bootstrap.sh bin/env.sh bin/mr`
Expected: no output, exit 0. Fix anything it reports before committing.

- [ ] **Step 5: Commit**

```bash
git add scripts/wsl/bootstrap.sh tests/test_setup_scripts.py
git commit -m "Add WSL bootstrap: packages, app user, uv, code, token, GPU check, models"
```

---

### Task 9: Windows setup script

**Files:**
- Create: `scripts/windows/setup.ps1`
- Modify: `tests/test_setup_scripts.py` (append static checks)

**Interfaces:**
- Consumes: `scripts/wsl/bootstrap.sh` and its arguments from Task 8.
- Produces: `scripts/windows/setup.ps1 [-Distro Ubuntu-24.04] [-LinuxUser meeting] [-RepoUrl URL] [-Branch master] [-SkipModels]`, run from an elevated Windows PowerShell 5.1 or 7. Exit codes: 0 done, 1 failure with a message, 3010 restart Windows and re-run.

What it does, in order, each step safe to repeat:

1. Refuse to run unelevated (`#Requires -RunAsAdministrator`). Print which Windows account it runs as, because WSL distros are per account.
2. Preflight: Windows 11 (build 22000 or later), winget present, warn if firmware virtualization looks off.
3. NVIDIA driver: `nvidia-smi` present and driver 570.65 or newer. winget cannot install it, so the script stops with the download link when it is missing or old.
4. Enable the `VirtualMachinePlatform` Windows feature if needed.
5. winget-install `Microsoft.WSL` if missing.
6. If steps 4 or 5 need a restart, stop with exit 3010.
7. `wsl --update`, `wsl --set-default-version 2`.
8. Create `%USERPROFILE%\.wslconfig` with a memory limit of 75% of RAM (at least 8 GB) only when the file does not exist.
9. Install Ubuntu 24.04 with `wsl --install --no-launch --web-download` if it is not registered.
10. Ask for the Hugging Face token (hidden input; Enter keeps the existing one) unless `HF_TOKEN` is already set.
11. Find `bootstrap.sh` next to the script, or download it from GitHub for the chosen branch. Refuse it if it has CRLF line endings.
12. Run it as root in the distro with `wsl --exec`, passing `HF_TOKEN` through `WSLENV`, then restart the distro so the default user applies.

- [ ] **Step 1: Append the failing static checks**

Append to `tests/test_setup_scripts.py`:

```python
import shutil

SETUP = ROOT / "scripts" / "windows" / "setup.ps1"


def test_setup_requires_admin_and_calls_bootstrap():
    text = SETUP.read_text(encoding="utf-8-sig")
    assert text.startswith("#Requires -RunAsAdministrator")
    assert r"..\wsl\bootstrap.sh" in text
    assert "scripts/wsl/bootstrap.sh" in text
    assert "[version]'570.65'" in text
    assert "Microsoft.WSL" in text


def test_setup_passes_only_arguments_bootstrap_accepts():
    setup = SETUP.read_text(encoding="utf-8-sig")
    help_text = bootstrap("--help").stdout
    for flag in ("--user", "--repo", "--branch", "--skip-models"):
        assert f"'{flag}'" in setup
        assert flag in help_text


@pytest.mark.skipif(shutil.which("pwsh") is None, reason="PowerShell not installed here; parsed on the lab server")
def test_setup_parses_in_powershell():
    command = (
        "$errors = $null; "
        f"[void][System.Management.Automation.Language.Parser]::ParseFile('{SETUP}', [ref]$null, [ref]$errors); "
        "if ($errors) { $errors | ForEach-Object { $_.ToString() }; exit 1 }"
    )
    proc = subprocess.run(["pwsh", "-NoProfile", "-Command", command], capture_output=True, text=True)
    assert proc.returncode == 0, proc.stdout + proc.stderr
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_setup_scripts.py -v`
Expected: the two new setup tests FAIL with `FileNotFoundError` for `setup.ps1`; the parse test is skipped unless `pwsh` exists.

- [ ] **Step 3: Write the implementation**

`scripts/windows/setup.ps1`:

```powershell
#Requires -RunAsAdministrator
<#
.SYNOPSIS
  Sets up the Windows 11 lab server to run the Meeting Record pipeline in WSL2.

.DESCRIPTION
  Safe to re-run: each step checks the current state before changing anything.
    1. Preflight: Windows 11, winget, CPU virtualization.
    2. NVIDIA driver 570.65 or newer (checked only; winget has no NVIDIA driver package).
    3. Virtual Machine Platform feature and the WSL package (winget).
    4. WSL update, WSL 2 default, memory limit in .wslconfig if none exists.
    5. Ubuntu 24.04.
    6. Inside Ubuntu, as root: scripts/wsl/bootstrap.sh (packages, app user, uv, code,
       Hugging Face token, GPU check, model download).

  Run it from the Windows account that will use the tool: WSL distributions belong
  to one Windows account.

.EXAMPLE
  powershell -ExecutionPolicy Bypass -File scripts\windows\setup.ps1
#>
[CmdletBinding()]
param(
    [string]$Distro = 'Ubuntu-24.04',
    [string]$LinuxUser = 'meeting',
    [string]$RepoUrl = 'https://github.com/adamaske/automated-meetings.git',
    [string]$Branch = 'master',
    [switch]$SkipModels
)

$ErrorActionPreference = 'Stop'
$env:WSL_UTF8 = '1'  # make wsl.exe print UTF-8 instead of UTF-16

$MinDriver = [version]'570.65'
$WingetPackages = @('Microsoft.WSL')
# winget: 0x8A150109 restart required to finish, 0x8A15010A restart required to install,
# 0x8A15002B no applicable upgrade (already installed).
$WingetRestartCodes = @(-1978334967, -1978334966)
$WingetAlreadyInstalledCodes = @(-1978335189)
$script:RestartNeeded = $false

function Write-Step([string]$Text) {
    Write-Host ''
    Write-Host "==> $Text" -ForegroundColor Cyan
}

function Stop-Setup([string]$Text, [int]$Code = 1) {
    Write-Host "error: $Text" -ForegroundColor Red
    exit $Code
}

function Test-Preflight {
    Write-Host "Running as Windows account: $env:USERDOMAIN\$env:USERNAME"
    $build = [Environment]::OSVersion.Version.Build
    if ($build -lt 22000) { Stop-Setup "Windows 11 is required (this is build $build)." }
    if (-not (Get-Command winget -ErrorAction SilentlyContinue)) {
        Stop-Setup 'winget was not found. Install "App Installer" from the Microsoft Store, then re-run.'
    }
    $cpu = Get-CimInstance Win32_Processor | Select-Object -First 1
    if ($cpu.VirtualizationFirmwareEnabled -eq $false) {
        Write-Warning 'Windows reports CPU virtualization as disabled. If WSL fails to start, enable Intel VT-x or AMD-V (SVM) in the BIOS/UEFI.'
    }
}

function Test-NvidiaDriver {
    $smi = Get-Command nvidia-smi -ErrorAction SilentlyContinue
    if (-not $smi) {
        Stop-Setup "nvidia-smi was not found. Install the NVIDIA Game Ready or Studio driver $MinDriver or newer for the RTX 3060 from https://www.nvidia.com/Download/index.aspx, restart, then re-run. Do not install any NVIDIA driver inside WSL."
    }
    $line = & $smi.Source --query-gpu=name,driver_version,memory.total --format=csv,noheader | Select-Object -First 1
    $name, $driver, $memory = $line -split ',\s*'
    if ([version]$driver -lt $MinDriver) {
        Stop-Setup "NVIDIA driver $driver is older than $MinDriver, which CUDA 12.8 needs. Update it from https://www.nvidia.com/Download/index.aspx, then re-run."
    }
    Write-Host "GPU: $name, driver $driver, $memory"
}

function Enable-VirtualMachinePlatform {
    $feature = Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
    if ($feature.State -eq 'Enabled') {
        Write-Host 'Virtual Machine Platform already enabled'
        return
    }
    $result = Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All -NoRestart
    if ($result.RestartNeeded) { $script:RestartNeeded = $true }
}

function Install-WingetPackage([string]$Id) {
    & winget list --id $Id --exact --accept-source-agreements | Out-Null
    if ($LASTEXITCODE -eq 0) {
        Write-Host "$Id already installed"
        return
    }
    & winget install --id $Id --exact --silent --accept-package-agreements --accept-source-agreements
    if ($LASTEXITCODE -eq 0 -or $WingetAlreadyInstalledCodes -contains $LASTEXITCODE) { return }
    if ($WingetRestartCodes -contains $LASTEXITCODE) {
        $script:RestartNeeded = $true
        return
    }
    Stop-Setup "winget failed to install $Id (exit code $LASTEXITCODE)."
}

function Initialize-Wsl {
    & wsl.exe --update
    if ($LASTEXITCODE -ne 0) { Stop-Setup "wsl --update failed (exit code $LASTEXITCODE)." }
    & wsl.exe --set-default-version 2 | Out-Null
    if ($LASTEXITCODE -ne 0) { Stop-Setup "wsl --set-default-version 2 failed (exit code $LASTEXITCODE)." }
}

function Set-WslMemory {
    $path = Join-Path $env:USERPROFILE '.wslconfig'
    if (Test-Path $path) {
        Write-Host "$path exists; leaving it unchanged. Transcription wants at least 16 GB for WSL."
        return
    }
    $totalGb = [math]::Floor((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB)
    $wslGb = [math]::Max(8, [math]::Floor($totalGb * 0.75))
    Set-Content -Path $path -Encoding ASCII -Value "[wsl2]`r`nmemory=${wslGb}GB`r`n"
    Write-Host "Wrote $path with memory=${wslGb}GB"
}

function Get-InstalledDistros {
    # No 2>$null here: with ErrorActionPreference Stop, Windows PowerShell 5.1 turns redirected native stderr into a terminating error.
    $names = & wsl.exe --list --quiet
    return @($names | ForEach-Object { $_.Trim() } | Where-Object { $_ })
}

function Install-Distro {
    if ((Get-InstalledDistros) -contains $Distro) {
        Write-Host "$Distro already installed"
        return
    }
    & wsl.exe --install --distribution $Distro --no-launch --web-download
    if ($LASTEXITCODE -ne 0) { Stop-Setup "wsl --install $Distro failed (exit code $LASTEXITCODE)." }
    if (-not ((Get-InstalledDistros) -contains $Distro)) {
        Stop-Setup "$Distro was installed but is not registered yet. Run 'wsl -d $Distro' once, create any user when asked, type 'exit', then re-run this script."
    }
}

function Read-HfToken {
    if ($env:HF_TOKEN) { return $env:HF_TOKEN }
    Write-Host 'Speaker diarization needs a Hugging Face read token:'
    Write-Host '  1. Sign in at https://huggingface.co and accept the terms at'
    Write-Host '     https://huggingface.co/pyannote/speaker-diarization-community-1'
    Write-Host '  2. Create a read token at https://huggingface.co/settings/tokens'
    $secure = Read-Host -AsSecureString 'Paste the token (input hidden; press Enter to keep an existing one)'
    $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secure)
    try { return [Runtime.InteropServices.Marshal]::PtrToStringBSTR($bstr) }
    finally { [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr) }
}

function Get-BootstrapScript {
    if ($PSScriptRoot) {
        $local = Join-Path $PSScriptRoot '..\wsl\bootstrap.sh'
        if (Test-Path $local) { return (Resolve-Path $local).Path }
    }
    $raw = ($RepoUrl -replace '^https://github\.com/', 'https://raw.githubusercontent.com/') -replace '\.git$', ''
    $dest = Join-Path $env:TEMP 'meeting-record-bootstrap.sh'
    Write-Host "Downloading $raw/$Branch/scripts/wsl/bootstrap.sh"
    Invoke-WebRequest -UseBasicParsing -Uri "$raw/$Branch/scripts/wsl/bootstrap.sh" -OutFile $dest
    return $dest
}

function Invoke-Bootstrap([string]$Token) {
    $windowsPath = Get-BootstrapScript
    if ([IO.File]::ReadAllText($windowsPath).Contains("`r")) {
        Stop-Setup "$windowsPath has Windows line endings. Re-download or re-clone the repository; its .gitattributes keeps .sh files LF."
    }
    $linuxPath = (& wsl.exe -d $Distro -u root --exec wslpath -a ($windowsPath -replace '\\', '/')).Trim()
    if ($LASTEXITCODE -ne 0 -or -not $linuxPath) { Stop-Setup "Could not translate $windowsPath to a WSL path." }

    $bootstrapArgs = @('--user', $LinuxUser, '--repo', $RepoUrl, '--branch', $Branch)
    if ($SkipModels) { $bootstrapArgs += '--skip-models' }

    $previousWslEnv = $env:WSLENV
    try {
        if ($Token) {
            $env:HF_TOKEN = $Token
            $env:WSLENV = if ($previousWslEnv) { "$previousWslEnv`:HF_TOKEN/u" } else { 'HF_TOKEN/u' }
        }
        & wsl.exe -d $Distro -u root --exec bash $linuxPath @bootstrapArgs
        $code = $LASTEXITCODE
    }
    finally {
        Remove-Item Env:HF_TOKEN -ErrorAction SilentlyContinue
        $env:WSLENV = $previousWslEnv
    }
    if ($code -ne 0) { Stop-Setup "bootstrap.sh failed inside $Distro (exit code $code). Fix the error above and re-run; finished steps are skipped." }
    & wsl.exe --terminate $Distro | Out-Null  # so the new default user applies next start
}

Write-Step 'Preflight'
Test-Preflight

Write-Step 'NVIDIA driver'
Test-NvidiaDriver

Write-Step 'Windows features and packages'
Enable-VirtualMachinePlatform
foreach ($id in $WingetPackages) { Install-WingetPackage $id }
if ($script:RestartNeeded) {
    Stop-Setup 'Restart Windows to finish enabling WSL, then run this script again.' 3010
}

Write-Step 'WSL'
Initialize-Wsl
Set-WslMemory

Write-Step "Linux distribution $Distro"
Install-Distro

Write-Step 'Hugging Face token'
$token = Read-HfToken

Write-Step "Setting up inside $Distro"
Invoke-Bootstrap $token

Write-Step 'Done'
Write-Host "Open the app shell with:  wsl -d $Distro -u $LinuxUser"
Write-Host 'Then:  cd ~/automated-meetings; bin/mr doctor'
exit 0
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_setup_scripts.py -v`
Expected: 6 passed, 1 skipped (PowerShell parse) on a machine without `pwsh`; 7 passed with it.

- [ ] **Step 5: Commit**

```bash
git add scripts/windows/setup.ps1 tests/test_setup_scripts.py
git commit -m "Add elevated Windows setup script: driver check, winget WSL, Ubuntu, bootstrap"
```

---

### Task 10: Docs, then the demo on the lab server

**Files:**
- Modify: `README.md`, `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md` (section 3 job layout)
- Create: `.scratch/meeting-record/build-step-1-acceptance.md` (results only, no meeting content)

**Interfaces:**
- Consumes: everything above.
- Produces: the build step 1 demo, run for real on the lab server, with timings and outcomes written down.

- [ ] **Step 1: Update the README**

In `README.md`, replace the line `Status: design phase. Nothing runs yet.` with:

```markdown
Status: build step 1. A Recording goes in, a diarized Transcript comes out, on the lab server.
```

Append to `README.md`:

````markdown
## Setup on the lab server (Windows 11)

Needs: Windows 11, the NVIDIA driver 570.65 or newer for the RTX 3060, a Hugging Face account.

1. Sign in at https://huggingface.co, accept the terms at https://huggingface.co/pyannote/speaker-diarization-community-1, and create a read token at https://huggingface.co/settings/tokens.
2. Download this repository (Code → Download ZIP) and extract it, or `git clone` it.
3. Open PowerShell as administrator, from the Windows account that will use the tool, in the extracted folder, and run:

   ```powershell
   powershell -ExecutionPolicy Bypass -File scripts\windows\setup.ps1
   ```

   If it asks you to restart Windows, restart and run the same command again. Every run skips what is already done. The first full run downloads about 20 GB of packages and models.

The app lives inside WSL at `/home/meeting/automated-meetings`. Settings are in `.env` there; see `.env.example`.

## Demo: Recording in, diarized Transcript out

Copy the Recording from the Meeting laptop to the server's Desktop, then open the app shell:

```powershell
wsl -d Ubuntu-24.04 -u meeting
```

```bash
cd ~/automated-meetings
bin/mr doctor
bin/mr new /mnt/c/Users/<you>/Desktop/meeting.m4a \
  --title "Lab meeting" --date 2026-09-11 --language no \
  --attendee "Kari Nordmann <kari@example.org>" --max-speakers 6 --run
bin/mr show 2026-09-11-lab-meeting
```

The Job directory is `~/meeting-record/jobs/<job id>/`: `meeting.json`, the audio, `transcript.raw.json`, `transcript.raw.txt`, and `log.jsonl` with timings.

## Troubleshooting

- **Out of GPU memory.** In `.env`, set `MR_BATCH_SIZE=4`. If it still fails, add `MR_COMPUTE_TYPE=int8`, then `MR_WHISPER_MODEL=NbAiLab/nb-whisper-medium`. Re-run with `bin/mr run <job id> --force transcribe`.
- **`libcudnn_ops.so.9` cannot be loaded.** Start the tool through `bin/mr`, or `source bin/env.sh` first; it puts the bundled CUDA libraries on the loader path.
- **pyannote access fails in `mr doctor`.** Accept the model terms with the same Hugging Face account that created `HF_TOKEN`.
- **No CUDA device inside WSL.** Update the Windows NVIDIA driver, run `wsl --shutdown` in Windows, and open the shell again. Never install an NVIDIA driver inside WSL.

## Development

```bash
uv sync
uv run pytest
```

The GPU smoke test runs only on the lab server: `source bin/env.sh && MR_GPU_TESTS=1 .venv/bin/pytest -m gpu -v`.
````

- [ ] **Step 2: Update the design doc's job layout**

In `docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md`, section 3, replace these three lines of the job tree:

```
  meeting.json          # metadata: title, start time, attendees (name+email), language, audio path
  audio.<ext>
  transcript.raw.json   # STT output: segments with speaker label, start/end, text, confidence
```

with:

```
  meeting.json          # metadata: title, date, language, attendees (name+email), speaker bounds, recording (file, sha256, duration)
  audio.<ext>           # the Recording as it arrived; audio.flac (mono 16 kHz) when the Recording had video
  transcript.raw.json   # normalized STT output: segments (id s0001.., start/end, speaker label, text, confidence, words)
  transcript.raw.txt    # the same, one line per segment, for reading
```

In the same section, after the sentence ending "retries on transient failures.", add:

```
Build step 1 has no retries: its only stage is local GPU work and fails fast. Retries arrive with the first API-calling stage.
```

- [ ] **Step 3: Run the full suite and commit the docs**

Run: `uv run pytest -v`
Expected: all passed; skipped only for the GPU smoke test and, without `pwsh`, the PowerShell parse test.

```bash
git add README.md docs/superpowers/specs/2026-09-11-meeting-summary-pipeline-design.md
git commit -m "Document lab server setup, the transcription demo, and the job layout"
git push
```

- [ ] **Step 4: Run setup on the lab server** (manual, on the server)

In elevated PowerShell in the repo folder: `powershell -ExecutionPolicy Bypass -File scripts\windows\setup.ps1`.
Expected: steps print `==>` headings; `nvidia-smi` from inside WSL lists the RTX 3060; `mr doctor` ends with `all checks passed`; `fetch-models` prints `ready:` for NB-Whisper large, the `no` and `en` alignment models, and the pyannote pipeline. If it stops for a restart (exit 3010), restart and re-run.

Also run the PowerShell parse check there, since the dev machine has no PowerShell:

```powershell
$e = $null; [void][System.Management.Automation.Language.Parser]::ParseFile((Resolve-Path scripts\windows\setup.ps1), [ref]$null, [ref]$e); $e
```

Expected: no output.

- [ ] **Step 5: Re-run setup to prove it is safe to repeat** (manual)

Run the same setup command again.
Expected: every step reports it is already done or skips its change, bootstrap pulls with nothing new, exit code 0.

- [ ] **Step 6: Run the GPU smoke test** (manual, in the app shell)

```bash
cd ~/automated-meetings
source bin/env.sh && MR_GPU_TESTS=1 .venv/bin/pytest -m gpu -v -s
```

Expected: 1 passed, with `timings_s` printed. If it fails with out-of-memory, apply the troubleshooting order in the README and note which setting was needed.

- [ ] **Step 7: Run the demo on a real Recording** (manual)

Use a consented lab Recording of about 30 to 60 minutes. Follow the README demo commands.
Expected: `transcribe: ok`, then `bin/mr show` prints lines like `[00:03:12] SPEAKER_01: ...` with more than one Speaker label for a multi-person Meeting.

If a Recording with both English and Norwegian is available, run it too with `--language no` and read the English stretches: are they transcribed in English or translated into Norwegian?

- [ ] **Step 8: Write down the results and commit** (manual)

Create `.scratch/meeting-record/build-step-1-acceptance.md` with no Transcript text, names, or quotes:

```markdown
# Build step 1 acceptance on the lab server

Date:
Windows build:
NVIDIA driver:
WSL version (wsl --version):
Did `wsl --install --no-launch` register Ubuntu directly, or was the manual first launch needed:
Setup re-run clean (yes/no):

| Recording | Length (min) | Language | Speakers expected / found | ASR (s) | Align (s) | Diarize (s) | Settings changed from default |
|---|---|---|---|---|---|---|---|
| sample (Knut Hamsun) | | no | 1 / | | | | |
| lab Recording 1 | | | | | | | |

Out-of-memory seen (yes/no, and which setting fixed it):
Mixed-language result (English kept / translated / not tested):
Anything that surprised us:
```

```bash
git add .scratch/meeting-record/build-step-1-acceptance.md
git commit -m "Record build step 1 acceptance results from the lab server"
git push
```
