# PR Intent Verification — Customer Discovery vs Specs vs POC Coverage Analysis

> **Purpose.** This document maps every problem, doctrine point, workstream concern, and solution area from the customer-discovery analysis (`pr-intent-validation-customer-discovery-analysis.md`) against three downstream artifacts produced in the sister repo `finwise-ces-PR-INTENT-VERIFIER-AGENT`:
> 1. **Spec #1** — `specs/agentic-development/pr-intent-verifier-agent/pr-intent-verifier-agent-specs.md` (the standalone verifier)
> 2. **Spec #2** — `specs/agentic-development/pr-intent-verification-workflow/pr-intent-verification-workflow-design.md` (the orchestrated multi-agent loop built **on top of** Spec #1)
> 3. **POC** — the running implementation in `.github/agents/` + `.github/skills/` (11 agents, 27 skills, 4 custom prompts, 1 CI workflow)
>
> The goal is to answer: *of the eleven customer problems we set out to address, how many are covered by what we have built so far, and what gaps remain?*

---

## 0. Quick-Read Summary

**Headline.** The current specs + POC cover the **core verification mechanic end-to-end** — drift detection, doc-lag closure, and iterative resolution. The remaining gaps are *not* a single missing "half" of the lifecycle; they fall into **five distinct shapes**, each demanding a different kind of next slice:

| Shape | Position relative to the verifier | What's missing | S# from customer discovery |
|---|---|---|---|
| **Upstream** | *Before* there is verifiable intent | Asks clarifying questions when the spec is too thin to verify against | S5 (+ partial S10, P9) |
| **Sibling evals** | *Parallel to* intent on the same PR | Tech-debt, security, test-quality, and style as discrete evals composed alongside intent verification | S2 composition, S9 |
| **Inside the core** | *Within* the existing verifier | Architecture/call-graph grounding the verifier consumes; a regression fixture suite for the skill itself | S7, S11 |
| **Enveloping pattern** | *Wraps* the core | Multi-shot generate-N → score-each → pick-best, with the verifier as the scoring function | S13 |
| **Downstream** | *After* the verdict | Eval calibration against ground truth; trust-staged deployment policy; honest AI-impact telemetry; post-merge PR Learn | S14, S8, S12, S6 |

The five shapes line up cleanly with the four recommendations in §7: **(A)** closes the upstream shape, **(B)** the sibling-eval shape, **(C)** the downstream shape, and **(D)** the two internal shapes (inside-the-core + enveloping).

**One-line scorecard:**

| Coverage tier | Count | Items |
|---|---|---|
| ✅ **Fully covered** (spec + POC implement it) | 3 / 11 problems | P1*, P4, P7 |
| ⚠️ **Partially covered** (some mechanism present, but the customer-described scope is bigger) | 5 / 11 problems | P5, P6, P8, P10, P11 |
| ❌ **Not covered** (no spec or POC implements it) | 3 / 11 problems | P2, P3, P9 |
| ➖ **Out of declared scope** | 0 / 11 | — |

> *P1 is fully covered only for the "match code against attached intent docs" half. The "generate clarifying questions when intent is missing" half (Taylor Williams' explicit ask) is **not** covered — see §5.1.

**Where the major gaps are concentrated:**
- **Trust loop** (SR2): eval calibration (S14), trust-staged deployment (S8), and honest AI-impact telemetry (S12) are entirely absent from both specs and the POC.
- **Spec lifecycle** (SR4): pre-code clarifying questions (S5) and post-merge PR Learn (S6) — neither has any implementation surface.
- **Adjacent patterns** (SR5): the Multi-Shot + Retrospective + ROI Selector (S13) for picking the best of N drafts is not in scope of either spec.
- **Grounding** (SR3): the Code Context Grounding Service (S7, S10) is approximated by the verifier's deterministic discovery (grep/glob/view) but no architecture-graph / call-graph / artifact index service exists.

**Where the work shines:**
- The **iterative coordinator + coder + PM loop** in Spec #2 directly validates the customer pattern that *Shreyansh (ODSP)* explicitly asked for, and *David Coulter (Talos)* aligned to.
- The **bidirectional classification** (ALIGNED / CODE_BEHIND / CODE_AHEAD / CONFLICT) plus the **Surface vs Appendix significance filter** are well beyond what the customer-discovery doc described — they are net-new architecture that makes the reports actually consumable.
- The **read-only / deterministic discovery** in the verifier (grep + glob + `view_range`, no LLM-side mutation) directly matches doctrine **D3** ("deterministic work outside the LLM where possible").

---

## 1. Source Material — At a Glance

| Source | Path | What it is | Size |
|---|---|---|---|
| **Customer Discovery** | `Customer-Discovery/pr-intent-validation-customer-discovery-analysis.md` | Synthesis of 5 interviews (May 18–21) defining 11 problems (P1–P11), 3 doctrine points (D1–D3), 2 workstream concerns (W1–W2), and 5 solution areas (SR1–SR5) with sub-solutions S1–S14 | 1 072 lines |
| **Spec #1 — Verifier Agent** | `.../pr-intent-verifier-agent/pr-intent-verifier-agent-specs.md` | Standalone bidirectional intent verifier — single-pass; 6 dimensions; 4 classifications; significance-filtered; emits one Markdown report per Loop | 577 lines |
| **Spec #2 — Verification Workflow** | `.../pr-intent-verification-workflow/pr-intent-verification-workflow-design.md` | Wraps Spec #1 in an orchestrated loop with `@pr-intent-resolution-coordinator` (hub) plus `@Collaborative dev lead` (code-side worker) and `@PM` (spec-side worker); tiered iteration caps; 7 termination gates; Run/Loop audit trail | 406 lines |
| **POC Implementation** | `.github/` of the same repo | 11 agent files (incl. coordinator @ 94 KB, verifier @ 22 KB, co-dev @ 21 KB, critic @ 19 KB, PM @ 11 KB), 27 skills (incl. `intent-verification` @ 134 KB, `agentic-eval`, `tech-debt-discovery`, `security-threat-modeler`, `postmortem`), 4 custom prompts, 1 CI workflow | n/a |

> **Vocabulary note.** Customer discovery says **"validation"**; the specs and POC say **"verification"**. They refer to the same thing for the purposes of this analysis — both compare *what code does* against *what intent documents say*.

---

## 2. Coverage Matrix — Problems × Sources

Legend:
- ✅ **Full** — directly addressed by the artifact's design and present in the implementation
- 🟢 **Strong** — addressed by design; some implementation surface in place
- ⚠️ **Partial** — design touches the problem but doesn't fully solve it as the customer described
- ❌ **None** — no mechanism in the artifact addresses this problem
- ➖ **n/a** — out of scope for that artifact's stated charter

### 2.1 Problems (P1–P11)

| # | Problem (one-line) | Spec #1 (Verifier) | Spec #2 (Workflow) | POC | Net coverage |
|---|---|---|---|---|---|
| **P1** | "Intent against *what*?" — no canonical spec for many PRs | 🟢 Resolves attached intent docs; supports multi-doc input | 🟢 Coordinator escalates when no report/spec found (No-Report-Path Rule) | 🟢 Verifier + PM agent + custom prompts (vision/scope, research) | ✅ for the **resolution** half; ❌ for the **clarifying-questions** half (S5) |
| **P2** | Eval accuracy only ~60% — no calibration | ❌ Confidence levels exist but no calibration loop | ❌ No calibration stage | ❌ No eval-of-eval mechanism | ❌ Not covered |
| **P3** | Capability-gap failures ship undetected | ❌ Verifier is per-PR, not telemetry over time | ❌ Workflow is per-PR | ❌ No production telemetry | ❌ Not covered |
| **P4** | Tests can be wrong (vacuous, brittle, mock-only) | 🟢 D4 dimension explicitly assesses "tests cover acceptance criteria" structurally | 🟢 Loop iterates on D4 findings | 🟢 SKILL.md §D4 + Critic agent + `test-quality-analysis` skill | ✅ Covered for structural test review; ❌ for runtime test correctness (out of declared static scope) |
| **P5** | Tech-debt blindness ("don't see what we've already shipped that hurts") | ❌ Verifier doesn't evaluate debt | ❌ Workflow doesn't evaluate debt | ⚠️ `tech-debt-discovery` skill exists but is NOT wired into the verifier or coordinator | ⚠️ Mechanism exists in isolation; no integration into the PR flow |
| **P6** | Subjective reviewer preferences enforced via opinion | ❌ Verifier has no preferences layer | ❌ Workflow has no preferences layer | ⚠️ `pm-spec-writing`, `feature-spec`, and review/critique skills enforce templates; no team-style rule engine | ⚠️ Partial — style consistency baked into spec authoring, not enforced as discrete checks on code |
| **P7** | Docs lag code by months (Shreyansh's explicit pain) | 🟢 Verifier emits `CODE_AHEAD` findings + recommends `update_spec` | 🟢 `@PM` worker patches specs when verifier finds CODE_AHEAD | 🟢 PM agent file + `pm-spec-patch` skill exists, plus full `pm-spec` workflow | ✅ **Fully covered** for the doc-sync direction once a verification is triggered |
| **P8** | Validators starved of app/code context | ⚠️ Verifier uses deterministic discovery (glob/grep/`view_range`) and reads the spec — no architecture graph | ⚠️ Same as Spec #1; loop adds nothing on this axis | ⚠️ `semantic-codebase-intelligence`, `explain-codebase` skills exist but are NOT used by the verifier | ⚠️ Partial — discovery is bounded and read-only, but there's no architecture/call-graph grounding (S7) |
| **P9** | Tacit knowledge gap — undocumented invariants live in heads | ❌ Verifier only reads attached docs | ❌ Workflow only reads attached docs | ❌ No invariant-extraction mechanism | ❌ Not covered |
| **P10** | Skills/agents drift — what worked yesterday breaks today | ⚠️ Per-Run reports + Significance Filter freeze a contract; YAML schema 1.2 versioning | ⚠️ Same | ⚠️ The intent-verification skill is itself a 134 KB stable contract; CI workflow is the only meta-eval | ⚠️ Partial — there's a stable contract, but no "skill regression" tests |
| **P11** | No way to pick best draft from N — "tournament" missing | ❌ Verifier compares 1 code state vs 1 intent | ❌ Loop iterates a single draft; not N parallel drafts | ❌ No Multi-Shot pattern (S13) | ❌ Not covered |

### 2.2 Doctrine (D1–D3)

| # | Doctrine | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **D1** | *Discrete per-check evals over mega-prompt* | 🟢 6 dimensions (D1–D6) + 8 triggers (T1–T8) act as discrete checks within a single verifier pass | 🟢 Per-finding routing to coder vs PM vs human is also discrete | 🟢 SKILL.md encodes the 8-trigger matrix and per-dimension routing precedence | ✅ Aligned (within the verification domain — other domains like tech-debt, security, perf are separate skills not yet composed) |
| **D2** | *Iterative loop over single-pass* | ❌ Verifier is explicitly single-pass; it "STOPS and recommends" | ✅ Coordinator owns the loop; soft cap 3 / hard cap 5; 7 termination gates | ✅ Coordinator agent + agentic-eval skill encode the loop | ✅ Aligned |
| **D3** | *Deterministic work outside the LLM where possible* | 🟢 Discovery via glob/grep/`view_range` is deterministic; classifications are rule-driven from triggers | 🟢 Ledger metrics (`resolved_count`, `regression_count`, `convergence_rate`) are mechanical | 🟢 Verifier skill encodes the rules; CI workflow is deterministic | ✅ Aligned (could go further: e.g., compute trigger evaluation deterministically rather than in-prompt) |

### 2.3 Workstream Concerns (W1–W2)

| # | Concern | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **W1** | Multiple teams (PR Lifecycle, AI Code Review, Talos, ODSP) building parallel systems | ⚠️ Could be hosted by any of them; no host-neutrality contract called out | ⚠️ Same — local-agent first; coordinator is host-agnostic in spirit | ⚠️ All code lives in `.github/agents/` + `.github/skills/` — local-only; no PR-host adapter (S4) | ⚠️ Partial — the local-first stance is aligned with §9.6 of customer discovery, but no formal cross-team interop contract yet |
| **W2** | PR-velocity metrics gameable; need honest AI-impact telemetry | ❌ Per-PR report only | ❌ Per-Run/Loop ledger is per-session, not aggregated | ❌ No telemetry pipeline | ❌ Not covered (S12) |

### 2.4 Solution Areas (SR1–SR5)

Each solution area contains 2–4 sub-solutions. Coverage at the sub-solution level:

#### **SR1 — PR Validation Agents & Skills** (the heart of the discovery)

| # | Sub-solution | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **S1** | Intent-Source Resolver (find the canonical spec / arch doc) | 🟢 Inputs section + Discovery Rule | 🟢 No-Report-Path Rule + Triage Confirmation Gate | 🟢 Verifier agent + `pm-context-gathering` skill | ✅ Covered for the discovery half (P1) |
| **S2** | Discrete Multi-Check Eval Framework (intent + security + tech-debt + style as separate evals) | 🟢 Intent only — 6 dimensions + 8 triggers | ⚠️ Same (workflow doesn't add new checks) | ⚠️ Intent-verification is mature; `security-threat-modeler`, `tech-debt-discovery`, `test-quality-analysis`, `log-pattern-analyzer` exist as separate skills but are NOT composed under a unified check framework | ⚠️ Partial — intent dimension is done; other checks are unwired |
| **S3** | Loop Orchestrator (the coordinator pattern) | ❌ Out of scope (single-pass) | ✅ Entire Spec #2 is this | ✅ `pr-intent-resolution-coordinator.agent.md` (94 KB) | ✅ Fully covered |
| **S4** | Host-Neutral Adapters (works on GitHub, ADO, GitLab) | ❌ No mention | ❌ No mention | ❌ Currently `.github/`-shaped (Copilot agents host) | ❌ Not covered |
| **S9** | Style/Preferences Rules (per-team, opt-in) | ❌ No preferences layer | ❌ No preferences layer | ⚠️ Spec critique skill enforces template style, not code style | ❌ Not covered for code-style; ⚠️ for spec-style |
| **S11** | Anti-Pattern Test Suite (golden bad-cases to keep the eval honest) | ❌ Verifier has worked examples but no test suite | ❌ Not mentioned | ⚠️ CI workflow exists; meta-eval of the verifier itself is not a regression suite | ❌ Not covered |

#### **SR2 — Eval Trust Stages & Feedback Loop**

| # | Sub-solution | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **S8** | Trust-Staged Deployment Policy (Tier 0 advisory → Tier 1 informational → Tier 2 enforced) | ❌ | ❌ | ❌ | ❌ Not covered |
| **S12** | Honest AI-Impact Telemetry (cycle-time, rework rate, % AI-generated, escape rate) | ❌ | ❌ | ❌ | ❌ Not covered |
| **S14** | Eval Calibration Loop (false-positive / false-negative tracking → tune dimensions / triggers) | ❌ Confidence is a per-finding label, not a calibration signal | ❌ Convergence-rate metric exists but is per-Run, not aggregated | ❌ | ❌ Not covered |

#### **SR3 — Code Context Grounding Service**

| # | Sub-solution | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **S7** | Architecture / Artifact Grounding (call graph, type graph, ownership index) | ⚠️ Deterministic glob/grep/`view_range` discovery (Step 3 of SKILL.md) — bounded, but not a graph | ⚠️ Same | ⚠️ `semantic-codebase-intelligence`, `explain-codebase` skills exist as siblings; not wired in | ⚠️ Partial |
| **S10** | Legacy Docs Bootstrap (auto-generate stub specs from large undocumented codebases) | ❌ | ❌ | ⚠️ `explain-codebase`, `feature-spec` skills could be composed but no orchestration named | ❌ Not directly covered |

#### **SR4 — Spec & Knowledge Lifecycle**

| # | Sub-solution | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **S5** | Pre-Code Clarifying Questions (Taylor Williams' explicit ask — agent asks clarifying questions before code is written) | ❌ Verifier is *after* code exists | ❌ Workflow assumes a spec already exists | ⚠️ `pm-context-gathering`, `architect-examiner.agent.md`, custom-prompts/vision-scope can ask clarifying questions — but these are not triggered when the verifier finds an unresolved intent | ⚠️ Pieces exist; integration into the verifier flow does not |
| **S6** | PR Learn (post-merge — extract lessons, update knowledge base) | ❌ | ❌ | ⚠️ `postmortem` skill (5 KB) exists, but only for production regressions, not routine merges | ❌ Not covered as designed |

#### **SR5 — Adjacent Generation Patterns**

| # | Sub-solution | Spec #1 | Spec #2 | POC | Net |
|---|---|---|---|---|---|
| **S13** | Multi-Shot + Retrospective + ROI Selector (generate N variants, pick best by composite score) | ❌ | ❌ | ❌ | ❌ Not covered |

---

## 3. Per-Problem Deep Dive

The matrix above is the scannable view; the sections below explain *why* each cell received its rating, what the customer voice said, and what would be required to close any gap.

### 3.1 P1 — "Intent against *what*?" (no canonical spec exists for many PRs)

> **Customer voice (Shreyansh, ODSP; Taylor, Outlook).** "Most of our repos don't have an up-to-date spec. We can't validate intent if intent has never been written down."

**Coverage:**
- **Spec #1** explicitly accepts "any combination of feature spec, architecture, plan, data model, contracts, task list (paths or attached content)" as inputs and escalates when the input is ambiguous. ✅
- **Spec #2's** No-Report-Path Rule (§Coordinator agent) forces an explicit user discovery when no spec or prior report exists — it refuses to fabricate. ✅
- **POC** has `pm-context-gathering`, `pm-spec-writing`, `pm-vision-scope`, and four `custom-prompts/` prompts for vision/scope and business/technical research. ✅

**Gap:** all three deliberately stay in the "verify what's attached" lane. The customer-discovery doc's S5 ("Pre-Code Clarifying Questions") and Taylor's explicit ask — *"have the agent generate clarifying questions instead of pure validation"* — is a different mode: **proactive, before the diff exists, when the spec is too thin to verify against**. None of the three sources triggers a clarifying-question flow when verification fails because the spec is too vague.

> Suggested closure: a "Verifier-detects-vague-intent → PM-elicits-clarifying-questions → user-answers → spec-fragment-created → re-verify" sub-loop in Spec #2. The mechanism pieces (`pm-context-gathering` + `architect-examiner`) exist already.

---

### 3.2 P2 — Eval accuracy ~60% (no calibration)

> **Customer voice (David Coulter, Talos).** "Current evals are low trust, high variability. We need calibration before we can stage adoption."

**Coverage:** ❌ across all three.
- Spec #1 has per-finding `confidence: High|Medium|Low`, but this is a self-report, not a *measured* calibration against ground truth.
- Spec #2's `convergence_rate` metric is a per-Run progress signal — it does not tell you whether the verifier's *findings* were correct.
- POC's `agentic-eval/SKILL.md` describes the reflection pattern abstractly but has no scoreboard, no held-out fixture set, and no false-positive/false-negative tracking.

> Suggested closure: introduce an **eval-of-eval** stage (SR2.S14) — a small golden set of "PR diff + spec + expected verdict" fixtures the verifier is run against, scoring precision/recall per dimension and per trigger. The output is a per-dimension trust score that feeds S8 trust staging.

---

### 3.3 P3 — Capability-gap failures ship undetected

> **Customer voice (David Coulter, Tiago).** "Things slip through that the model just couldn't reason about — concurrency, distributed systems edge cases. We never even know we missed them."

**Coverage:** ❌ across all three. This is fundamentally a **production-telemetry** problem (does the verdict match what users see in production?), which the per-PR verification scope cannot answer.

> Suggested closure: S12 telemetry pipeline that joins **verifier verdict** ⨯ **escape rate** ⨯ **incident root cause classifier**. This is squarely SR2 territory and depends on real production data, not just the local verifier.

---

### 3.4 P4 — Tests can be wrong (vacuous, brittle, mock-only)

> **Customer voice (Taylor Williams).** "Tests are our de-facto enforcement, so when the tests are wrong we ship the wrong behavior. The agent should at least flag suspicious test shapes."

**Coverage:** 🟢 **Fully covered structurally**, ❌ for runtime correctness.
- Spec #1's **D4 dimension** rubric explicitly checks "each acceptance criterion has at least one corresponding test; error and edge paths tested, not only happy path" (SKILL.md lines 470–484).
- Spec #2's loop will iterate D4 CODE_BEHIND findings (e.g., "no test exists for AC-3") through the coder.
- The POC has both this dimension and a separate `test-quality-analysis/SKILL.md` skill — though that skill is not wired into the verifier today.

**Caveat.** The verifier is explicitly *structural* — it can flag "no test exists" but not "this test is vacuous / mock-only / doesn't actually exercise the path". The latter is left to the Critic agent and could be tightened by composing `test-quality-analysis` into the D4 dimension.

---

### 3.5 P5 — Tech-debt blindness

> **Customer voice (general).** "We keep shipping AI-generated code that creates more debt than it pays down, and we can't see it accumulate."

**Coverage:** ⚠️ **A skill exists, but it's not in the loop.**
- The POC ships `.github/skills/tech-debt-discovery/SKILL.md` (6 KB) — a complete methodology for inventorying TODO/FIXME markers, dep drift, hotspots, churn. ✅
- But neither Spec #1 nor Spec #2 invokes it. The verifier is exclusively about spec-to-code alignment; the workflow's worker matrix is `@Collaborative dev lead` (code-write) and `@PM` (spec-write). There is no "debt discovery" worker.

> Suggested closure: add a `@tech-debt-auditor` worker (or have the coordinator invoke the `tech-debt-discovery` skill conditionally — e.g., when the verifier sees `>N` CODE_AHEAD findings, which is a heuristic for accreting unmanaged code). This is a small extension to Spec #2.

---

### 3.6 P6 — Subjective reviewer preferences enforced via opinion

> **Customer voice (Taylor).** "Reviewers block PRs for style reasons that aren't documented anywhere — every team has different preferences."

**Coverage:** ❌ for code-style; ⚠️ for spec-style.
- The PM agent and `pm-spec-critique` skill enforce a spec **template** (consistent shape, section ordering, traceability) — that addresses spec preferences.
- Code-style preferences (naming, layering opinions, "use this helper not that one") have no representation. The verifier's Significance Filter would actually demote these to Appendix (see SKILL.md §6.5 "code-snippet-drift anti-pattern") — i.e., **the system is designed to *not* fight on style**, which is the opposite of what S9 proposes.

> Net: the design philosophy here is *intentionally* the opposite of S9. The verifier treats style as Appendix; S9 wants style as a discrete, configurable check. These are reconcilable (per-team rules file → discrete check), but neither spec covers it today.

---

### 3.7 P7 — Docs lag code by months ✅ **(strong full coverage)**

> **Customer voice (Shreyansh, ODSP).** "Our docs are 6-12 months behind the code. We need automation that closes that gap, not just flags it."

**Coverage:** ✅ This is the **standout coverage** of the work to date.
- Spec #1's `CODE_AHEAD` classification (with `recommendation_kind: update_spec`) is the formal output for this case.
- Spec #2 §8.1 explicitly **adopts** `@PM` as the spec-side worker that *executes* the update, instead of just recommending it. This is bidirectional — the loop closes by mutating the spec when the code wins.
- POC has `pm.agent.md` + `pm-spec-patch` skill + `pm-spec-writing` skill in place.

**This is the single area where the customer-discovery problem has been most fully translated through spec → POC.**

---

### 3.8 P8 — Validators starved of app/code context

> **Customer voice (general).** "The agent looks at the PR diff in isolation, doesn't understand the architecture it's modifying."

**Coverage:** ⚠️ Bounded coverage.
- The verifier explicitly uses **on-demand reading** with `view_range` slices, never bulk-loading whole trees (SKILL.md §Step 4, "context rot" guidance). It searches with glob/grep first to scope.
- This is good discipline against context blowout — but it is *not* the same as a **grounding service** that pre-computes a call graph, type graph, or ownership index.
- The POC has skills (`semantic-codebase-intelligence`, `explain-codebase`, `architect-examiner`) that *could* feed grounding into the verifier — but they are not currently composed into the verification flow.

> Suggested closure: add an upstream "context bootstrap" step that produces a small architecture digest (entry points, layer map, public-API list) which the verifier consumes as part of its Discovery Map (Step 3 of SKILL.md). This is S7 minus the heavy graph infra.

---

### 3.9 P9 — Tacit knowledge gap

> **Customer voice (Shreyansh, Taylor).** "Half the invariants in our system aren't written down — they live in senior engineers' heads. The agent will violate them silently."

**Coverage:** ❌ Not covered. The verifier and workflow only read attached documents. They have no mechanism to:
- Extract latent invariants from existing code (e.g., "this service is always called via the queue, never synchronously")
- Surface them as discovered claims that the human can confirm/deny
- Persist confirmed invariants into a knowledge file

> This is adjacent to **S10 (Legacy Docs Bootstrap)** but stricter — it's *invariant extraction*, not spec-stub generation. The POC's `explain-codebase` skill is the closest existing surface, but it doesn't extract invariants per se.

---

### 3.10 P10 — Skills/agents drift

> **Customer voice (multiple).** "What worked yesterday breaks today. The agent's behavior is changing under us."

**Coverage:** ⚠️ Partial — the *contract* is versioned, but there is no *regression test* for the skill itself.
- The intent-verification skill is a stable 134 KB methodology file with explicit `schema_version: "1.2"` in the YAML output. Coordinator refuses non-matching schema versions (Spec #2 §6.3). ✅ This pins the contract.
- The Significance Filter's worked examples and anti-patterns sections lock down expected behavior on canonical inputs. ✅
- **What's missing:** a held-out fixture set with expected verifier outputs, run on every change to the skill, that catches regressions. This is what S11 ("Anti-Pattern Test Suite") prescribes.
- CI workflow exists (`workflows/ci.yml` @ 15 KB) but its scope is the underlying codebase, not the verifier-as-a-skill regression suite.

> Suggested closure: a `.github/skills/intent-verification/fixtures/` folder with `case-N.spec.md` + `case-N.code/` + `case-N.expected-verdict.yaml`, plus a CI job that runs the verifier against each fixture and diffs the YAML output.

---

### 3.11 P11 — No way to pick best draft from N

> **Customer voice (general — implicit across multiple interviews).** "We let the agent generate one draft, then iterate. What if we generated five and picked the best?"

**Coverage:** ❌ Not covered. Both specs assume a single code state at a time. There is no:
- N-shot generation orchestrator
- Composite scoring rubric (intent score × test score × debt score × style score)
- Tournament / pareto selector

This is **S13** in the customer-discovery doc and is the largest *missing pattern*, not just a missing feature.

> Suggested closure: a separate spec/agent — the verifier could be a **scoring function** inside a tournament loop. The pieces (verifier produces a deterministic numeric verdict via the YAML) are already there; what's missing is the outer "generate-N → score-each → pick-best" wrapper.

---

## 4. Side-by-Side — Spec #1 vs Spec #2

The two specs are layered: **Spec #2 calls Spec #1**. Understanding what each adds (and what each does NOT add) helps when prioritizing the next slice of work.

| Capability | Spec #1 (Verifier) | Spec #2 (Workflow) | Delta — what Spec #2 adds |
|---|---|---|---|
| Read intent docs | ✅ | (delegates to #1) | — |
| Discover code via glob/grep | ✅ | (delegates) | — |
| Classify findings (ALIGNED / CODE_BEHIND / CODE_AHEAD / CONFLICT) | ✅ | (consumes from #1) | — |
| 6 dimensions (D1 Stack/Tech, D2 Arch, D3 Data/API, D4 Functional/Tests, D5 Quality Attrs, D6 Security) | ✅ | (consumes) | — |
| Significance Filter (8 triggers T1–T8; Surface vs Appendix) | ✅ | (consumes) | — |
| Confidence calibration (High/Med/Low — self-reported) | ✅ | (consumes) | — |
| One Markdown report per Loop with YAML Machine-Readable Summary | ✅ | (consumes) | — |
| **Iterate** until ALIGNED or termination gate | ❌ explicit single-pass | ✅ | **Soft cap 3 / hard cap 5; 7 gates** |
| Dispatch findings to a code-side worker | ❌ "recommends and STOPS" | ✅ `@Collaborative dev lead` | **The bidirectional doc-sync loop** |
| Dispatch findings to a spec-side worker (close P7) | ❌ recommends `update_spec` only | ✅ `@PM` | **The other half of bidirectional sync** |
| Triage Confirmation Gate (human approves before dispatch) | n/a | ✅ | **Human-in-the-loop checkpoint** |
| CONFLICT Decision Prompt (human reconciles spec vs code) | n/a (flagged only) | ✅ | **Explicit human arbitration** |
| Iteration ledger metrics (`resolved_count`, `regression_count`, `convergence_rate`, `unchanged_findings`) | n/a | ✅ | **Convergence telemetry per Run** |
| Run / Loop naming (`Run<X>/...-Run<X>-Loop<Y>.md` per §6.4) | partial (single counter) | ✅ (two counters, subfoldered) | **Campaign vs iteration audit trail** |
| Re-verification full vs delta scope policy | n/a | ✅ (always full) | **Trust integrity for ALIGNED verdict** |
| `--auto` invocation flag (skip Triage Gate every iteration) | n/a | ✅ | **Power-user mode** |
| `--continue-run=<X>` (resume an existing campaign) | n/a | ✅ | **Resumable workflows** |

> **One-sentence summary of the delta:** Spec #2 is *exactly* the doctrine D2 ("iterative loop over single-pass") wrapper around Spec #1, with the spec-side worker (`@PM`) being the mechanism that closes P7 ("docs lag code") rather than just flagging it.

---

## 5. POC Implementation Mapping

This section answers: *given a customer problem, which agent or skill file in the POC realizes it?*

### 5.1 Agents present in `.github/agents/`

| Agent file | Size | Role | Maps to |
|---|---|---|---|
| `pr-intent-resolution-coordinator.agent.md` | **94 KB** (largest) | Hub-and-spoke orchestrator per Spec #2 | S3 Loop Orchestrator; P7 (loop closure) |
| `pr-intent-verifier.agent.md` | 22 KB | Standalone verifier per Spec #1 | S1, S2 (intent dim), P1 (resolution), P4, P7 (flagging) |
| `co-dev.agent.md` | 21 KB | "Collaborative dev lead" — owns the Researcher / Coder / Critic trio for code-side work | Code-write worker for Spec #2; supports D1 doctrine (discrete passes) |
| `coder.agent.md` | 16 KB | Implementation specialist invoked by co-dev | Code-write workhorse |
| `critic.agent.md` | 19 KB | Lens-focused code review (correctness/security/perf/maintainability/clarity/etc.) | Adjacent to S2 (security, quality lenses); related to P4 (test review) |
| `researcher.agent.md` | 17 KB | Technology research worker invoked by co-dev | Reduces P8 by surfacing API/version facts before coding |
| `pm.agent.md` | 11 KB | Spec/docs worker; loads pm-spec-writing / pm-spec-critique / pm-vision-scope skills | **P7 closure; S5 partial (pm-context-gathering); S1** |
| `architect-examiner.agent.md` | 6 KB | "Demonstrate understanding" mode — clarifying questions about architecture | **S5 partial — closest existing surface to Pre-Code Clarifying Questions** |
| `codebase-explainer.agent.md` | 2 KB | Pointer to the explain-codebase skill | Adjacent to S7 grounding |
| `teaching-mode-coder.agent.md`, `microsoft-teaching-mode-coder.agent.md` | 21 KB each | Pair-coding "explain as you go" modes | Adjacent — not part of verification |

### 5.2 Skills present in `.github/skills/` (27 total — only those relevant to customer-discovery solutions listed)

| Skill folder | Size of `SKILL.md` | What it does | Maps to |
|---|---|---|---|
| `intent-verification` | **134 KB** | The full methodology used by the verifier — 9-step verification process, 6-dimension rubric, 8-trigger Significance Filter, report template, worked examples, anti-patterns | S1, S2 (intent dim), all of Spec #1 |
| `agentic-eval` | 7 KB | Reflection / evaluator-optimizer / rubric / LLM-as-judge patterns | Doctrine D2; pattern source for Spec #2 |
| `tech-debt-discovery` | 6 KB | Debt marker scan + dep analysis + git history hotspots | **S2 (debt dim) — exists but unwired** |
| `security-threat-modeler` | 9 KB | Threat modeling skill | **S2 (security dim) — exists but unwired** |
| `test-quality-analysis` | — | Test shape critique | **P4 deeper; S2 (test-quality dim) — exists but unwired** |
| `log-pattern-analyzer` | — | Log triage skill | Could feed S12 telemetry (not designed for it) |
| `postmortem` | 5 KB | Production-regression postmortem template | **Adjacent to S6 PR Learn — but only for production escapes, not routine merges** |
| `hypothesis-driven-debugging` | — | Bug investigation discipline | — |
| `semantic-codebase-intelligence` | — | Codebase Q&A | **S7 partial — could ground the verifier but isn't wired in** |
| `explain-codebase` | — | Codebase narrative skill | **S10 partial — could bootstrap stub specs but no orchestration** |
| `pm-context-gathering` | — | PM discovery skill | **S5 partial — clarifying-question pieces; not triggered by verifier** |
| `pm-vision-scope`, `pm-spec-writing`, `pm-spec-critique`, `pm-spec-patch`, `pm-approach-selection`, `pm-spec` | — | The spec-authoring family | P7 (PM agent loads these to close the loop) |
| `feature-spec` | — | Feature-spec template | S1 supporting |
| `doc-coauthoring`, `release-notes`, `stories-journal` | — | Adjacent doc skills | — |
| `refactor`, `scaffolding-generator` | — | Code-mutation skills | Adjacent |
| `tech-deep-research`, `microsoft-tech-deep-research` | — | Research helpers | Used by researcher agent |
| `git-commit` | — | Commit-message helper | — |
| `make-skill-template` | — | Meta-skill for authoring new skills | Meta — could support P10 closure (regression fixtures for skills) |

### 5.3 Custom prompts and CI

| File | Purpose | Maps to |
|---|---|---|
| `custom-prompts/new.idea.vision.scope.prompt.md` | Vision & scope prompt | S5 (elicitation, pre-code) |
| `custom-prompts/research.business.prompt.md` | Business research prompt | S5, S1 |
| `custom-prompts/research.technical.prompt.md` | Technical research prompt | S7 adjacent |
| `custom-prompts/architecture-technology.prompt.md` | Arch + tech prompt | S5, S7 adjacent |
| `.github/workflows/ci.yml` (15 KB) | CI for the underlying codebase | Adjacent — not the skill-regression fixture suite suggested for P10/S11 |

### 5.4 What the POC has **but the specs don't call out**

These are real assets that aren't yet referenced by either Spec #1 or Spec #2:

- `tech-debt-discovery` (P5 / S2)
- `security-threat-modeler` (S2)
- `test-quality-analysis` (P4 deeper / S2)
- `semantic-codebase-intelligence`, `explain-codebase` (S7 / S10)
- `architect-examiner` + `pm-context-gathering` (S5)
- `postmortem` (closest existing surface for S6)

> These represent **low-cost integration opportunities** — the skills are written; the spec work is to define how the coordinator composes them.

---

## 6. Consolidated Gaps — What Nothing Currently Covers

Sorted by customer signal strength (interview frequency × value):

| Gap | Tied to | Why it matters | Indicative effort |
|---|---|---|---|
| **Pre-Code Clarifying Questions** — generate clarifying questions when the spec is too thin to verify | P1 (deeper half), S5, Taylor's explicit ask | Without this, the verifier returns "NOT_ASSESSED" and the user is stuck; this is the largest *behavioral* gap | Medium — pieces exist (`pm-context-gathering`, `architect-examiner`) but need to be wired as a verifier sub-loop |
| **Eval Calibration Loop + Trust Staging + Telemetry** | P2, P3, W2, S8/S12/S14 | David Coulter's whole "low trust, high variability" message; the gating concern for organizational adoption | Large — needs a fixture set, scoring infra, telemetry pipeline |
| **Multi-Shot + Retrospective + ROI Selector** | P11, S13 | The "best of N" pattern explicitly called out in customer-discovery as the missing meta-pattern | Medium — verifier already produces machine-readable scoring inputs; needs an outer tournament wrapper |
| **Composed Discrete-Eval Framework** (intent + tech-debt + security + test-quality as one coordinator-managed suite) | P5, S2 | Skills exist as siblings but are not composed; the customer pattern is "discrete evals stacked", not "intent verification alone" | Medium — define a per-PR check matrix and add `@tech-debt-auditor`, `@security-modeler` workers; coordinator dispatches per-dimension |
| **Architecture / Call-Graph Grounding** | P8, S7 | Verifier's bounded discovery is good discipline but not the same as an indexed graph the verifier can ask "who calls X?" | Large if done as a service; small if done as a precomputed digest the verifier consumes |
| **Skill-Regression Fixture Suite** | P10, S11 | Without this, every refactor of the intent-verification SKILL.md is a regression risk | Small-Medium — golden-set fixtures + a CI job |
| **Tacit-Knowledge / Invariant Extraction** | P9 | The hardest unsolved customer pain — extracting invariants from existing code and surfacing them as candidate claims | Large — research-grade |
| **PR Learn (post-merge)** | P6 (partial), S6 | Loop the *outcomes* of merged PRs back into the knowledge base | Medium — `postmortem` skill is the seed; needs a routine-merge cousin |
| **Style/Preferences Rules (per-team, opt-in code style)** | P6, S9 | Currently the verifier *demotes* style differences (Appendix); customers want a configurable enforce-or-not control | Small — a `style-preferences.yaml` config + a discrete check |
| **Host-Neutral Adapters** | W1, S4 | Currently `.github/` Copilot-agents-shaped only; cross-team adoption needs ADO/GitLab paths | Medium — coordinator is already agent-runtime-agnostic in spirit |

---

## 7. Recommendations — Prioritized Next Slices

If you can do only **one** thing next, do **(A)**. If two, do **(A)** then **(B)**. If three, add **(C)**.

### A. Close the clarifying-question loop (P1 deeper + S5)
**Why:** This was the single most actionable feedback from Taylor Williams ("have the agent generate clarifying questions instead of pure validation"), and the pieces already exist in the POC. It is the difference between *"sorry, I can't verify"* and *"here are 3 questions whose answers let me verify"*.

**What:** Add a verifier sub-state — when a claim lands in NOT_ASSESSED with reason `SPEC_TOO_VAGUE`, the coordinator dispatches `@PM` with the `pm-context-gathering` + `architect-examiner` skills to elicit clarifying answers, write them back into the spec, then re-verify.

**Effort:** Medium. ~1–2 weeks of spec + agent edits; no new infra.

---

### B. Wire the discrete-eval suite (P5, S2)
**Why:** Customers consistently asked for *intent + tech-debt + security + test-quality* as discrete, composable checks (D1 doctrine). The skills exist; they're not composed.

**What:** Spec #3 (new) — "PR Multi-Check Coordinator" that wraps Spec #2 and adds `@tech-debt-auditor`, `@security-modeler`, `@test-quality-reviewer` as parallel workers. Each emits a machine-readable verdict; the coordinator aggregates.

**Effort:** Medium. ~2–3 weeks. The verifier's YAML schema becomes the per-check contract template.

---

### C. Start the trust loop (P2, P3, S14 first; S8/S12 next)
**Why:** Without calibration, the organizational adoption argument collapses (David Coulter's explicit signal). S14 unblocks S8 (trust staging) and S12 (honest telemetry).

**What:** Build a `fixtures/` folder with ~20 golden cases (spec + code + expected verdict), run on every PR to the verifier skill, score precision/recall per dimension. Surface the per-dimension trust score in the coordinator's iteration ledger. *Don't* try S8 trust staging until you have measured calibration data.

**Effort:** Small-Medium for S14 (~1–2 weeks); S8 + S12 are larger and depend on production deployment data.

---

### D. Lower-priority but high-leverage
- **Skill-regression fixtures (S11 / P10)** — small effort, large quality dividend; protects all future skill edits.
- **Architecture grounding digest (S7 minimal)** — precompute an entry-points / public-API / layer-map markdown file the verifier consumes during Discovery Map (Step 3); large quality dividend for D2 + D3 dimensions.
- **Multi-Shot wrapper (S13)** — defer until A–C are in; requires the calibrated verifier from (C) to be the scoring function.

---

## 8. Appendix — Mapping Tables

### 8.1 Customer Discovery section → this analysis section

| Customer Discovery section | Where addressed here |
|---|---|
| §1 Executive Summary | §0 + §1 |
| §6A Problems (P1–P11) | §2.1 + §3 |
| §6B Doctrine (D1–D3) | §2.2 |
| §6C Workstream (W1–W2) | §2.3 |
| §7 Solution Areas (SR1–SR5 with S1–S14) | §2.4 + §6 |
| §9 Audit / Read-only quotes | (preserved verbatim where quoted) |
| §9.6 Local-first commitment | §2.3 W1 row |

### 8.2 Spec → POC artifact

| Spec component | POC realization |
|---|---|
| Spec #1 §5 Bidirectional Classification | `intent-verification/SKILL.md` §Divergence Classification |
| Spec #1 §6 6 Dimensions | `intent-verification/SKILL.md` §Verification Dimensions |
| Spec #1 §8 Verification Process | `intent-verification/SKILL.md` §Verification Process — 9 Steps |
| Spec #1 §9 Output Format | `intent-verification/SKILL.md` §Report Template + §Machine-Readable Summary |
| Spec #2 §2 Components | `pr-intent-resolution-coordinator.agent.md` + `pr-intent-verifier.agent.md` + `co-dev.agent.md` + `pm.agent.md` |
| Spec #2 §3 Orchestration | `pr-intent-resolution-coordinator.agent.md` §Report Contract / §Routing Table |
| Spec #2 §6.4 Run/Loop naming | `intent-verification/SKILL.md` §Report File — Naming and Location |
| Spec #2 §7 Convergence Model | `pr-intent-resolution-coordinator.agent.md` §Convergence Rules |

### 8.3 Solution sub-solution → existing POC asset (where one exists)

| Sub-solution | Existing POC asset | Wired into verifier? |
|---|---|---|
| S1 Intent-Source Resolver | `pr-intent-verifier.agent.md` + `pm-context-gathering` | ✅ |
| S2 Discrete Multi-Check (intent) | `intent-verification` skill | ✅ |
| S2 Discrete Multi-Check (tech-debt) | `tech-debt-discovery` skill | ❌ |
| S2 Discrete Multi-Check (security) | `security-threat-modeler` skill | ❌ |
| S2 Discrete Multi-Check (test-quality) | `test-quality-analysis` skill | ❌ |
| S3 Loop Orchestrator | `pr-intent-resolution-coordinator.agent.md` | ✅ |
| S4 Host-Neutral Adapters | — | n/a |
| S5 Pre-Code Clarifying Questions | `pm-context-gathering`, `architect-examiner.agent.md`, `custom-prompts/new.idea.vision.scope.prompt.md` | ❌ (not triggered by verifier) |
| S6 PR Learn | `postmortem` skill (production-only seed) | ❌ |
| S7 Architecture/Artifact Grounding | `semantic-codebase-intelligence`, `explain-codebase` | ❌ |
| S8 Trust-Staged Deployment | — | — |
| S9 Style/Preferences Rules | `pm-spec-critique` (spec-style only) | partial |
| S10 Legacy Docs Bootstrap | `explain-codebase` + `feature-spec` | ❌ (no composition) |
| S11 Anti-Pattern Test Suite | `intent-verification` skill §Worked Examples + §Anti-Patterns (docs, not test fixtures) | ❌ (not a fixture suite) |
| S12 Honest AI-Impact Telemetry | — | — |
| S13 Multi-Shot + Retrospective + ROI Selector | — | — |
| S14 Eval Calibration Loop | — | — |

---

## 9. Closing Note

The work to date has been **deep on the verification mechanic** (the standalone verifier and the orchestrated loop are well-specified and have a substantial POC behind them). The gaps are **not** a single missing half of the lifecycle — they fall into the **five shapes** named in §0: *upstream* (asking clarifying questions before code), *sibling evals* (tech-debt, security, test-quality, style composed alongside intent), *inside the core* (grounding, regression fixtures), an *enveloping pattern* (multi-shot generate-N → pick-best), and *downstream* (calibration, trust staging, AI-impact telemetry, PR Learn).

That shape is consistent with the customer-discovery doc's own message: **start with the verification core because it is the most concrete and most asked-for, and let the surrounding shapes follow once the core is calibrated and trusted**. The recommendations in §7 are arranged accordingly — upstream first (A: the clarifying-question loop makes the existing verifier *usable* on more PRs), sibling evals second (B: composes assets that are already written), downstream third (C: requires fixtures + measurement that only become valuable once A and B are in place), and the internal/enveloping shapes deferred to (D) as high-leverage but lower-urgency.

> **Net assessment:** ~45% of the customer-described problem surface is covered by the current specs + POC, but those 45% are **the right 45%** to have built first — they are the verification primitive everything else (upstream, sibling, internal, enveloping, downstream) composes on top of.
