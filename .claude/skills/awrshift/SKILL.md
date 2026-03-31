---
name: awrshift
description: "Adaptive decision framework — one dynamic flow with user checkpoints at every phase. Guides from problem to solution through structured research, metrics, factcheck, and sandbox testing. Use when you face a non-trivial decision, need to research before building, plan a feature or experiment, evaluate trade-offs, or the user says 'awrshift', 'let's think this through', 'research first', 'experiment', 'investigate', 'what's the best approach', 'compare options'. Also trigger on: 'исследуй', 'разберись', 'проанализируй', 'эксперимент', or when starting any new project phase, migration, launch, or architecture decision. Do NOT use for simple tasks with clear instructions — just do those directly."
---

# AWRSHIFT — Adaptive Decision Framework v2

Think before you build. Research before you code. Decide with evidence.

## Core Principle: User-in-the-Loop

Every phase transition requires explicit user confirmation via AskUserQuestion. The system NEVER proceeds silently. The user always knows: what just happened, what happens next, and chooses how to proceed.

## The Flow

One adaptive flow. Depth adjusts per phase based on user's scope choice.

```
IDENTIFY → RESEARCH → EVALUATE-DESIGN → HYPOTHESIZE → PLAN → FACTCHECK → TEST (sandbox) → DECIDE → [IMPLEMENT]
```

**Mandatory (NEVER skip):** IDENTIFY, RESEARCH, EVALUATE-DESIGN, PLAN, FACTCHECK, DECIDE
**Adaptive (user chooses):** HYPOTHESIZE (skip if single clear path), TEST (skip if plan-only)
**Gated (explicit GO required):** IMPLEMENT in main project

## The Checkpoint Pattern

At EVERY phase boundary, execute this exact sequence:

1. **Summary** — what was done, key findings, artifacts created
2. **Preview** — what the NEXT phase does and why it matters
3. **Choice** — AskUserQuestion with options A/B/C/D + free-text

**CRITICAL:** Always use AskUserQuestion tool for choices. Never present options as plain text. The user must get structured, clickable choices.

Example checkpoint:
```
AskUserQuestion:
  question: "IDENTIFY complete. Problem defined, 3 unknowns found. Next: RESEARCH — I'll investigate unknowns using parallel agents. How to proceed?"
  options:
    - "Proceed with research on all 3 unknowns"
    - "I have context on some unknowns — let me share first"
    - "Add more unknowns to investigate"
    - "Skip research — I already know the answer"
```

---

## Phase Execution

### IDENTIFY (mandatory)

Define the problem before solving it. Use AskUserQuestion for structured questionnaire.

**Step 1 — Questionnaire (one question at a time):**

Q1: "What problem are we solving? Describe the current situation."
→ Free text

Q2: "What does 'done' look like? Desired end state."
→ Free text

Q3: "What do we already know? Prior research, decisions, constraints?"
→ Options: A) "I have context to share" B) "No prior context" C) "Check project history"

Q4: "How deep should we go?"
→ Options:
  A) "Just need a quick answer — minimal research"
  B) "Need a solid plan with evidence"
  C) "This is complex — multiple approaches to compare"
  D) "Not sure yet — let's start and see"

**Step 2 — Produce IDENTIFY summary:**
```
Problem: [what's wrong or needed]
Current: [state now]
Target: [desired state]
Gap: [delta]
Unknowns: [numbered list]
Initial depth: [user's choice — system adapts dynamically from here]
```

**Step 3 — Checkpoint:** Show summary, confirm with user, ask to proceed to RESEARCH.

---

### RESEARCH (mandatory)

Gather evidence before forming opinions.

**Step 1 — Generate research questions** from unknowns. Group by topic.

**Step 2 — Checkpoint:** Show questions, ask user to validate via AskUserQuestion before dispatching agents.

**Step 3 — Execute research:**
Start with 1-2 focused queries. If findings reveal complexity or contradictions, propose expanding:
- More queries on specific unknowns
- Parallel agents (Agent tool) for independent topics
- Save outputs to `experiments/{NNN}/research/` when depth warrants it

**Step 4 — Compile findings:** Synthesize into one document. Highlight confirmed facts, contradictions, remaining unknowns, surprises.

**Step 5 — Checkpoint:** Summarize findings, ask user to proceed to EVALUATE-DESIGN or do more research.

---

### EVALUATE-DESIGN (mandatory)

Define HOW to measure success BEFORE planning. Without metrics, evaluation is subjective.

**Step 1 — Propose 3-7 measurable metrics:**

Each metric needs: name, how to measure, target value, why it matters.

**Step 2 — Gemini rubric check:**
Send metrics to Gemini for independent review — catches self-preference bias and edge case vulnerabilities.
```bash
python3 .claude/skills/gemini/gemini.py second-opinion \
  "Review these success metrics for [problem]. Check for: self-preference bias, missing edge cases, unrealistic targets, metrics that can be gamed. Metrics: [table]" \
  --save experiments/{NNN}/gemini-metrics-review.md
```
**Gemini output rule:** Extract 2-3 sentence summary for the checkpoint. Full Gemini response stays in the saved file — user can request it via "Show full review" option.

**Step 3 — Checkpoint:**
```
AskUserQuestion:
  question: "Here are success metrics: [table]. Gemini flagged: [2-3 key points only]. Full review saved to experiments/{NNN}/gemini-metrics-review.md."
  options:
    - "Metrics look good, proceed"
    - "Show full Gemini review"
    - "Modify these metrics"
    - "Skip formal metrics — I'll evaluate qualitatively"
```

If user wants market research on metrics → run focused research round, return with updated proposal.

---

### HYPOTHESIZE (adaptive)

When multiple viable approaches exist, name them explicitly.

**Skip condition:** If RESEARCH revealed a single clear winner, ask:
```
AskUserQuestion:
  question: "Research points to one clear approach: [X]. Explore alternatives?"
  options:
    - "One approach is enough, go to PLAN"
    - "I have an alternative to suggest"
    - "Brainstorm alternatives with Gemini (brainstorm skill)"
    - "Compare 2-3 approaches side by side"
```

**If comparing (2+ hypotheses):**
Name each hypothesis (H1, H2, H3) with one-sentence description, expected outcome, risk, effort. Present comparison table.

**Gemini falsification (standard/deep with 2+ hypotheses):**
Send hypotheses to Gemini for adversarial challenge — different model family catches blind spots.
```bash
python3 .claude/skills/gemini/gemini.py second-opinion \
  "Given these hypotheses for [problem]: [H1, H2, H3]. For EACH: give 3 concrete scenarios where it fails or where an alternative outperforms. Also: what are we NOT considering?" \
  --save experiments/{NNN}/gemini-hypotheses-review.md
```
**Gemini output rule:** Extract 2-3 key failure scenarios per hypothesis for the checkpoint. Full response stays in saved file.

Feed Gemini's key critique points into hypothesis comparison table. Ask user which to plan for.

---

### PLAN (mandatory)

Concrete implementation plan with sequenced tasks.

**Plan must include:**
- Sequenced tasks with dependencies
- File ownership (what gets created/modified)
- Acceptance criteria linked to EVALUATE-DESIGN metrics
- Risk mitigations
- **Sandbox boundary** — ALL work in `experiments/` folder

**Checkpoint:** Show plan, ask user to proceed to FACTCHECK.

**If user wants to skip FACTCHECK:** Do NOT silently skip. Show a soft warning:
```
AskUserQuestion:
  question: "FACTCHECK catches wrong assumptions and missed risks before you invest time in testing. Skipping means we trust the plan as-is. In past experiments, factcheck caught issues ~60% of the time. Still want to skip?"
  options:
    - "Run factcheck anyway (recommended)"
    - "Skip — I'm confident in the plan"
    - "Do a quick self-check only, skip Gemini"
```

---

### FACTCHECK (mandatory)

Verify plan against original context and research BEFORE execution.

**Step 1 — Self-check (always):**
- Does plan address the problem from IDENTIFY?
- Does plan use evidence from RESEARCH?
- Does plan address all metrics from EVALUATE-DESIGN?
- Does plan respect project constraints?
- What could break? What assumptions are untested?

**Step 2 — Gemini cross-check (standard/deep — mandatory):**
Different model family catches blind spots Claude misses. Write plan summary + context to temp file, send to Gemini:
```bash
python3 .claude/skills/gemini/gemini.py second-opinion @factcheck-prompt.txt \
  --save experiments/{NNN}/gemini-factcheck.md
```
Prompt pattern: "Here's our plan for [problem]. Context: [IDENTIFY summary + RESEARCH findings]. Plan: [tasks]. Check for: missed risks, wrong assumptions, implementation complexity we're underestimating, better alternatives we haven't considered."

Critically evaluate Gemini's response — accept unique insights, reject when it lacks project context.

**Gemini output rule:** Extract 2-3 key points for the checkpoint. Full response stays in `gemini-factcheck.md` — user can request via option.

**Step 3 — Checkpoint:**
```
AskUserQuestion:
  question: "Factcheck complete. Self-check: [pass/fail summary]. Gemini flagged: [2-3 key points]. Full review: experiments/{NNN}/gemini-factcheck.md"
  options:
    - "Issues are minor, proceed to TEST"
    - "Show full Gemini review"
    - "Fix issues and re-check"
    - "Critical issue — revise PLAN"
```

---

### TEST — Sandbox Execution (adaptive)

Execute plan ONLY in experiment sandbox. See Sandbox Safety Rules below.

**After testing — Checkpoint:**
```
AskUserQuestion:
  question: "Test complete in sandbox. Results: [metrics vs targets]. Artifacts in experiments/{folder}/."
  options:
    - "Results look good, proceed to DECIDE"
    - "Iterate — fix issues and re-test"
    - "Abandon — results don't meet criteria"
    - "Show me the artifacts before deciding"
```

---

### DECIDE (mandatory)

Explicit GO / NO-GO / PIVOT with evidence.

**Decision includes:** metrics comparison (target vs actual), evidence summary, recommendation.

**Checkpoint:**
```
AskUserQuestion:
  question: "Decision time. [metrics table]. Recommendation: [GO/NO-GO/PIVOT] because [reason]."
  options:
    - "GO — implement in main project"
    - "NO-GO — archive experiment, document learnings"
    - "PIVOT — modify approach and re-test"
    - "Need more data before deciding"
```

---

### IMPLEMENT — Main Project (gated)

Move validated work from sandbox to main project. ONLY after DECIDE(GO).

**Pre-implementation gate (mandatory):**
```
AskUserQuestion:
  question: "Here's exactly what will change in main project:
    CREATE: [file list]
    MODIFY: [file list with what changes]
    Proceed?"
  options:
    - "Proceed with all changes"
    - "Implement only specific items"
    - "Show diff preview first"
    - "Wait — let me review sandbox artifacts again"
```

NEVER start without this gate passing.

---

## Sandbox Safety Rules

These protect the main project from unvalidated changes. NEVER violate.

### During experiment (IDENTIFY through DECIDE):
1. Create files ONLY in `experiments/{NNN}-{slug}/`
2. Read main project files — allowed (for context)
3. Modify main project files — **FORBIDDEN**
4. Create rules, skills, or configs in main project — **FORBIDDEN**
5. Install packages or change dependencies — **FORBIDDEN**
6. Need to test against main project code — create copies in experiment folder

### After DECIDE(GO) + IMPLEMENT gate:
7. Show exact file list before any changes
8. One-by-one confirmation for modifications
9. Document rollback plan
10. If anything goes wrong — stop, tell user, offer git revert

### If you accidentally modify main project during experiment:
1. Stop immediately
2. Tell user what was modified
3. Offer to revert
4. Document in experiment PLAN.md

---

## Experiment Documentation

### File structure

Simple (single file):
```
experiments/{NNN}-{short-name}.md
```

Complex (folder):
```
experiments/{NNN}-{short-name}/
├── PLAN.md              ← Master: status, phases, metrics, decisions
├── research/
│   └── 01-{topic}.md
├── factcheck.md
└── [artifacts]
```

### PLAN.md Template

```markdown
# Experiment {NNN}: {Title}

**Status:** {IDENTIFY | RESEARCH | EVALUATE-DESIGN | HYPOTHESIZE | PLAN | FACTCHECK | TEST | DECIDE | DONE (GO/NO-GO/PIVOT)}
**Started:** {date}

## Problem
{From IDENTIFY}

## Success Metrics
| Metric | Target | Actual | Status |
|--------|--------|--------|--------|

## Phases
| Phase | Status | Key Output |
|-------|--------|------------|

## Key Decisions
{Inline as they happen}

## Research Findings
{Summary or pointer to research/}

## Factcheck Results
{Summary or pointer to factcheck.md}
```

### Numbering
Continue from highest existing number in `experiments/`. Check before creating.

---

## Integration with Other Skills

### gemini (built-in)
Gemini second-opinion is embedded in 3 phases: EVALUATE-DESIGN (rubric check), HYPOTHESIZE (falsification), FACTCHECK (cross-check). See each phase for exact prompts.
```bash
python3 .claude/skills/gemini/gemini.py second-opinion "prompt" --save result.md
```
If gemini skill is not installed, fall back to self-check only and note it in experiment documentation.

### brainstorm (on demand)
**When:** HYPOTHESIZE phase, user wants multi-round Claude x Gemini dialogue.
**How:** Offer as AskUserQuestion choice: "Brainstorm alternatives with Gemini"
Brainstorm produces converged recommendation → feed back as hypothesis.

---

## Adaptive Behavior

One mode. The system dynamically adjusts depth at each phase based on complexity discovered along the way. User controls depth via AskUserQuestion choices — not by selecting a "mode" upfront.

**How depth adapts dynamically:**
- RESEARCH: start with 1-2 queries. If findings reveal complexity → ask user to expand to parallel agents
- HYPOTHESIZE: skip if research shows single clear path. Offer if multiple viable approaches emerge
- EVALUATE-DESIGN: always propose metrics. Gemini rubric check when metrics are non-trivial
- FACTCHECK: always self-check. Gemini cross-check when plan involves significant changes
- TEST: offer sandbox test. Skip if experiment is plan-only or research-only

If complexity escalates mid-experiment, propose upgrading depth via AskUserQuestion. Never silently escalate or silently skip.

**Gemini integration points (built into flow):**
| Phase | Gemini Role | When |
|-------|------------|------|
| EVALUATE-DESIGN | Rubric check — catches self-preference bias | When metrics are non-trivial |
| HYPOTHESIZE | Falsification — stress-tests each hypothesis | When 2+ hypotheses compared |
| FACTCHECK | Cross-check — different model catches blind spots | When plan involves significant changes |

Gemini output = input for decision-making, NEVER the decision itself. Always critically evaluate: accept unique insights, reject when Gemini lacks project context.

**Minimum that NEVER gets skipped:**
1. IDENTIFY — define the problem
2. EVALUATE-DESIGN — define success criteria
3. PLAN — have a plan before executing
4. FACTCHECK — verify before testing
5. DECIDE — end with explicit decision
6. Documentation — create experiment file
7. Sandbox — work in experiments/ during experiment
8. AskUserQuestion — at every phase transition
