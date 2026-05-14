# CHANGELOG — nicoechaniz/hermes-agent fork

Team-facing summary of changes to our fork. For upstream changes between syncs,
see `~/wiki/projects/hermes-agent/notes/upstream-changes-review.md`.

---

## 2026-05-11

### Upstream sync (138 commits behind → caught up)

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
