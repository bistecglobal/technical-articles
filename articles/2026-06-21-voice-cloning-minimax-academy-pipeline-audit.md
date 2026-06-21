---
article: 2026-06-21-voice-cloning-minimax-academy-pipeline.md
auditor: bistec-articles audit skill
date: 2026-06-21
verdict: PASS (with fixes applied)
score: 22/25
---

# Audit Report — Voice Cloning at Scale

## Verdict: PASS — fixes applied

Score: **22/25**. Two factual errors corrected before publish.

---

## Claim Inventory

| # | Claim | Evidence | Verdict |
|---|---|---|---|
| 1 | Four curriculum tracks | `academy-audio-narrator.md` lists architect, engineer, sse, techlead | ✅ |
| 2 | "Twenty-two tracks multiplies that into weeks" | WRONG — 22 was IRP weeks (`git log`: "Industry Readiness Program weeks 1-22"), not 22 separate tracks | ❌ Fixed |
| 3 | Pipeline stages: SESSION.md → HTML → validate → SPEAKER-NOTES → AUDIO-NARRATIVE → WAV → MP4 | `academy-video/SKILL.md` pipeline diagram | ✅ |
| 4 | Kokoro = preset voices, no voice cloning | `academy-audio-narrator.md` backend table | ✅ |
| 5 | NeuTTS Air commit `62e1333`, 0.7B Qwen GGUF, ~12 sec ref | `git show 62e1333` commit message | ✅ |
| 6 | Supertonic-3 commit `b963e3d`, 31-language ONNX, no zero-shot cloning | `supertonic_to_speech.py` docstring, commit `b963e3d` | ✅ |
| 7 | "NeuTTS in particular produced narration that drifted" | WRONG — NeuTTS was added TO FIX drift in MLX/Qwen3 (`62e1333`: "default backends produce voice clones that drift") | ❌ Fixed |
| 8 | CPU local models 20–30 min per deck | SKILL.md: "TTS: ~20-30 min per deck" | ✅ |
| 9 | Can't run local models in parallel (CPU/RAM contention) | `academy-audio-narrator.md`: "TTS is CPU-bound... Do not spawn multiple TTS processes" | ✅ |
| 10 | MiniMax commit `6e2ea3a` | `git show 6e2ea3a` confirmed | ✅ |
| 11 | Upload via `POST /v1/files/upload` | `minimax_to_speech.py:139` | ✅ |
| 12 | Clone via `POST /v1/voice_clone`, model `speech-2.8-hd`, async | `minimax_to_speech.py:165-176` | ✅ |
| 13 | Poll: status 2054 = not ready, 0 = provisioned | `minimax_to_speech.py:199-200` | ✅ |
| 14 | Cache key = SHA256(path + mtime)[:16] in `~/.mmx/voice_cache.json` | `minimax_to_speech.py:91-94` | ✅ |
| 15 | TTS via `POST /v1/t2a_v2`, hex output | `minimax_to_speech.py:221-241` | ✅ |
| 16 | 10,000-char T2A limit, 8,000-char chunk size | `MAX_CHUNK_CHARS = 8000  # well under the 10k T2A limit` | ✅ |
| 17 | Chunk at paragraph boundaries | `chunk_text()` in `minimax_to_speech.py:285-302` | ✅ |
| 18 | 1-second silence between slides | `minimax_to_speech.py:386: all_samples.extend([0.0] * SAMPLE_RATE)  # 1 sec silence` | ✅ |
| 19 | `--slide N` resume flag | `minimax_to_speech.py` argparse `--slide` flag, `start_slide` param | ✅ |
| 20 | Two source formats: AUDIO-NARRATIVE.md vs SPEAKER-NOTES.md | `extract_narrative_sections()` vs `extract_speech_text()` in `minimax_to_speech.py` | ✅ |
| 21 | Markdown stripped before TTS | `re.sub()` calls in `extract_narrative_sections()` | ✅ |
| 22 | Higgsfield path commit `83963a9` | `git show 83963a9` confirmed | ✅ |
| 23 | Four production paths in SKILL.md | `academy-video/SKILL.md` lines 9-15 | ✅ |
| 24 | Academy-audio-narrator agent at `.claude/agents/academy-audio-narrator.md` | File confirmed at that path | ✅ |

---

## Forward-Looking Scan

No matches for "will", "plan to", "coming soon", "in the future", "next step", "roadmap", "soon". **Clean.**

---

## Rubric Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality (claims cited) | 4/5 | 2 errors found and fixed; all others directly code-cited |
| Technical depth | 5/5 | API flow, caching logic, chunking, parsing all covered with code |
| Clarity for target audience | 4/5 | Pipeline context well established |
| BistecGlobal voice | 4/5 | Professional, practitioner-focused |
| Title specificity | 5/5 | Specific, accurate, descriptive |

**Total: 22/25 — PASS**

---

## Fixes Applied

### Fix 1 — "Twenty-two tracks" → corrected to "twenty-two weeks"

The IRP program had 22 weeks of content (commit: `feat(academy): Industry Readiness Program weeks 1-22 decks`), not 22 tracks. Active curriculum tracks are four (engineer, architect, senior-engineer, tech-lead). Intro reworded to remove the false count.

### Fix 2 — NeuTTS drift misattribution corrected

Original text: *"NeuTTS in particular produced narration that drifted in voice identity across long slide sequences."*

Commit `62e1333` shows NeuTTS was added to fix drift in MLX/Qwen3 backends: *"default backends produce voice clones that drift, sound robotic, or mispronounce English."* NeuTTS was the solution, not the source of the problem. Text rewritten to correctly attribute drift to earlier local backends (MLX, Qwen3).
