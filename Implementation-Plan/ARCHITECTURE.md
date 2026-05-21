# Pitch Practice — Architecture and Implementation Plan

ABOUTME: Architecture and phased build plan for an AI-driven SE pitch role-play tool.
ABOUTME: Hand this to Claude Code as the design spec for v1 prototype implementation.

## 1. Goal and scope

Build a turn-based pitch role-play prototype where an Okta SE can practice a sales conversation against a single AI-driven buyer persona. The SE speaks (push-to-talk), optionally shares a current slide image, and the persona responds in voice and text, staying in character throughout a 20–50 turn session.

**Success criterion for v1**: Joe can run a 30-minute mock pitch against the mid-market SaaS CISO persona, find the experience realistic enough to surface real training feedback, and identify what the persona schema or wrapper prompt needs to be tightened.

### In scope (v1)

- Single persona (mid-market SaaS CISO; YAML already drafted)
- Turn-based push-to-talk, no interruption, no streaming
- Local STT, local TTS, cloud LLM, cloud vision
- Streamlit UI on `localhost`
- Session transcript export to markdown

### Explicitly out of scope (deferred to v2+)

- Real-time conversational mode (interruption, VAD turn-taking)
- Avatar / talking-head video
- Coaching or scoring layer
- Multi-persona switching in the UI (the data structure supports it; the UI keeps it simple for v1)
- Per-persona voice cloning (one stock voice for v1)
- Multi-user deployment, authentication, persistence
- Web hosting

## 2. Architecture overview

### Component diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Streamlit UI (app.py)                          │
│   ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│   │ Audio recorder  │  │ Slide uploader   │  │ Conversation display     │  │
│   └────────┬────────┘  └────────┬─────────┘  └──────────────────────────┘  │
│            │                    │                                           │
│            ▼                    ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                       RolePlaySession.send()                        │  │
│   │           (the TDD'd core — pure, no UI dependencies)               │  │
│   └────────┬──────────────────┬────────────────────────┬────────────────┘  │
│            │                  │                        │                    │
│            ▼                  ▼                        ▼                    │
│   ┌────────────────┐  ┌──────────────────┐  ┌─────────────────────────┐    │
│   │ STT adapter    │  │ Claude API       │  │ TTS adapter             │    │
│   │ (faster-whisper│  │ (Sonnet 4.6,     │  │ (Kokoro or similar,     │    │
│   │  local, GPU)   │  │  cloud, vision)  │  │  local, GPU)            │    │
│   └────────────────┘  └──────────────────┘  └─────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Per-turn data flow

1. SE clicks push-to-talk, speaks 5–30 seconds, releases.
2. Streamlit captures audio bytes (WAV/WebM).
3. `WhisperTranscriber.transcribe(audio_bytes) -> str` runs on GPU, returns transcript.
4. SE optionally uploads the current slide image (PNG/JPEG).
5. Streamlit calls `session.send(transcript, slide_image_bytes)`.
6. `RolePlaySession`:
   - Appends the user turn to history (with image, if present)
   - Builds the Anthropic messages list from history
   - Builds the system prompt from wrapper template + persona YAML (cached)
   - Calls `client.messages.create(...)` against `claude-sonnet-4-6`
   - Appends persona turn to history
   - Returns response text
7. `TTSVoice.synthesize(response_text) -> bytes` runs locally, returns audio.
8. Streamlit displays the text and plays the audio.

### Why this shape

The `RolePlaySession` class is the testable boundary. It takes text in and returns text in a synchronous call. STT and TTS are adapters with simple interfaces. The Streamlit UI is presentation only — it never talks to Claude or to the persona YAML directly.

This shape makes the v2 migration to real-time mechanical: swap synchronous STT/TTS adapters for streaming ones, swap `session.send()` for `session.send_streaming()`, swap Streamlit for a real-time UI framework. Persona definitions, wrapper prompt, message construction logic, and history management all carry over unchanged.

## 3. Core abstraction — `RolePlaySession`

This is the heart of the system. Most of the TDD effort lives here.

### API

```python
# pitch_practice/session.py

from dataclasses import dataclass, field
from datetime import datetime
from typing import Literal
from anthropic import Anthropic

from pitch_practice.persona import Persona
from pitch_practice.prompts import build_system_prompt


@dataclass
class Turn:
    """One turn in a role-play session."""
    speaker: Literal["user", "persona"]
    text: str
    timestamp: datetime
    slide_image: bytes | None = None  # only set on user turns
    slide_media_type: str | None = None  # e.g. "image/png"


class RolePlaySession:
    """A turn-based role-play conversation against a persona.

    Stateful: holds history across send() calls.
    Synchronous: send() blocks until the persona responds.
    """

    def __init__(
        self,
        persona: Persona,
        wrapper_template: str,
        client: Anthropic,
        model: str = "claude-sonnet-4-6",
        max_response_tokens: int = 1024,
    ):
        self._persona = persona
        self._system_prompt = build_system_prompt(wrapper_template, persona)
        self._client = client
        self._model = model
        self._max_response_tokens = max_response_tokens
        self._history: list[Turn] = []

    def send(
        self,
        transcript: str,
        slide_image: bytes | None = None,
        slide_media_type: str | None = None,
    ) -> str:
        """Send the user's turn, return the persona's response text."""
        ...

    @property
    def history(self) -> list[Turn]:
        """Read-only view of the conversation history."""
        return list(self._history)

    def export_transcript(self) -> str:
        """Render a markdown transcript of the session."""
        ...
```

### Key implementation notes

- **Image format for Claude API**: each user turn with a slide becomes a multi-part content block: `[{"type": "image", "source": {"type": "base64", "media_type": "...", "data": "..."}}, {"type": "text", "text": transcript}]`. User turns without slides are plain text strings.
- **No streaming in v1**: synchronous `messages.create`. v2 will add `messages.stream()` support behind a different method.
- **No retry/backoff logic in v1**: keep it simple. Anthropic SDK has its own retry; we accept its defaults.
- **History grows unbounded**: a 30-minute session is well within Claude's context. We do not need to summarize or truncate for v1.
- **The system prompt is built once at session init**: persona is immutable for a session's lifetime.

## 4. Module layout

```
pitch-practice/
├── pyproject.toml                 # uv-managed dependencies
├── README.md                      # setup + run instructions
├── .env.example                   # ANTHROPIC_API_KEY
├── .gitignore
├── pitch_practice/
│   ├── __init__.py
│   ├── persona.py                 # Persona dataclass, YAML loader, validation
│   ├── prompts.py                 # wrapper template loading + substitution
│   ├── session.py                 # RolePlaySession (the TDD'd core)
│   ├── stt.py                     # WhisperTranscriber adapter
│   ├── tts.py                     # TTSVoice adapter
│   ├── cli.py                     # CLI entry: text-in / text-out for early validation
│   └── app.py                     # Streamlit UI
├── personas/
│   └── midmarket-saas-ciso.yaml   # the reference persona (already drafted)
├── prompts/
│   └── role_play_wrapper.md       # the wrapper template (already drafted)
├── voices/                        # TTS voice references (gitignored if large)
└── tests/
    ├── conftest.py                # shared fixtures
    ├── fixtures/
    │   ├── sample_audio.wav
    │   └── sample_slide.png
    ├── test_persona.py
    ├── test_prompts.py
    ├── test_session_unit.py       # message construction, history, transcript
    ├── test_session_integration.py  # real API; pytest mark for gating
    ├── test_stt.py                # GPU-marked
    └── test_tts.py                # GPU-marked
```

## 5. Dependencies

### Python packages

| Package | Purpose | Notes |
|---|---|---|
| `anthropic` | Claude SDK | Pin to current minor; the prototype uses `messages.create` only |
| `pydantic` | Persona validation | v2 |
| `pyyaml` | YAML loading | |
| `faster-whisper` | Local STT | Uses CTranslate2; CUDA wheels |
| `streamlit` | UI | |
| `python-dotenv` | env loading | |
| TTS engine | Local speech synthesis | **See note below** |
| `pytest`, `pytest-mock` | Testing | `pytest-mock` only for non-API mocks (filesystem, etc.) |

### TTS engine selection

The local TTS landscape moves fast. Recommended evaluation order at implementation time (Claude Code should verify current state):

1. **Kokoro TTS** — small (~82M params), fast, high quality for size, fixed voice set. Best v1 default if still well-maintained.
2. **Piper** — ONNX-based, very fast, CPU-friendly, fixed voices. Fallback if Kokoro has setup friction.
3. **XTTS-v2 (Coqui fork)** — heavier but supports voice cloning. Defer to v2 when adding per-persona voices.

The `TTSVoice` adapter interface is engine-agnostic, so swapping later costs nothing.

### Hardware requirements

GPU memory footprint at runtime (for sizing comfort, not constraints):

| Component | VRAM |
|---|---|
| `faster-whisper` large-v3 | ~4 GB |
| Kokoro TTS | <2 GB |
| XTTS-v2 (if used) | ~5 GB |

A 24–32 GB single-GPU machine has plenty of headroom. Whisper and TTS can coexist with room to spare.

CUDA setup: install CUDA Toolkit version matching PyTorch/CTranslate2 requirements. `faster-whisper` documents required cuDNN versions — follow their README.

## 6. Phased build plan

Each phase ends in a runnable, validated state. Do not skip a phase or merge phases.

### Phase 1 — Persona, prompt, session core (no audio, no UI)

**Build**:
- `pitch_practice/persona.py` — `Persona` pydantic model matching the YAML schema, plus `load_persona(path) -> Persona`
- `pitch_practice/prompts.py` — `load_wrapper_template(path) -> str`, `build_system_prompt(template, persona) -> str` (substitutes `{{persona}}` with YAML-rendered persona)
- `pitch_practice/session.py` — `Turn`, `RolePlaySession` with `send(text)` only (no images yet), `history`, `export_transcript`
- `pitch_practice/cli.py` — simple REPL: print persona name, loop on stdin, call `session.send(line)`, print response

**Tests (TDD these first)**:
- `test_persona.py`: loads valid YAML; raises on missing required fields; preserves field order; round-trips through dict
- `test_prompts.py`: substitutes persona into template; template without `{{persona}}` raises; persona renders as readable YAML in prompt
- `test_session_unit.py`: appends turns to history; builds messages list in correct format; export_transcript produces expected markdown shape
- `test_session_integration.py` (gated by `ANTHROPIC_API_KEY`): 3-turn conversation with the CISO persona; assert responses are non-empty and don't contain AI-tell phrases like "as an AI"

**Validation checkpoint**: Run `python -m pitch_practice.cli`. Type a pitch opener. Verify Sarah responds in character. Type 5 more turns. Verify she raises an objection from her list when triggered. Save transcript with `/export` command.

**Estimated effort**: 2–3 hours with Claude Code.

---

### Phase 2 — Slide image support

**Build**:
- Extend `RolePlaySession.send()` to accept `slide_image: bytes, slide_media_type: str`
- Extend `Turn` to store slide image and media type
- Extend message-list construction to emit the multi-part content block when an image is present
- Extend CLI: `/slide path/to/image.png` command to attach an image to the next turn

**Tests**:
- `test_session_unit.py`: image content block constructed with correct shape; base64 encoding correct; media type passed through
- `test_session_integration.py`: send a slide image with a description-prompting transcript ("What do you think of this?"); assert persona reacts to image content

**Validation checkpoint**: Take a screenshot of an actual Okta pitch slide. Run CLI with `/slide`. Verify Sarah's response addresses specific content from the slide, not generic governance talk.

**Estimated effort**: 45 minutes.

---

### Phase 3 — Local STT integration

**Build**:
- `pitch_practice/stt.py` — `WhisperTranscriber` class wrapping `faster-whisper`. Loads model once at init (default `large-v3`). `transcribe(audio_bytes, media_type) -> str` method.
- CLI extension: optional `--audio path.wav` flag to transcribe and feed in as a turn

**Tests**:
- `test_stt.py` (GPU-marked): transcribe `fixtures/sample_audio.wav`, assert non-empty result and reasonable content match

**Validation checkpoint**: Record yourself reading a pitch opener. Pipe through CLI. Verify transcription quality is adequate (>95% word accuracy on clean speech).

**Estimated effort**: 45 minutes including CUDA setup if not already done.

---

### Phase 4 — Local TTS integration

**Build**:
- `pitch_practice/tts.py` — `TTSVoice` class wrapping the chosen engine. `synthesize(text) -> tuple[bytes, str]` returning (audio bytes, media type, e.g., `"audio/wav"`).
- CLI extension: `/speak <text>` to test TTS standalone

**Tests**:
- `test_tts.py` (GPU-marked): synthesize a known string, assert audio bytes non-empty and parseable as WAV

**Validation checkpoint**: Listen to Sarah Reyes saying "Tell me about why I should care about governance for my SOC 2 audit." Verify the voice is intelligible and the persona feel (professional, mid-30s female if matching Sarah, or whatever's chosen) is acceptable. Don't over-tune voice for v1.

**Estimated effort**: 1 hour (TTS setup is the variable).

---

### Phase 5 — Streamlit UI

**Build**:
- `pitch_practice/app.py` — Streamlit app with:
  - Sidebar: persona name + description (read-only for v1)
  - Main: chat-style conversation display
  - Bottom input row: `st.audio_input` for recording, `st.file_uploader` for current slide, submit handled by audio submission
  - On submit: STT → session.send() → TTS → display + autoplay
  - `Export transcript` button
- Use `st.session_state` to hold the `RolePlaySession` instance across reruns
- Hash audio bytes to avoid re-processing the same recording on Streamlit reruns

**Tests**:
- No unit tests on the UI layer (Streamlit testing harness is heavy for a v0 prototype; manual validation only). The session, STT, and TTS are all tested independently.

**Validation checkpoint**: Run `streamlit run pitch_practice/app.py`. Conduct a full 15–20 turn mock pitch. End with transcript export. Confirm:
- Audio capture works end-to-end
- Slide upload triggers visible persona reaction
- Voice playback feels natural enough to maintain immersion
- Transcript captures everything correctly

**Estimated effort**: 1.5–2 hours.

---

### Phase 6 — Iteration based on real session feedback

This is open-ended and the most important phase. The persona YAML and wrapper prompt are v1 best-guesses; only real sessions reveal what's wrong.

Expected adjustments:
- Persona is too easy or too hard → tune `communication_style.trust_posture`, `pushback_pattern`, or `difficulty`
- Objections raised too aggressively or not at all → tune `triggers` for clarity, or strengthen the wrapper prompt's "natural flow" guidance
- Persona breaks character under pressure → strengthen the wrapper prompt's boundary section
- Persona is too talkative or too terse → tune `communication_style.verbosity`

**Capture this in `personas/midmarket-saas-ciso.yaml` and `prompts/role_play_wrapper.md` directly.** No code changes needed for tuning.

## 7. Testing strategy

| Layer | Approach | When tests run |
|---|---|---|
| Persona loading, prompt assembly, message construction, Turn dataclass | TDD strict; pure unit tests, no I/O | Every commit |
| `RolePlaySession.send` end-to-end with real API | Real Claude API, gated by env var | Pre-PR; cost is pennies |
| STT and TTS adapters | Real models, GPU-marked, fixture inputs | On dev machine |
| Streamlit UI | Manual validation per phase checkpoint | Per phase |

**No mocking of the Anthropic API.** If we mock it, we test our mock, not the system. Real API calls are cheap and the only way to know the persona actually responds well.

**No mocking of the local STT/TTS models.** Same reason — real models with fixture audio.

## 8. Future-proofing — what's designed in vs deferred

| v2 capability | What v1 already supports | What v1 doesn't have |
|---|---|---|
| Multi-persona switching | Persona is parameterized; YAML files in a directory | UI dropdown to switch mid-session |
| Real-time conversational | Session API is a clean boundary; STT/TTS are adapters | Streaming, VAD, interruption |
| Avatar (Zoom-style talking head) | TTS output is already produced as bytes ready to drive lip-sync | Avatar service integration |
| Coaching layer | Transcript export is structured | Separate evaluator LLM, rubric, scoring |
| Per-persona voice cloning | TTSVoice adapter interface is engine-agnostic | XTTS-v2 swap-in |
| Local LLM swap | `model` param on RolePlaySession; Claude client can be swapped for an OpenAI-compatible local server (e.g., vLLM, LM Studio) | Adapter wrapping local server's chat API |

**Local LLM swap, concretely**: when ready to benchmark a local model, wrap the local server (vLLM, Ollama, LM Studio) in a class that matches the same minimal interface the session uses today (just `messages.create`-equivalent). The session code does not change.

## 9. Setup notes for Claude Code

When implementing this, prefer:
- `uv` for dependency management (`uv init`, `uv add`)
- `pytest` markers for the slow/GPU/integration tests so the default `pytest` run is fast
- Type hints throughout; `pydantic` for the persona schema
- Naming per Joe's preferences: descriptive nouns, no `Wrapper`/`Manager`/`Helper` suffixes, no temporal context ("new", "legacy", "v2")
- Every Python file starts with two `# ABOUTME:` comment lines per project style
- Small, focused commits per phase

The persona YAML and wrapper prompt are already drafted (see project files `midmarket-saas-ciso.yaml` and `role_play_wrapper.md`). Move them to `personas/` and `prompts/` respectively in the new project.

## 10. Handoff checklist for the Claude Code session

When opening Claude Code, provide it with:

1. This document
2. `personas/midmarket-saas-ciso.yaml`
3. `prompts/role_play_wrapper.md`
4. Joe's `CLAUDE.md` / user preferences (rule #1, TDD, naming, etc.)
5. Instruction: "Start with Phase 1. TDD strictly. Stop at each phase's validation checkpoint and let me run it before proceeding."

Recommended first message to Claude Code: *"Read the architecture doc. Confirm you understand the scope of Phase 1. Write the failing tests for `persona.py` first, then we'll proceed."*
