# Claude Code conventions for pitch-roleplay-bot

<!-- ABOUTME: Build conventions and guardrails for Claude Code working in this repo. -->
<!-- ABOUTME: Read this first on every fresh session. Architecture lives in docs/ARCHITECTURE.md. -->

## What this project is

Turn-based pitch role-play tool for Okta Solutions Engineers. The SE speaks (push-to-talk), optionally shares a slide image, and an AI buyer persona responds in voice and text, staying in character for a 20–50 turn session. v1 ships a single persona (mid-market SaaS CISO, "Sarah Reyes") in a Streamlit UI on localhost, with local STT (faster-whisper), local TTS, and cloud LLM (Claude Sonnet 4.7).

Full design and phased plan: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Code conventions — non-negotiable

- **ABOUTME headers.** Every Python file starts with two `# ABOUTME:` lines explaining what the file is. Markdown uses `<!-- ABOUTME: -->`. YAML uses `# ABOUTME:`.
- **Naming.** Descriptive nouns. No `Wrapper`, `Manager`, `Helper`, `Util`, or `Service` suffixes. No temporal context in names (`new_`, `legacy_`, `v2_`, `old_`).
- **Type hints throughout.** `pydantic` for schemas.
- **Small, focused commits per phase.** Do not merge phases. Each phase ends in a runnable, validated state.

## Build discipline

- **TDD strict.** Write the failing test first. Run it. See it fail. Then implement. No exceptions in the `RolePlaySession`, `Persona`, and `prompts` modules — these are the testable core.
- **Stop at each phase's validation checkpoint** (defined in `docs/ARCHITECTURE.md` section 6). Let Joe run the checkpoint before proceeding to the next phase.
- **Do not add scope beyond the current phase.** If something useful comes up that isn't in this phase, flag it and defer.

## Testing rules

- **Do not mock the Anthropic API.** Real API calls in integration tests, gated by `ANTHROPIC_API_KEY` being set. Mocked LLM tests test the mock, not the system.
- **Do not mock local STT/TTS models.** Real models with fixture audio inputs. Mark these tests so the default `pytest` run stays fast.
- **`pytest-mock` only for non-API surfaces** (filesystem, env, etc.).
- **Pytest markers**: use `@pytest.mark.integration` for real-API tests and `@pytest.mark.gpu` for STT/TTS tests so contributors can run `pytest -m "not gpu and not integration"` by default.

## LLM client design — important for the planned liteLLM swap

Joe intends to migrate the LLM client to **liteLLM** eventually, for multi-provider flexibility.

- Keep the Anthropic client behind the minimum interface `RolePlaySession` actually uses.
- Take `client` as a constructor argument to `RolePlaySession`. Don't import `anthropic` deep inside the session code.
- Don't lean on Anthropic-specific event shapes inside `RolePlaySession`. The session should be agnostic to the provider.
- When the swap happens, the change is a thin adapter around `litellm.completion()` that returns what `RolePlaySession` expects — no session changes.

## Dependency management

- Use `uv` for Python deps and venv (`uv init`, `uv add`, `uv run`).
- Pin Python to 3.12 unless a dep forces otherwise.
- `pyproject.toml` is the source of truth. Don't create `requirements.txt` separately.

## Repo layout (current and target)

```
pitch-roleplay-bot/
├── README.md
├── CLAUDE.md                    ← you are here
├── pyproject.toml               ← created at Phase 1 start
├── .env.example
├── .gitignore
├── docs/
│   └── ARCHITECTURE.md
├── personas/
│   └── midmarket-saas-ciso.yaml
├── prompts/
│   └── role_play_wrapper.md
├── pitch_practice/              ← Phase 1+
│   ├── __init__.py
│   ├── persona.py
│   ├── prompts.py
│   ├── session.py
│   ├── stt.py                   ← Phase 3
│   ├── tts.py                   ← Phase 4
│   ├── cli.py
│   └── app.py                   ← Phase 5
├── voices/                      ← Phase 4 (gitignored)
└── tests/
    ├── conftest.py
    ├── fixtures/
    ├── test_persona.py
    ├── test_prompts.py
    ├── test_session_unit.py
    ├── test_session_integration.py
    ├── test_stt.py
    └── test_tts.py
```

## What not to do

- Don't add retry/backoff logic in v1. The Anthropic SDK has its own retries; accept the defaults.
- Don't add history summarization/truncation. A 30-minute session fits comfortably in Claude's context.
- Don't build multi-persona UI selection in v1. The data model supports it; the UI keeps it simple.
- Don't introduce streaming (`messages.stream()`) in v1. Synchronous `messages.create` only.
- Don't add a coaching/scoring layer in v1.
- Don't commit `.env` or any file with a real API key.
