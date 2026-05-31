```diff
+                      .
+                     /|\
+                    / | \
+                   /  |  \
+                  /   |   \
+                 / ,--+--, \
+                /,'   |   ',\
!               //  ,--+--,  \\
!              //__/    |    \__\\
-                 \     |     /
-                  \  __|__  /
-                   \/     \/
!                    '.   .'
!                  ____'.'____
!                 /           \
+                /  I C A R U S  \
+               /                 \
+              '~~~~~~~~~~~~~~~~~~~'
```

> **Self-memory for Hermes agents — with LLM-powered extraction, multi-source context injection, and memory safety fixes.**
>
> *Remember your work. Don't corrupt it.*

> ⚠️ **Fork of [esaradev/icarus-plugin](https://github.com/esaradev/icarus-plugin).** See [Enhancements](#enhancements-over-upstream) for what we changed and why.

---

## Compatibility

| Requirement | Version |
|-------------|---------|
| **Hermes Agent** | 0.15.2 (tested; 0.6.0+ should work) |
| **Python** | 3.11+ |
| **OpenRouter** | Required for LLM extraction (any chat model) |

---

## What it does

Icarus is a **Hermes plugin** that gives agents persistent, cross-session memory. It runs as 16 tools + 4 lifecycle hooks inside Hermes — no external services, no dashboard.

**Core features:**

- **Automatic session capture** — every session end produces structured fabric entries (decisions, resolutions, notes) written to `FABRIC_DIR` as markdown
- **Context injection** — relevant memories, Qdrant knowledge, past sessions, and facts are injected into the agent's context at the start of each session
- **Quality scoring** — sessions are scored on substance and completeness; entries carry `training_value` (high/normal/low)
- **Training pipeline** — fabric entries can be exported as fine-tuning pairs and used to train replacement models via Together AI

For the full upstream feature list (cross-agent handoffs, model replacement, eval tools), see the [original README](https://github.com/esaradev/icarus-plugin).

Icarus writes plain markdown files. Obsidian can read them, but Icarus itself is **not** an Obsidian plugin — it's a Hermes plugin.

---

## Quick install

```bash
git clone https://github.com/ClaudioDrews/icarus-plugin.git
mkdir -p ~/.hermes/plugins/icarus
cp -r icarus-plugin/* ~/.hermes/plugins/icarus/
```

Then add to your Hermes profile `.env` (`~/.hermes/.env`):

```bash
# Required
FABRIC_DIR=~/Vault/fabric
OPENROUTER_API_KEY=sk-or-...

# Strongly recommended (see Enhancements below)
ICARUS_EXTRACTION_MAX_TOKENS=4096
ICARUS_EXTRACTION_MODEL=deepseek/deepseek-v4-flash
```

Verify with `/plugins` inside Hermes:

```
✓ icarus v0.3.0 (16 tools, 4 hooks)
```

---

## How we use it

In our setup, Icarus runs inside **Hermes 0.15.2** on Linux (Mint). `FABRIC_DIR` points to `~/Vault/fabric/` — a subfolder of an Obsidian vault — so all entries are browseable in Obsidian with wikilinks and daily notes.

We rely on Icarus primarily for:

1. **Session documentation** — every agent session automatically produces a structured fabric entry with context, decisions, and outcomes
2. **Cross-session awareness** — `fabric_recall` and context injection give the agent memory of past work without manual prompting
3. **Quality metadata** — `training_value` tagging and session scoring let us track which sessions produced valuable work vs. noise

The training/model-replacement pipeline is available but not currently used in our workflow.

---

## Enhancements over upstream

This fork adds ~550 lines to `hooks.py` and fixes a critical bug in `state.py`. Every change was driven by real breakage in production.

### Memory file isolation (critical bug fix)

**What broke:** Icarus wrote its creative state (learnings, questions, cycle counter) to `MEMORY.md` using `.write_text()` — overwriting the entire file on every session end. The Hermes `memory` tool uses a `§`-delimited format for that file. Icarus's markdown format destroyed it, causing **silent data loss** and recurring "Icarus write conflict" errors that blocked the memory tool from working for days before we noticed.

**Fix:** `write_memory_file()` now writes to **`CREATIVE.md`** instead of `MEMORY.md`. Two writers, two files, zero conflicts.

### LLM-powered session extraction

**What was inadequate:** Upstream extracts session summaries by truncating raw message text — `text[:500]` for the result, `text[:80]` for the summary. No semantic summarization, no structure. Entries were often cut mid-word. 50% of fabric entries in our vault were truncated and useless before this fix.

**Fix:** `on_session_end()` now uses an **LLM** (`_llm_extract_entries`) via OpenRouter to read the full session transcript and produce structured JSON entries with proper context, decision, and outcome fields. No truncation — the LLM writes the summary, not a character limit. The legacy path remains as fallback when the LLM fails.

> ⚠️ **This is why `ICARUS_EXTRACTION_MAX_TOKENS=4096` matters.** The default of 1024 tokens is too small for sessions with 3+ entries. Set it to 4096 in your `.env`.

### Multi-source context injection

Upstream injects only fabric entries into the agent's context. This fork adds three more:

| Source | How | What it finds |
|--------|-----|---------------|
| **Qdrant** | Semantic search via `context_enhancer` pipeline | Technical docs, wiki concepts, past decisions |
| **Sessions** | FTS5 over `state.db` | Past conversations on similar topics |
| **Facts** | FTS5 over `memory_store.db` | Structured facts with entity resolution |

Each source has **per-session deduplication** — the same result won't be injected twice.

### Quality of life

| Feature | What it prevents |
|---------|-----------------|
| **Backtick sanitization** | `_sanitize_learning()` strips unpaired backticks that corrupt markdown rendering |
| **System injection filter** | Detects orchestrator preambles (`[IMPORTANT:`, `[SYSTEM:`) so they aren't captured as "user tasks" |
| **Social closer detection** | Skips "ok", "thanks", "👍" — prevents trivial messages from triggering context searches |
| **Configurable limits** | `ICARUS_RESULT_MAX_CHARS` and `ICARUS_TASK_MAX_CHARS` env vars control fallback truncation |

---

## Environment variables

All variables go in your Hermes profile `.env` (e.g. `~/.hermes/.env`).

| Variable | Default | Recommended | Notes |
|----------|---------|-------------|-------|
| `FABRIC_DIR` | `~/fabric/` | `~/Vault/fabric/` | Where Icarus writes markdown entries |
| `OPENROUTER_API_KEY` | — | `sk-or-...` | Required for LLM extraction. Also checks `OPENROUTER_FULL_API_KEY` and `OPENROUTER_DS_API_KEY` |
| `ICARUS_EXTRACTION_MAX_TOKENS` | `1024` | **`4096`** | ⚠️ Increase from default — 1024 is too small for real sessions |
| `ICARUS_EXTRACTION_MODEL` | `deepseek/deepseek-v4-flash` | same | Any OpenRouter chat model |
| `ICARUS_RESULT_MAX_CHARS` | `500` | `500` | Fallback truncation (only when LLM extraction fails) |
| `ICARUS_TASK_MAX_CHARS` | `300` | `300` | Fallback truncation for task capture |
| `ICARUS_OBSIDIAN` | — | `1` | Enable Obsidian wikilinks and daily notes |
| `OBSIDIAN_VAULT_PATH` | — | `~/Vault` | Vault root (only needed if `FABRIC_DIR` is a subfolder) |
| `TOGETHER_API_KEY` | — | — | Only needed for training/eval tools |

---

## Files

```
__init__.py           registration (16 tools, 4 hooks)
plugin.yaml           manifest
schemas.py            tool schemas
tools.py              tool handlers
hooks.py              lifecycle hooks (heavily modified — see Enhancements)
state.py              fabric I/O, session scoring (MEMORY.md fix)
obsidian.py           opt-in Obsidian formatting
fabric-retrieve.py    ranked retrieval with scoring
export-training.py    training pair extraction
scripts/
  eval-replacement.py model comparison eval
  smoke-handoff.sh    end-to-end handoff proof
  test-plugin.sh      66-test fixture suite
```

---

## License

MIT — same as upstream.
