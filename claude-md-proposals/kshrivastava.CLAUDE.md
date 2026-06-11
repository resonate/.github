# CLAUDE.md — resonate/kshrivastava
# This is a NEW file to be created at the root of the kshrivastava repo

# kshrivastava — Multi-Agent Orchestration V3

Personal repository for Kapil Shrivastava (POCs and experiments). The primary artifact is the **V3 multi-agent orchestration framework** in `multi-agent-orchestration-v3/`.

---

## V3 Multi-Agent Orchestration Framework

### What It Is

A Temporal-based orchestration system where AI agents collaborate to plan, code, review, and retrospect on software engineering stories. Each story runs as a Temporal workflow; agents implement domain-specific **roles** and consult domain-expert **skills** mid-run.

```
story ──▶ Temporal Workflow ──▶ Planner ──▶ Coder ──▶ Reviewer ──▶ Retro
                                    │           │
                                    └──▶ Specialist (consult-by-default)
                                          ↑
                                  append-agent / cleanroom /
                                  audience-delivery / ...
```

### Repository Structure

```
multi-agent-orchestration-v3/
├── framework/
│   ├── role/
│   │   ├── runner.py              # Runs a role; injects available_specialists catalog
│   │   ├── loader.py              # Loads role YAML + prompt
│   │   └── schemas.py             # CoderOutput, PlannerOutput, SpecialistOutput, etc.
│   ├── recipe/
│   │   ├── interpreter.py         # Runs recipe phases; Contract-4 consult hook
│   │   └── specialist_consult.py  # run_specialist_consult()
│   └── runtime/
│       └── cassettes.py           # Request/response cassette recording for replay
├── roles/
│   ├── planner.yaml + prompts/planner.md
│   ├── coder.yaml  + prompts/coder.md
│   ├── reviewer.yaml + prompts/reviewer.md   # claude-sonnet-4 (upgraded from Haiku)
│   ├── retro.yaml   + prompts/retro.md
│   └── specialist.yaml + prompts/specialist.md  # Generic; self-selects skill by description
├── skills/
│   ├── WIRING.md                  # Specialist wiring docs
│   ├── append-agent/
│   │   ├── SKILL.md               # Grounding + playbooks for Append domain
│   │   └── references/            # Source map: Confluence/Jira/GitHub/AWS/Splunk
│   ├── cleanroom/                 # Placeholder (routable, content pending)
│   └── audience-delivery/         # Placeholder (routable, content pending)
├── recipes/
│   └── default.yaml               # Phase sequence: Plan → Code → Review → Retro
├── tests/unit/                    # 826+ unit tests
└── docs/
    ├── ENGINEER_ONBOARDING.md
    └── presentations/
        └── special-agent-explainer.html
```

### Key Concepts

**Roles** — YAML-defined agents with a `role: <name>` field and matching `prompts/<name>.md`. Models:
- Planner: `claude-opus-4` (quality-sensitive planning)
- Coder: `claude-sonnet-4`
- Reviewer: `claude-sonnet-4` (upgraded from Haiku — review quality is critical)
- Retro: `claude-haiku-4`
- Specialist: `claude-sonnet-4`

**Skills** — Domain knowledge bundles in `skills/<domain>/`. Each needs a `SKILL.md` with a sharp `description` (the routing key) and grounding workflow. Auto-discovered by `discover_skills()` — no code wiring needed to add a domain.

**Specialist / Consult System (Contract 4 — LIVE):**
1. Any role emits `consultations_needed: [{specialist, query}]` in its output
2. `framework/recipe/interpreter.py` (`_extract_consultations`) calls `run_specialist_consult()`
3. The generic `specialist` role self-selects the skill by description matching
4. `SpecialistOutput {answer, citations, confidence}` is threaded into caller's `prior_attempt_summaries`; caller re-runs with answer in context
5. Per-phase round cap mirrors the escalation cap

**Consult-by-default:**
- `available_specialists` catalog (`{name, description}`) injected into EVERY role's `task_input` by `runner.py`
- Planner/coder prompts: scan catalog first; consult before guessing or escalating a domain fact
- Catalog excluded from cassette fingerprint — recordings replay regardless of deployed specialists

**Ticket-based story lookup:**
- `get_state` / `get_history` accept a bare Jira ticket (e.g. `CDP-117874`) resolving to workflow
- Writes still need explicit `workflow_id`

### Key Commands

```bash
cd multi-agent-orchestration-v3

# Install dependencies
uv sync

# Run unit tests
pytest tests/unit/                  # ~826 tests
pytest tests/unit/test_<name>.py    # specific file

# Start the dev worker (requires AWS SSO + Temporal cluster)
aws sso login
python framework/worker.py

# Story CLI
python cli.py start  --story "CDP-12345" --description "Fix the frobnicator"
python cli.py get-state   --story CDP-12345
python cli.py get-history --story CDP-12345
```

### Rules and Gotchas

- **This is a POC repo.** Production V3 will live elsewhere once the framework is proven.
- **Dev worker restart required** after merging branches. Reviewer→Sonnet model change takes effect only after worker restart.
- **Adding a skill**: Drop `skills/<domain>/SKILL.md` with a sharp `description`. No code changes — `discover_skills()` auto-discovers it. The `description` is the routing key.
- **`workspace_isolation: per_attempt_worktree`**: Specialist uses per-attempt worktrees. Read-only (Q&A) invocations simply don't write.
- **Consult round cap**: After `MAX_CONSULTATION_ROUNDS_PER_PHASE` consults per phase the agent proceeds with what it has. Audit trail in `ctx.keys["_consultation_log"]`.
- **Cassette fingerprint**: `available_specialists` excluded — replay works regardless of which domains are deployed at playback time.
