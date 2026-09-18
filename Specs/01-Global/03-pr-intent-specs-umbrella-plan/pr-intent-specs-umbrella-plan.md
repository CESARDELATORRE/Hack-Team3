# PR Intent Specs — Umbrella Plan

> **Purpose.** One-page map of the PR Intent Verification spec family — what is committed, what is proposed, what is on the horizon. The authoritative source for strategic doctrine is the **north-star strategy doc**; this umbrella plan exists to make the spec family browsable and to surface where each doc fits in the strategy phases.
>
> **North star:** [`../02-pr-intent-verification-strategy/pr-intent-verification-strategy.md`](../02-pr-intent-verification-strategy/pr-intent-verification-strategy.md) — defines bidirectional drift, hub-and-spoke orchestration, finding classifications (BEHIND / AHEAD / CONFLICT / ALIGNED), iteration discipline, and the Phase 1 / 2 / 3 roadmap.
>
> **Folder convention.** Cross-cutting strategy and platform docs live under [`Specs/01-Global/`](..). Feature-scoped specs (one feature per folder) live under [`Specs/02-Features/`](../../02-Features). Customer-discovery and other research artifacts live under [`Specs/01-Global/01-pr-intent-customer-discovery/`](../01-pr-intent-customer-discovery/).

---

## The strategy in one paragraph

Specs and code drift continuously as features evolve. The PR Intent Verification platform detects that drift **bidirectionally** (the spec can be ahead of the code *or* the code ahead of the spec), filters out architecturally insignificant noise, and routes what remains to the right worker — code work to a coder, doc work to a PM — with a human in the loop on every conflict. The work is sequenced in three phases: Phase 1 ships the standalone verifier and the resolution-loop coordinator on local agents; Phase 2 invests in the platform foundation (verdict contract, eval trust stages, host adapters) so the same checks can run anywhere the changeset lives; Phase 3 moves upstream to elicit intent *before* the code is written. The full reasoning lives in the [strategy doc](../02-pr-intent-verification-strategy/pr-intent-verification-strategy.md); this umbrella plan only maps the artifacts.

---

## Spec family by phase

### Phase 1 — Local intent verification (committed, shipped)

The two specs that are **implemented** today, each backed by a custom agent + supporting skill.

| Spec | Status | Implementation |
|------|--------|----------------|
| **[F1 — PR Intent Verifier Agent](../../02-Features/F1-pr-intent-verifier-agent/pr-intent-verifier-agent-specs.md)** | IMPLEMENTED | `.github/agents/pr-intent-verifier.agent.md` + `.github/skills/intent-verification/SKILL.md` — standalone single-pass code ↔ spec auditor producing one `Run<N>` report (markdown + YAML 1.1). Read-only, leaf agent, no dependencies on other custom agents. |
| **[F2 — PR Intent Verification Workflow](../../02-Features/F2-pr-intent-verification-workflow/pr-intent-verification-workflow-design.md)** | IMPLEMENTED | `.github/agents/pr-intent-resolution-coordinator.agent.md` — hub-and-spoke coordinator that wraps the verifier in a resolution loop: triage findings, run human approval gates, dispatch code work to `@Collaborative dev lead` and doc work to `@PM` in parallel, re-verify, iterate until `ALIGNED` (typical 1–3 iterations; hard cap 5). |

Both specs are kept aligned with the implementation as it evolves — the agent and skill files are the operational source of truth; F1 and F2 are the design-of-record.

### Phase 2 — Platform foundation + eval trust (proposed)

The strategy commits to *direction* here; specs do not yet exist. Proposed pieces:

- **Verdict contract & doctrine** — extract the verifier's YAML 1.1 contract into a shared envelope (`{check, status, evidence, confidence, suggested_action}`) so other check types (security, debt, tests, style) can plug into the same coordinator. Companion: doctrine docs that codify discrete-per-check evaluation, iterative-loop convergence, and deterministic work outside the LLM.
- **Eval trust stages** — a per-check policy file that progresses each check from `silent` → `advisory` → `gated`, with feedback capture, promotion criteria, and metrics.
- **Host adapters** — surface the same checks across CLI, VS Code, ADO, GitHub, and other hosts that own the changeset at different lifecycle stages (local edit → pushed branch → draft PR → open PR). The validation logic stays the same; only the surfacing host and the authorized consequence change.
- **Codebase-grounding digest** — artifact-driven facts (architecture, patterns, conventions) the verifier can lean on for repos where intent docs are sparse or legacy.

These items are listed as **proposals**, not commitments. They will turn into individual `Specs/01-Global/` or `Specs/02-Features/` specs only after the strategy doc explicitly graduates one of them into the active roadmap.

### Phase 3 — Upstream intent elicitation (horizon)

Outside the current implementation roadmap, but called out in the strategy as the long-term direction:

- **Pre-code intent elicitation** — Socratic loops that help the user articulate intent *before* they start coding, so the verifier has something concrete to verify against from the first commit.
- **Multi-check fan-out** — the same coordinator pattern, but routing across multiple check families (intent + debt + security + tests + style) with shared trust-stage policies.

Phase 3 is intentionally vague at the spec level — it is informed by customer-discovery research and will be specified only when Phase 2 foundations are in place.

---

## Where to find things

| You're looking for | Look here |
|--------------------|-----------|
| The strategy / why we are doing this / phase commitments | [`Specs/01-Global/02-pr-intent-verification-strategy/`](../02-pr-intent-verification-strategy/) |
| The verifier spec + agent + skill (Phase 1, F1) | [`Specs/02-Features/F1-pr-intent-verifier-agent/`](../../02-Features/F1-pr-intent-verifier-agent/) · `.github/agents/pr-intent-verifier.agent.md` · `.github/skills/intent-verification/SKILL.md` |
| The coordinator + workflow design (Phase 1, F2) | [`Specs/02-Features/F2-pr-intent-verification-workflow/`](../../02-Features/F2-pr-intent-verification-workflow/) · `.github/agents/pr-intent-resolution-coordinator.agent.md` |
| Customer-discovery raw signal feeding Phase 2 / 3 | [`Specs/01-Global/01-pr-intent-customer-discovery/`](../01-pr-intent-customer-discovery/) |
| This umbrella map | (you are here) |

---

## Open candidates (not yet committed)

Items that may justify a future spec once Phase 1 has more telemetry. None are scheduled; each will become a real spec only when the strategy doc explicitly adds it to the roadmap.

- Verdict-envelope contract spec (Phase 2 foundation)
- Eval trust-stage policy spec (Phase 2 foundation)
- Codebase-grounding digest spec (Phase 2 — artifact-driven facts for sparse-intent repos)
- Pre-code intent elicitation spec (Phase 3)
- Multi-check coordinator spec (Phase 3)

> **Process for graduating an item.** Update the strategy doc first (Phase X commitment + acceptance criteria). Then file a new spec under the appropriate `Specs/0X-…/` folder and link it from this umbrella plan. The strategy doc is always the north star; the umbrella plan reflects what has been graduated, not what might be.
