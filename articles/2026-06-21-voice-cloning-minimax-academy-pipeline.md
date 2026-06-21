---
title: "Voice Cloning at Scale: MiniMax TTS Backend for AI-Generated Training Content"
project: academy-videos
tags: [AI, TTS, VoiceCloning, DevTools, LearningPlatform]
status: audited
date: 2026-06-21
---

# Voice Cloning at Scale: MiniMax TTS Backend for AI-Generated Training Content

When Bistec Academy started producing narrated video content at scale — across four curriculum tracks, up to six months of material each — the bottleneck wasn't the slides or the scripts. It was the voice.

Recording narration manually is linear work. One slide deck takes an afternoon. A program spanning twenty-two weeks of curriculum multiplies that into months. The engineering team needed a way to generate narrator audio that matched the trainer's voice, ran without a recording studio, and could be restarted mid-deck when an API request failed. This post covers how they built it.

## The Pipeline Context

The academy video pipeline is a multi-stage system that converts a session plan (`SESSION.md`) into a narrated MP4. The stages are:

```
SESSION.md → HTML slides → validate → SPEAKER-NOTES.md → AUDIO-NARRATIVE.md → WAV → MP4
```

The TTS step — converting `AUDIO-NARRATIVE.md` into per-slide WAV files — was the stage with the most failure modes. Early backends included local models:

- **Kokoro** (`notes_to_speech.py`): fast ONNX inference with preset voices (no voice cloning)
- **NeuTTS Air** (`neutts_to_speech.py`, commit `62e1333`): 0.7B Qwen-based GGUF model with ~12-second reference clip cloning, CPU-only
- **Supertonic-3** (`supertonic_to_speech.py`, commit `b963e3d`): 31-language ONNX model, fast but without zero-shot cloning
- **MLX / Qwen3**: Apple Silicon local inference paths

Each of these had a specific failure mode at scale. The earliest local backends (MLX, Qwen3) produced voice clones that drifted in identity across long slide sequences and mispronounced English technical terms — NeuTTS Air (`62e1333`) was added specifically to address this. But all local models shared a deeper problem: they tied up the machine for 20–30 minutes per deck, couldn't run in parallel safely (they competed for cores and RAM), and required a venv-activated local environment to run at all.

The MiniMax backend (commit `6e2ea3a`) moved the heavy work to an API.

## Voice Cloning Once, Running Forever

The central design decision in `minimax_to_speech.py` is that voice cloning is a one-time setup step, not a per-run cost. The flow:

1. **Upload** the reference audio (10 seconds to 5 minutes of WAV) via `POST /v1/files/upload`
2. **Clone** the voice: `POST /v1/voice_clone` with `model: speech-2.8-hd`, which kicks off an async provisioning job
3. **Poll** the T2A endpoint until the voice is ready (status code 2054 = not yet provisioned; 0 = ready)
4. **Cache** the resulting `voice_id` in `~/.mmx/voice_cache.json`, keyed by `SHA256(path + mtime)`

```python
def _cache_key(ref_audio_path: str) -> str:
    mtime = os.path.getmtime(ref_audio_path)
    raw = f"{ref_audio_path}:{mtime}"
    return hashlib.sha256(raw.encode()).hexdigest()[:16]
```

On every subsequent run, the script checks this cache before touching the API. If the reference file changes (different recording, updated mtime), the key changes and a new clone is triggered. If it's the same file, the cached `voice_id` is reused — skipping the upload and provisioning entirely.

This matters for decks with 12–18 slides. With the voice pre-provisioned, each slide's narration is a single `POST /v1/t2a_v2` call returning hex-encoded audio.

## Synthesis and Chunking

MiniMax's T2A v2 API has a 10,000-character limit per request. The script chunks text at 8,000 characters, breaking at paragraph boundaries first:

```python
MAX_CHUNK_CHARS = 8000  # well under the 10k T2A limit

def chunk_text(text: str, max_chars: int = MAX_CHUNK_CHARS) -> list[str]:
    paragraphs = text.split("\n\n")
    chunks = []
    current = ""
    for para in paragraphs:
        para = para.strip()
        if not para:
            continue
        if len(current) + len(para) + 2 > max_chars:
            if current:
                chunks.append(current.strip())
            current = para
        else:
            current = current + "\n\n" + para if current else para
    if current.strip():
        chunks.append(current.strip())
    return chunks
```

Each chunk is synthesized to hex, decoded, and concatenated in-memory using `soundfile`. The resulting per-slide WAV is written to `<deck>/audio/<slide-name>.wav`, then a combined track with 1-second silence between slides is also written.

The `--slide N` flag lets a run resume from a specific slide rather than restarting from the beginning. This is important when a network timeout drops one chunk mid-deck — there's no need to re-synthesize the first ten slides.

## Parsing AUDIO-NARRATIVE.md

The script supports two source formats. `AUDIO-NARRATIVE.md` is clean prose, one `## Slide N` section per slide. `SPEAKER-NOTES.md` is a facilitator guide that mixes presentation cues, discussion prompts, and spoken content — the script calls into `notes_to_speech._extract_spoken_parts()` to strip the non-spoken sections before sending to the API.

```python
def extract_narrative_sections(markdown: str) -> list[tuple[str, str]]:
    slide_blocks = re.split(r"^## (Slide \d+[^\n]*)", markdown, flags=re.MULTILINE)
    sections = []
    for i in range(1, len(slide_blocks), 2):
        name = slide_blocks[i].strip().rstrip("—:- ")
        body = slide_blocks[i + 1] if i + 1 < len(slide_blocks) else ""
        body = re.sub(r"\*{1,3}([^*]+)\*{1,3}", r"\1", body)
        body = re.sub(r"`([^`]+)`", r"\1", body)
        body = re.sub(r"\[([^\]]+)\]\([^)]+\)", r"\1", body)
        body = body.strip()
        if body:
            sections.append((name, body))
    return sections
```

Markdown formatting — bold, code spans, links — is stripped before synthesis to avoid the model reading them aloud.

## Adding Generative Video Clips

Alongside the MiniMax backend, commit `83963a9` added a fourth production path to the pipeline: Higgsfield-based generative video. This isn't TTS — it's AI-generated motion clips (using Seedance 2.0 for video, GPT Image 2 for stills) that slot into the final composite as intro/outro sequences, B-roll, or visual metaphors for abstract concepts.

The academy SKILL.md now documents four production paths:

- **Academy path**: SESSION.md → HTML → TTS → MP4
- **PPTX-direct path**: existing deck → PNG renders → TTS → MP4
- **Animated path** (Revideo): motion canvas scenes for kinetic diagrams
- **Generative path** (Higgsfield): photoreal clips generated from prompts, spliced in

The guidance is explicit: use generative clips only for moments where motion adds value. The default remains static slides with TTS narration; the Higgsfield path is an opt-in overlay.

## What Changed

Before these commits, every narrator audio run required a local model, a venv activation, and a machine that was otherwise idle. Identity consistency across long tracks required careful reference clip selection and re-runs. The MiniMax backend shifts the consistency guarantee to the API — the same `voice_id` produces consistent output across any number of runs, on any machine with credentials.

The practical outcome: decks that previously required a full local machine run can now be regenerated as a background API call. The resume flag means a dropped API call doesn't require restarting from slide 1.

The academy pipeline is in active production at Bistec, with tracks across engineer, architect, senior engineer, and tech lead curricula. The TTS backend selection (MiniMax, NeuTTS, Kokoro, Supertonic) is documented in the `academy-audio-narrator` agent manifest at `.claude/agents/academy-audio-narrator.md`, giving operators a single reference for choosing the right backend per use case.

---

*Implementation: [`minimax_to_speech.py`](https://github.com/bistecglobal/process-docs), [`minimax_clone_voice.py`](https://github.com/bistecglobal/process-docs) — commits `6e2ea3a`, `83963a9`*
