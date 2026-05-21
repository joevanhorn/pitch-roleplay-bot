# pitch-roleplay-bot

Turn-based pitch role-play tool for Okta Solutions Engineers. Practice a sales conversation against an AI-driven buyer persona — push-to-talk, optionally share slides, get voice + text responses that stay in character for a 30-minute mock pitch.

## Status

v1 prototype, pre-implementation. Persona schema and wrapper prompt drafted; architecture finalized. Code begins at Phase 1.

## Start here

- **Design and phased plan**: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- **Conventions for Claude Code (or any contributor)**: [CLAUDE.md](CLAUDE.md)
- **Reference persona**: [personas/midmarket-saas-ciso.yaml](personas/midmarket-saas-ciso.yaml) — Sarah Reyes, CISO at a mid-market SaaS company
- **Role-play system prompt**: [prompts/role_play_wrapper.md](prompts/role_play_wrapper.md)

## Quick setup (once Phase 1 lands)

```bash
# Install uv if needed
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"   # Windows
# or: curl -LsSf https://astral.sh/uv/install.sh | sh        # macOS / Linux

uv sync
cp .env.example .env   # then fill in ANTHROPIC_API_KEY
uv run python -m pitch_practice.cli
```

## Scope

**In v1**: single persona, turn-based push-to-talk, local STT, local TTS, cloud LLM, Streamlit UI, transcript export.

**Out of v1**: real-time conversational mode, avatars, coaching/scoring layer, multi-user deployment, web hosting. The architecture is set up so v2 can add these without rewriting the core.
