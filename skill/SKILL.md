---
name: brenner-bot
description: "Run a Brenner Protocol multi-agent research session to rigorously evaluate hypotheses, design discriminative tests, and apply adversarial critique using Sydney Brenner's scientific method. Use this skill whenever the user wants to evaluate competing ideas, stress-test a hypothesis, get a scientific verdict on a decision, or apply structured hypothesis-killing to any question. Also triggers for: 'which of these ideas is strongest', 'kill the weakest hypotheses', 'stress-test this idea', 'design experiments to separate these theories', 'apply the Brenner method', 'evaluate these options rigorously', 'what does the science say about', 'brenner bot', 'run a brenner session', or any request for rigorous multi-agent evaluation of competing explanations or business options."
---

# Brenner Bot

Runs a fully automated Brenner Protocol session using `brenner session robot` — no Agent Mail, no services, no coordination overhead. Three agents debate in adversarial roles across multiple rounds until hypotheses converge to a defensible verdict.

## Why this produces good results

Three agents with incompatible mandates: RedForest adds, BlueLake tests, GreenMountain kills. A single LLM asked to evaluate N hypotheses will hedge every one. Three agents in adversarial roles across multiple rounds, forced to respond to each other's arguments, produce explicit kills with named reasons. That's the signal. The linter enforces structural completeness so you can't ship a half-baked artifact.

## The three roles

| Agent | Model | Role | Primary job |
|-------|-------|------|-------------|
| RedForest | Codex | `hypothesis_generator` | Generate hypotheses, hunt paradoxes, import cross-domain patterns, always include a third alternative |
| BlueLake | Claude | `test_designer` | Design discriminative tests that kill hypotheses, score on evidence-per-week, add potency controls |
| GreenMountain | Gemini | `adversarial_critic` | Attack the framing, run scale checks, quarantine anomalies, kill theories that go ugly |

## Session directory convention

Sessions live in a `brenner/` folder at the root of the current project:

```
<project-root>/
  brenner/
    RS-YYYYMMDD-<slug>/
      context.md          ← your research question + background (required)
      excerpt.md          ← Brenner transcript quotes (optional, built via `brenner excerpt build`)
      artifact.md         ← compiled output, updated after each round
      session_state.json  ← resumable state
      round_N/            ← per-round prompts + agent outputs
```

`RS` = Research Session. Use today's date.

## Running a session

### CLI flags

| Flag | Required | Description |
|------|----------|-------------|
| `--session-dir <path>` | **Yes** | Path to session directory. Dir name becomes the thread ID. Reads `context.md` and `excerpt.md`; writes `artifact.md`, `session_state.json`, `round_N/`. |
| `--question <string>` | **Yes** | One-sentence research question. Agents evaluate through this lens — encode criteria explicitly (see Framing). |
| `--context-file <path>` | No | Defaults to `<session-dir>/context.md`. |
| `--excerpt-file <path>` | No | Defaults to `<session-dir>/excerpt.md`. Safe to omit. |
| `--max-rounds <n>` | No | Default 5. Typical sessions converge in 3. |
| `--claude-bin <path>` | No | Path to Claude CLI. Defaults to `BRENNER_CLAUDE_BIN` env var, then `claude` in PATH. |
| `--codex-bin <path>` | No | Path to Codex CLI. Defaults to `BRENNER_CODEX_BIN` env var, then `codex` in PATH. |
| `--gemini-bin <path>` | No | Path to Gemini CLI. Defaults to `BRENNER_GEMINI_BIN` env var, then `gemini` in PATH. |

### Quickstart

```bash
# From your project root
SESSION=brenner/RS-$(date +%Y%m%d)-<slug>
mkdir -p $SESSION

cat > $SESSION/context.md << 'EOF'
<2-4 sentences of essential background>
EOF

brenner session robot \
  --session-dir $(pwd)/$SESSION \
  --question "Your research question with explicit evaluation criteria" \
  --max-rounds 5
```

**Resuming an interrupted session** — `session_state.json` is written after each round. Re-run with the same args and it picks up from round N+1.

**Building an excerpt** (optional but recommended — grounds agents in Brenner's actual method):
```bash
brenner corpus search "your topic keywords" --limit 5
brenner excerpt build --sections 58,78,161 > $SESSION/excerpt.md
```

## Framing the evaluation correctly

This is the most important decision. Agents evaluate through whatever lens you give them.

**Bad framing**: "Which idea is strongest?" → agents evaluate at venture scale, hedge everything.

**Good framing**: Embed the evaluation criteria in `--question`:
```
"Which of these ideas best satisfies micro-SaaS constraints (specificity moat,
recurring pain, days-to-weeks build, zero employees, $5K/mo floor too small for VCs),
and what decisive tests separate viable from venture-scale ideas?"
```

**context.md should**:
1. Define evaluation criteria with detailed descriptions
2. List the hypotheses or options being evaluated
3. State what you want from the session
4. Include key questions to drive analysis

Anti-pattern: putting a generic question in `--question` and hoping agents infer the framework from context. They won't.

## Convergence criteria

The orchestrator checks these after round 3+:

1. **Kill-rate exceeds add-rate** — more hypotheses killed than added this round
2. **≥3 active hypotheses** remain (not over-killed)
3. **third_alternative** hypothesis present
4. **real_third_alternative** critique present in adversarial section
5. **Linter: VALID** — all 7 required sections structurally complete

Minimum 3 rounds. Round 1 scaffolds, Round 2 debates, Round 3+ converges.

## Artifact structure (7 required sections)

| # | Section | Purpose | Constraint |
|---|---------|---------|------------|
| 1 | Research Thread | Falsifiable problem statement | Singleton, EDIT only |
| 2 | Hypothesis Slate | Candidate explanations | Must include `third_alternative` |
| 3 | Predictions Table | Per-hypothesis observable predictions | One row per H |
| 4 | Discriminative Tests | Ranked tests with potency checks | Scored LR×Speed/Ambiguity |
| 5 | Assumption Ledger | Load-bearing assumptions | ≥1 `scale_check` required |
| 6 | Anomaly Register | Quarantined exceptions | Don't sweep under Occam's broom |
| 7 | Adversarial Critique | Attacks on the framing | ≥1 `real_third_alternative` |

## Delta format (what agents emit)

Each agent contribution is a JSON object in a ` ```delta ` fence. One object per fence, never arrays.

```delta
{
  "operation": "ADD",
  "section": "hypothesis_slate",
  "target_id": null,
  "payload": {
    "name": "Short title",
    "claim": "One-sentence falsifiable claim",
    "mechanism": "Causal story in the system's primitives",
    "anchors": ["§103"]
  },
  "rationale": "Why this is worth considering"
}
```

**Operations**: `ADD` (target_id: null), `EDIT` (target_id required), `KILL` (target_id required, use `"reason"` field not `"kill_reason"`)

**Sections**: `hypothesis_slate`, `discriminative_tests`, `assumption_ledger`, `anomaly_register`, `adversarial_critique`, `research_thread`, `predictions_table`

`research_thread` is a singleton — EDIT only, never ADD. Compiler assigns IDs (H1, T1, A1, C1, etc.) — agents must not invent their own.

## Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| No deltas parsed from agent | Agent produced prose instead of delta blocks | Check `round_N/<agent>_out.md`; tighten the prompt |
| `EH-001`: fewer than 3 active hypotheses | GreenMountain over-killed | Auto-patcher adds synthesis H; check `artifact.md` |
| Sparse output after round 1 | Normal — agents underfill on first pass | Orchestrator auto-patches structural gaps; round 2 fills them |
| `INVALID_JSON` in round output | Codex sometimes generates unterminated strings | Those deltas are skipped; session continues with valid ones |
| Session not converging after 5 rounds | Question too broad or agents not killing aggressively | Read `artifact.md`, refine `--question`, restart |
| Agent binary not found | CLI not in PATH | Use `--claude-bin`, `--codex-bin`, `--gemini-bin` flags or `BRENNER_*_BIN` env vars |

## The four always-on guards

Inject these into any manual prompt:
- **Third alternative**: always include a real "both could be wrong" option
- **Potency**: distinguish "won't" from "can't" before killing a hypothesis
- **Scale**: calculate orders of magnitude before accepting a mechanism
- **Anomaly hygiene**: quarantine exceptions; don't let Occam's broom hide debt

## Reference files

- `../apps/web/src/lib/artifact-merge.ts` — Linter, artifact schema, merge algorithm
- `../apps/web/src/lib/delta-parser.ts` — Delta parsing and validation
- `../apps/web/src/lib/session-kickoff.ts` — Kickoff message composition, agent roles
- `../brenner.ts` — Full CLI including the `session robot` command implementation
