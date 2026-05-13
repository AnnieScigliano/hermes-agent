<<<<<<< HEAD
# Changelog — nicoechaniz/hermes-agent fork

> **Provider note:** Profile configs reference DeepSeek, Kimi, and MiniMax providers because that's our stack. Team members using different providers (OpenRouter, Anthropic, Nous, etc.) should adapt `model.provider`, `model.default`, and `model.base_url` in each profile's `config.yaml`. API keys go in each profile's `.env` (or symlink to shared `.env`). The `max_turns` and `reasoning_effort` values are provider-agnostic and should work across backends.

## 2026-05-09 — Multi-Agent Coding Roster + Kanban Hardening

### New Profiles
- **riqui** (deepseek-v4-flash, max_turns=30, reasoning=minimal): Surgical coding Kanban worker. Fixed protocol violation (was max_turns=15 + reasoning=none → iteration exhaustion before kanban_complete).
- **miki** (kimi-k2.6, kimi-coding OAuth via ~/.kimi/, max_turns=30, reasoning=high): Coding agent. Tested working.
- **maxi** (MiniMax-M2.7, minimax provider, Anthropic endpoint, max_turns=30, reasoning=high): Coding agent. Config created but blocked by CLI api_mode detection bug (404 — hardcoded chat_completions vs anthropic_messages).
- **claudio** (planned): Proxy profile → Claude Code CLI
- **gepeto** (planned): Proxy profile → Codex CLI

### Kanban System
- **Protocol violation root cause:** max_turns too low + reasoning=none on weak models → iteration exhaustion → model writes kanban_complete as text (not function call) → clean exit without transition → effective_limit=1 → auto-blocked
- **Fix:** max_turns ≥ 25 + reasoning ≥ minimal for all Kanban coding workers
- **Self-spawn guard:** Dispatcher DOES spawn tasks assigned to gateway's own profile (compaii). Tasks must stay in `todo`/`triage` until manually claimed.
- **Smoke test pattern:** t_4631001e (17s, riqui) validated the fix

### RTK Plugin
- **FIXED** by Riqui (t_ad89b059): Replaced corrupted `rtk_hermes/__init__.py` (circular self-import) with 332-line source from GitHub
- Binary symlinked for gateway PATH
- Plugin loads cleanly on gateway restart (no WARNING)

### Memory Infrastructure
- HMK chapters 9-11 seeded: dispatcher guard, profile roster, maxi api_mode debug
- Project MEMORY.md updated with full profile roster and dispatcher critical rule

### Known Issues
- **maxi:** `hermes -p maxi chat` returns 404. CLI hardcodes api_mode=chat_completions. Provider transport=anthropic_messages is ignored. curl confirms endpoint works.
- **Upstream:** ~90 commits behind (v2026.5.7+), needs sync

## 2026-05-08 — Upstream Sync v2026.5.7

- Full rebase onto upstream/main (993 commits, 7 conflicts resolved)
- All 10 custom features preserved
- Gateway split: hermes-gateway.service (CompAII) + hermes-gateway@steve.service
# Changelog — nicoechaniz/hermes-agent fork

> **Provider note:** Profile configs reference DeepSeek, Kimi, and MiniMax providers because that's our stack. Team members using different providers (OpenRouter, Anthropic, Nous, etc.) should adapt `model.provider`, `model.default`, and `model.base_url` in each profile's `config.yaml`. API keys go in each profile's `.env` (or symlink to shared `.env`). The `max_turns` and `reasoning_effort` values are provider-agnostic and should work across backends.

Team-facing summary of changes to our fork. For upstream changes between syncs,
see `~/wiki/projects/hermes-agent/notes/upstream-changes-review.md`.

## 2026-05-09 — Multi-Agent Coding Roster + Kanban Hardening

### New Profiles
- **riqui** (deepseek-v4-flash, max_turns=30, reasoning=minimal): Surgical coding Kanban worker. Fixed protocol violation (was max_turns=15 + reasoning=none → iteration exhaustion before kanban_complete).
- **miki** (kimi-k2.6, kimi-coding OAuth via ~/.kimi/, max_turns=30, reasoning=high): Coding agent. Tested working.
- **maxi** (minimax/MiniMax-M2.7, max_turns=25): Fast auxiliary / TTS tasks.
- **compaii** (deepseek-v4-pro, max_turns=80): Architecture + research.

### Kanban Fixes
- `kanban_worker` now enforces `toolsets` field in task spec
- `kanban review` create → test job flow validated end-to-end
- Auto-specify triage agent added for malformed task specs

### Active Feature Branches

| Branch | Status | Notes |
|--------|--------|-------|
| `feat/daemoncraft` | Active | Gateway adapter + embodied heartbeat |
| `feat/kanban-review` | Active | Ship-review orchestration |
| `feat/kimi` | Active | Kimi OAuth coding integration |
| `feat/minimax-defaults` | Merged to main | MiniMax prompt caching + defaults |
| `fix/kimi-context-length-resolution` | Merged to main | Kimi context window fix |

### Pending

- `feat/daemoncraft` is on the old base — needs rebase onto new `nousmain` on next sync cycle
- `nousmain` reset to upstream/main (`8e2eb4b51`)
- `main` merged with upstream (2 conflicts resolved: Kimi OAuth headers + ProviderProfile fallback)
- `feat/daemoncraft` merged into `main` (5 commits: embodied heartbeat, @name! interrupt, compressor fix, embodied_plan, kanban-review)
- Deployed to `~/.hermes/hermes-agent`, gateway restarted

### Kimi K2.6 context window bug — fixed

**Problem:** Hermes rejected `kimi-k2.6` with "context window of 32,768 tokens, below minimum 64,000". Real context is 262,144 (256K).

**Root cause:** Two issues in the context-length resolution chain (`agent/model_metadata.py`, `agent/models_dev.py`):
1. `PROVIDER_TO_MODELS_DEV` was missing `"kimi"` and `"moonshot"` entries (only had `kimi-coding`/`kimi-coding-cn`)
2. OpenRouter metadata (community-maintained, incorrect for kimi-k2.6) was consulted BEFORE the project's own curated `DEFAULT_CONTEXT_LENGTHS`

**Fix (3 changes in 2 files):**
1. Added `"kimi"` → `"kimi-for-coding"` and `"moonshot"` → `"kimi-for-coding"` to `PROVIDER_TO_MODELS_DEV`
2. Gated OpenRouter fallback behind `not effective_provider` — known providers skip third-party metadata and go straight to curated defaults
3. Added explicit `DEFAULT_CONTEXT_LENGTHS` entries for `kimi-k2.6`, `kimi-k2.5`, `kimi-k2`, `k2p6`, `k2p5` (all 262,144)

**Result:** `provider: kimi` with `kimi-k2.6` now resolves to 262,144. No config workaround needed.

**PR upstream:** https://github.com/NousResearch/hermes-agent/pull/23950
**Cherry-picked to:** `feat/kimi-oauth-clean`

### Kanban cleanup

- hermes-agent board had ~2,778 synthetic test tasks from kanban development
- DB backed up to `kanban.db.backup-20260511-145448`, then deleted
- Fresh empty board auto-created on next CLI access

### Branches

| Branch | Status | Notes |
|--------|--------|-------|
| `nousmain` | Clean | Tracks upstream/main exactly |
| `main` | Integration | upstream + all our features |
| `feat/daemoncraft` | Active | Consolidated DC work (needs rebase onto new nousmain) |
| `feat/kimi-oauth-clean` | Active | Kimi OAuth + context-length fix |
| `fix/kimi-context-length-resolution` | Merged to main | Kimi context window fix |

### Pending

- `feat/daemoncraft` is on the old base — needs rebase onto new `nousmain` on next sync cycle
>>>>>>> feat/kimi
