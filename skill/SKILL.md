---
name: brenner-bot
description: "Run a Brenner Protocol multi-agent research session to rigorously evaluate hypotheses, design discriminative tests, and apply adversarial critique using Sydney Brenner's scientific method. Use this skill whenever the user wants to evaluate competing ideas, stress-test a hypothesis, get a scientific verdict on a decision, or apply structured hypothesis-killing to any question. Also triggers for: 'which of these ideas is strongest', 'kill the weakest hypotheses', 'stress-test this idea', 'design experiments to separate these theories', 'apply the Brenner method', 'evaluate these options rigorously', 'what does the science say about', 'brenner bot', 'run a brenner session', or any request for rigorous multi-agent evaluation of competing explanations or business options."
---

# Brenner Bot

Runs a Brenner Protocol session: three agents with incompatible mandates debate hypotheses across multiple rounds until only what survives adversarial pressure remains. You handle framing, human review, stress testing, and the verdict. The CLI handles agent orchestration.

## Why this produces good results

A single LLM asked to evaluate N hypotheses will hedge every one. Three agents in adversarial roles — forced to respond to each other's arguments across multiple rounds — produce explicit kills with named reasons. That's the signal. The linter enforces structural completeness so you can't ship a half-baked artifact.

## The three roles

| Agent | Model | Role | Primary job |
|-------|-------|------|-------------|
| RedForest | Codex | `hypothesis_generator` | Generate hypotheses, hunt paradoxes, import cross-domain patterns, always include a third alternative |
| BlueLake | Claude | `test_designer` | Design discriminative tests that kill hypotheses, score on evidence-per-week, add potency controls |
| GreenMountain | Gemini | `adversarial_critic` | Attack the framing, run scale checks, quarantine anomalies, kill theories that go ugly |

## Session directory convention

```
<project-root>/
  brenner/
    RS-YYYYMMDD-<slug>/
      context.md              ← research question + background (required)
      excerpt.md              ← Brenner transcript quotes (optional)
      artifact.md             ← compiled output, updated after each round
      verdict.md              ← plain-language answer, written at session end
      session_state.json      ← resumable state
      operator_notes_round_N.md  ← HITL feedback, injected into round N+1
      round_N/                ← per-round prompts + agent outputs
      stress_test/            ← post-convergence adversarial pass
```

`RS` = Research Session. Use today's date.

---

## Step 0 — Framing interview (MANDATORY before context.md)

**Do not skip. Do not write context.md without completing this step.**

Assess how much context the user has provided:

**If the user gave substantial background** (product docs, prior research, detailed constraints): confirm the five items below and ask about any gaps. One message, wait for reply.

**If the user provided minimal context** (just a question with no background): ask all five at once:

- **The actual decision** — what output do they need from this session?
- **Hard constraints** — time, budget, team size, anything that rules options out upfront
- **What's already ruled out** — don't waste rounds re-litigating settled ground
- **Success criteria** — what does a good answer change for them?
- **Key uncertainties** — what are they most unsure about?

**Save the interview.** After the user responds, write their answers to `$SESSION/framing_interview.md` before writing `context.md`. Format:

```markdown
# Framing Interview — RS-YYYYMMDD-<slug>

**Question asked:** [the user's original prompt]

**Actual decision:** [user's answer]
**Hard constraints:** [user's answer]
**What's already ruled out:** [user's answer]
**Success criteria:** [user's answer]
**Key uncertainties:** [user's answer]
```

Then write the proposed `--question` string and a one-paragraph context summary. Show both to the user and **wait for explicit approval** before starting the session.

---

## Step 0b — Build a corpus excerpt (recommended)

Ground the session in Brenner's actual method by searching his transcript corpus for quotes relevant to the research question. This injects his reasoning style — not just a summary of it — into the agents' context.

```bash
# Search the corpus for relevant sections
brenner corpus search "your topic keywords" --limit 5

# Build an excerpt from the sections that look most relevant
brenner excerpt build --sections 58,78,161 > $SESSION/excerpt.md
```

The `excerpt.md` file is automatically picked up by `round build` and injected into all three agents' prompts. Without it, agents apply the Brenner operators from description alone. With it, they apply them from example — there's a meaningful difference in how they reason.

**When to skip**: if the research question is purely a business/market/product decision and the biological or experimental science framing would distort rather than help. In that case, omit `excerpt.md` entirely — agents will still follow the operator algebra from the kernel.

---

## Step 0c — Check the kill registry

Before writing `context.md`, check whether `brenner/killed-hypotheses.md` exists at the project root. This file accumulates kills across all sessions in the project.

If it exists:
1. Read it and extract kills relevant to this session's topic
2. Include a `## Prior Session Kills` section in `context.md` summarizing relevant entries
3. When asking framing questions, ask: *"The following argument patterns have already been killed in prior sessions — should any be resurrected for this question?"*

This prevents agents from re-deriving and debating hypotheses that have already been stress-tested to death.

---

## HITL vs unattended mode

**Default is HITL. Do not run unattended unless the user explicitly says so.**

After each round close: summarize what each agent did in plain English (no Brenner jargon, no hypothesis IDs), surface suspicious claims as direct yes/no questions, then ask: *"Anything to push back on or add before round N+1?"* Wait for a response.

Write any substantive feedback to `operator_notes_round_N.md` before building the next round.

**Unattended mode** requires an explicit trigger: "run all rounds", "fully automated", "don't stop between rounds". If unattended, still print a round summary after each close.

---

## Running a session

### Setup

```bash
SESSION=brenner/RS-$(date +%Y%m%d)-<slug>
mkdir -p $SESSION
# write context.md after framing interview
```

### Per-round loop

**Build prompts:**
```bash
brenner round build \
  --session-dir $(pwd)/$SESSION \
  --round N \
  --question "Your research question with explicit evaluation criteria"
```

Writes `round_N/{bluelake,redforest,greenmountain}_prompt.md`.

**Run all three agents in parallel** — issue all three simultaneously, wait for all to complete before closing.

_BlueLake (Claude sub-agent via Agent tool) — NEVER write this inline:_

Use the Agent tool every round, no exceptions. Do not write BlueLake's delta output yourself. The sub-agent gets context isolation — it sees only `bluelake_prompt.md`, not your accumulated session context. That isolation is the point.

```
Agent tool:
  subagent_type: general-purpose
  prompt: Read the file at <absolute path to round_N/bluelake_prompt.md> in full,
          then respond to it exactly as instructed. Output ONLY delta fenced JSON blocks.
  description: "BlueLake round N"
```

The sub-agent returns text. You write it to `round_N/bluelake_out.md`.

_RedForest (Codex via Bash):_

**CRITICAL — file-read approach only.** Codex silently produces zero output when the prompt is passed as a shell argument via `$(cat ...)`. Always pass a short instruction telling Codex to read the file:

```bash
CLAUDECODE="" codex exec --full-auto \
  "Read the file at $(pwd)/$SESSION/round_N/redforest_prompt.md in full, then respond to it exactly as instructed. Output ONLY delta fenced JSON blocks." \
  > $(pwd)/$SESSION/round_N/redforest_out.md 2>&1 &
```

`CLAUDECODE=""` prevents Codex from detecting a nested Claude session. Run in background with `&`. Use the full path to `codex` if it's not in your Bash PATH.

_GreenMountain (Gemini via Bash):_

```bash
gemini --yolo --output-format text \
  -p "$(cat $(pwd)/$SESSION/round_N/greenmountain_prompt.md)" \
  > $(pwd)/$SESSION/round_N/greenmountain_out.md 2>&1 &
```

Inline `$(cat ...)` is fine for Gemini — it handles large prompts correctly. Run in background with `&`. Use the full path to `gemini` if it's not in your Bash PATH.

Wait for all three to complete before closing the round.

**Close the round:**
```bash
brenner round close \
  --session-dir $(pwd)/$SESSION \
  --round N
```

Returns: `{ adds, kills, lintValid, activeHypotheses, killedHypotheses, converged, convergedReason }`

**HITL after each round** — see "Presenting results" below. Write operator notes, then repeat.

**Check convergence:** if `converged: true` → proceed to stress test. Otherwise repeat from build with round N+1.

### Resuming an interrupted session

`session_state.json` is written after each round. Re-run with the same args and it picks up from round N+1.

---

## Convergence criteria

Checked after round 3+:

1. **Kill-rate exceeds add-rate** — more hypotheses killed than added this round
2. **third_alternative** hypothesis present
3. **real_third_alternative** critique present in adversarial section
4. **Linter: VALID** — all 7 required sections structurally complete

**The ≥3 active hypothesis floor is removed.** If a question genuinely converges to 1 or 2 survivors after 3 rounds, that's the answer. Convergence to a single surviving hypothesis is valid data — it means the question has a clear answer. Do not pad the slate with placeholders to satisfy a count rule.

Minimum 3 rounds. Round 1 scaffolds, Round 2 debates, Round 3+ converges.

---

## Survivor stress test (after convergence)

Before the verdict, give survivors one final adversarial pass.

**Write prompts manually** to `$SESSION/stress_test/{bluelake,redforest,greenmountain}_prompt.md`. Each prompt contains:
- The surviving hypotheses (plain English, no IDs)
- The current artifact state (killed hypotheses, open critiques, active assumptions)
- This instruction: *"These hypotheses survived the full session. Your job is to kill them. Attack specifically — no framing critique, no new hypotheses, no adds. For each survivor: what concrete evidence, scenario, or counter-example would falsify it? If you can't construct a kill, explain precisely why it's robust."*

**Run all three agents** using the same invocation pattern as regular rounds. Write outputs to `$SESSION/stress_test/{bluelake,redforest,greenmountain}_out.md`.

**Do NOT close this round through the CLI.** It's a prose attack pass, not a delta round. Read outputs directly.

**HITL after the stress test** — summarize each agent's attack in plain English. For any kill attempt:
- Present the argument to the user: *"[Agent] argued that [hypothesis] fails because [reason]. Does this land, or does the product/idea have an answer to this?"*
- Wait for the user's response before proceeding to the verdict.

**If any survivor is killed**: confirm with the user and include it in the verdict.

**If all survivors are killed**: present this to the user — *"The stress test killed everything. The question may be underdetermined, or the hypothesis space was too narrow. How would you like to proceed?"*

---

## Verdict

**Step 1 — Get the verdict prompt:**
```bash
brenner session verdict-prompt \
  --session-dir $(pwd)/$SESSION \
  --question "..." > $SESSION/verdict_prompt.md
```

**Step 2 — Append stress test results** to `verdict_prompt.md` before running SilentRiver. Include each agent's attack and the operator's response to it. The verdict should be informed by what survived the stress test, not just the main session.

**Step 3 — Spawn an isolated SilentRiver sub-agent** via the Agent tool. The sub-agent sees only the verdict prompt — not your accumulated session context. That isolation is the point.

```
Agent tool:
  subagent_type: general-purpose
  prompt: Read the file at <absolute path to verdict_prompt.md> in full.
          You are SilentRiver, the verdict synthesizer for a Brenner Protocol session.
          Write for the founder — no hypothesis IDs, no Brenner jargon, no hedging.
          Output the verdict text only, in the exact format specified in the file.
```

The sub-agent returns the verdict text. Write it to `$SESSION/verdict.md`.

**Verdict format:**
```
## Verdict

**The answer**: [one sentence — direct answer to the question]

**What got eliminated and why**:
- [plain-language description]: eliminated because [plain English reason]

**What survived**:
- [plain-language description]: [why it held up under adversarial pressure]

**What to do first**: [one specific, concrete next action]

**Biggest unresolved risk**: [what the debate flagged but couldn't kill]
```

Rules: no hypothesis IDs, no Brenner jargon, no hedging. Write for the person making the decision, not for the protocol.

---

## Kill registry

After presenting the verdict, append this session's kills to `brenner/killed-hypotheses.md` at the project root. Create it if it doesn't exist:

```markdown
# Cross-Session Killed Hypothesis Registry

Accumulates kills across all Brenner Protocol sessions in this project.
Read at the start of every new session to avoid re-deriving settled argument patterns.

---
```

Then append one entry per killed hypothesis:

```markdown
### [TOPIC] — [HYPOTHESIS IN PLAIN ENGLISH] (killed DATE)
- **Session**: RS-YYYYMMDD-slug
- **Kill reason**: Plain-English reason it was eliminated
- **Kill agent**: RedForest / GreenMountain / BlueLake
- **Surviving alternatives**: What survived instead
```

Use plain English throughout — no H1/H2 IDs. This file is read by humans and future agents.

---

## Presenting results — after every round

Read the agent outputs. Summarize in **bullet points**, translating all Brenner jargon into plain English. Never say "H1", "T2", "C3" to the user.

**Jargon translation:**

| Instead of... | Say... |
|--------|--------------|
| H1, H2, H3... | "the idea that [claim]" |
| KILL H2 | "eliminated [the idea that...] because..." |
| ADD to hypothesis_slate | "proposed a new idea: that..." |
| adversarial_critique | "attacked the assumption that..." |
| discriminative_test | "designed a test to check whether..." |
| third_alternative | "proposed a completely different approach: that..." |
| predictions_table | "mapped out which test results would support or kill which ideas" |

**Format:**
```
## Round N

**BlueLake** (test designer):
- [what test was designed, in plain English]
- [what was killed/edited and why]

**RedForest** (Codex, hypothesis builder):
- [new idea proposed, in plain English]
- [what was killed and why]

**GreenMountain** (Gemini, critic):
- [what assumption was attacked]
- [what was eliminated and why]
```

**Then scan for suspicious claims** — numbers that look invented, assumptions that contradict context.md, framing the user would instantly recognize as wrong. Surface these as direct yes/no questions, not open-ended checklists.

Then ask: *"Anything to push back on or add before round N+1?"*

---

## Framing the question correctly

The most important decision. Agents evaluate through whatever lens you give them.

**Bad**: "Which idea is strongest?" → agents hedge everything.

**Good**: Embed evaluation criteria in `--question`:
```
"Which of these pricing models best fits a two-person team targeting
high-velocity web app teams (correct-fix rate ≥65%, consolidation ≥8:1,
and a human approval gate at staging link only), and what tests separate
viable from unworkable models?"
```

`context.md` should: define evaluation criteria in detail, list hypotheses or options, state what you want from the session, include key questions to drive analysis.

---

## Artifact structure (7 required sections)

| # | Section | Purpose |
|---|---------|---------|
| 1 | Research Thread | Falsifiable problem statement — singleton, EDIT only |
| 2 | Hypothesis Slate | Candidate explanations — must include `third_alternative` |
| 3 | Predictions Table | Per-hypothesis observable predictions |
| 4 | Discriminative Tests | Ranked tests with potency checks, scored LR×Speed/Ambiguity |
| 5 | Assumption Ledger | Load-bearing assumptions — ≥1 `scale_check` required |
| 6 | Anomaly Register | Quarantined exceptions |
| 7 | Adversarial Critique | Attacks on the framing — ≥1 `real_third_alternative` |

---

## Delta format

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
  "rationale": "[inference]"
}
```

**Operations**: `ADD` (target_id: null), `EDIT` (target_id required), `KILL` (target_id required — use `"reason"` field, not `"kill_reason"`)

**`rationale` is metadata only** — it is never compiled into the artifact. All reasoning must go in payload content fields (`mechanism`, `evidence`, `reason`, `attack`, etc.).

`research_thread` is a singleton — EDIT only, never ADD. Compiler assigns IDs (H1, T1, A1, C1) — agents must not invent their own.

---

## Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| No deltas parsed from agent | Agent produced prose instead of delta blocks | Re-invoke with: "Output ONLY ```delta fenced JSON blocks. One object per fence. No prose." |
| `EH-001`: fewer than 3 active hypotheses | GreenMountain over-killed | Auto-patcher adds synthesis H; check `artifact.md` — these are placeholders, not agent reasoning |
| Codex produces zero output | Prompt passed inline via `$(cat ...)` — Codex silently produces nothing when the prompt is a shell argument | Use file-read approach: pass a short instruction telling Codex to read the prompt file |
| Codex model error ("model not supported") | `-c model=gpt-5.3` flag used on ChatGPT-backed Codex account | Remove the flag entirely |
| `INVALID_JSON` in round output | Codex generates unterminated strings | Skipped automatically; session continues with valid deltas |
| Round 2+ agents write shallow one-liners | Reasoning placed in `rationale` field (stripped from artifact) | Remind agents: `rationale` is not compiled; all reasoning goes in `mechanism`/`evidence`/`reason` |
| Session not converging after 5 rounds | Question too broad | Read `artifact.md`, refine `--question`, add operator notes narrowing scope |
| Agent binary not found | CLI not in PATH | Use `--claude-bin`, `--codex-bin`, `--gemini-bin` flags or `BRENNER_*_BIN` env vars |

---

## The four always-on guards

Inject these into any manual prompt:
- **Third alternative**: always include a real "both could be wrong" option
- **Potency**: distinguish "won't" from "can't" before killing a hypothesis
- **Scale**: calculate orders of magnitude before accepting a mechanism
- **Anomaly hygiene**: quarantine exceptions; don't let Occam's broom hide debt

---

## Reference files

- `../apps/web/src/lib/artifact-merge.ts` — Linter, artifact schema, merge algorithm
- `../apps/web/src/lib/delta-parser.ts` — Delta parsing and validation
- `../apps/web/src/lib/session-kickoff.ts` — Kickoff message composition, agent roles
- `../brenner.ts` — Full CLI including `session robot`, `round build`, `round close`, `session verdict-prompt`
