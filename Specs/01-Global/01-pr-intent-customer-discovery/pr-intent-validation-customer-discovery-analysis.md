# PR Intent Validation — Customer Discovery Analysis

**Author:** Cesar de la Torre
**Date compiled:** 2026-05-22
**Discovery window:** May 15 – May 21, 2026
**Source data:** Microsoft Teams meeting recordings & transcripts (5 interviews)
**Scope:** Validate the problem of spec ↔ code drift in AI-assisted PRs, surface customer needs, and shape the PR Intent Validation workstream that will integrate with the PR Lifecycle / AI Code Review roadmap.

> All customer quotes in this document are **verbatim** from meeting transcripts. Interviewer (Cesar de la Torre) lines were intentionally removed so the synthesis reflects what the customers actually said.

---

## 1. Executive Summary

Across five interviews in five different orgs (PR Lifecycle, Engineering Systems / Office, ODSP, Talos / autonomous-dev, Outlook), a consistent picture emerged:

- **The problem is real, high-value, and growing — but workload-specific.** Teams shipping AI-generated PRs at scale (PR Lifecycle for CodeQL auto-fix, ODSP mobile, Talos autonomous dev) feel acute pain from spec ↔ code drift. Teams doing narrow, mechanical, non-business-logic PRs (Tiago, Office monorepo maintenance) feel almost none.
- **Iterative loops + human-in-the-loop beat single-pass validation.** Customers explicitly prefer a coordinator/coder/PM-style loop over a static report. Where a single-shot validator says "this might be wrong", a loop fixes the work before a human is paged. But **humans must remain in the loop** for hard reasoning domains (concurrency, distributed systems, subjective design taste).
- **The most-asked-for new capability is a "Socratic / clarifying-questions" phase** — getting the model to extract tacit human knowledge **before** writing code, rather than catching drift after the fact. This is related to the "iterative loop + human in the loop" while balancing "specs/docs <--> code" when code is behind or further.
- **Discrete, evidence-based evals beat monolithic prompts.** The two customers who have actually built eval pipelines (PR Lifecycle with 3 separate validations; Talos with discrete per-**check** evals) converged independently on the same architectural insight: **multiple narrow evals — one per check** — outperform a single mega-prompt that "poisons context". *A **check** here means one specific thing being evaluated about the change — e.g. "does the code match the stated intent?", "does it introduce a known security issue?", "does it add the kind of tech-debt that hurts the next change?". Each check gets its own narrow eval rather than being bundled into one giant prompt.* ODSP reaches the same insight architecturally — using a deterministic Architecture Explorer to ground a separate PR-comment agent — rather than via explicit per-check evals.
- **Docs/specs are the chronic upstream defect.** Three interviewees independently surfaced that "the spec/docs lag the code by months" (Shreyansh), "developers don't provide spec details in PRs" (Jyothi), or "today if I just have a feature… it can try to infer the intent from it, but it's less specific" (David) — meaning any intent validator must answer: *intent against what?*. But this is a culture issue. PMs+specs need to be closer to the devs+repos.
- **Trust in evals is the gating constraint.** Even the customers most bullish on automation (David Coulter / Talos) refuse to gate merges on evals until they trust them. David's measured **intent-eval** accuracy on ~100 samples over the last month is *"maybe 60%, maybe"* — applied specifically to the intent check. This is the single biggest blocker to value.


**Bottom line:** the PR Intent Validation workstream is well-aligned with where multiple Microsoft teams are already heading. The opportunity is to provide a **discrete, multi-check, looped, low-trust-tolerant validation framework** that handles the "intent against what?" problem (spec/docs bootstrapping & sync) as a first-class concern, not an assumption.

---

## 2. Methodology

| Aspect | Detail |
|---|---|
| Interview format | 30–45 min Microsoft Teams 1:1 / small group, semi-structured |
| Recording | Full audio + transcript via Teams |
| Extraction | WorkIQ M365 Copilot — verbatim transcript per interviewee, interviewer lines stripped |
| Analysis | Per-customer summary → cross-cutting themes → problems → solutions |
| Bias controls | (1) Quotes preserved verbatim; (2) "no signal" responses (Tiago) explicitly retained as falsification evidence; (3) customer ideas separated from interviewer's hypotheses |

**Sample composition (5 interviews across 5 orgs):**

| Org / Domain | Workload type | AI-PR usage today |
|---|---|---|
| PR Lifecycle (internal platform) | Automated PR factory (CodeQL auto-fix, etc.) | Heavy — they *are* the producer |
| Engineering Systems / Office OMR | Compiler/infra maintenance, mechanical refactors | Moderate — 20-variant batch experiments |
| ODSP (Office mobile, Android) | Feature work on a monorepo app | Heavy — custom PR agent + architecture explorer |
| Talos (autonomous dev workflows) | End-to-end autonomous coding pipelines | Heavy — building the orchestrator itself |
| Outlook (prev. distributed systems framework) | Complex framework w/ deep invariants | Heavy — agent swarms for PR review |

---

## 3. Per-Customer Summary Table

> Each row = one interview. "Customer voice" column paraphrases only what the customer themselves said (not Cesar).

| # | Date | Interviewee(s) | Org / Context | Current Practice (customer voice) | Headline Pain Point (customer voice) | Top Idea / Ask (customer voice) | Drift = Real Problem? |
|---|---|---|---|---|---|---|---|
| 1 | May 18 | **Jyothi Marreddy** (+ Rupam Mittal observing) | **PR Lifecycle** — building a code-validation plugin for auto-generated PRs (CodeQL scenario, soon 5+ more) | Already runs **3 validations** on auto-generated PRs: (1) intent vs. issue, (2) test coverage of changes, (3) scenario-based guideline conformance. Cycle 2 = single-pass comment; **Cycle 3 (this cycle) = loop**: validate → feed back to Agency → 2–3 iterations → final comment. | "Generally in PR, developers doesn't provide spec details… how will we know which spec doc we want to update?" — **specs are not attached to PRs**, so intent validation has nothing to anchor to. | Make the validation plugin a **sub-agent inside Copilot Code Review** so it works on any human PR, not just auto-generated ones. Link PRs to work items as the source of "intent." | ✅ Yes — already building it |
| 2 | May 20 | **Tiago Macarios** | **Engineering Systems / Office OMR** — compiler & infra maintenance, ~4,000 devs on the same monorepo | Very narrow, mechanical PRs (e.g., compiler upgrades). Reviews locally with SDVDiff. Strict one-commit-per-iteration in ADO. **Generates 20 variants** of a change and asks the model to write a retrospective + pick the best ROI. | **Drift is NOT his problem**: *"it's very rare that they drift"*, *"it usually works fine"* — because his work is mechanical. **His** pains: (a) model over-optimizes for corner cases and "loses itself" (e.g., tried to write a C parser to solve a problem it was assigned), (b) different reviewers have **subjective opinions** so the workflow must be re-tuned per area. | A **learning loop**: model should update its own prompt from PR feedback so reviewer preferences propagate. Some way to encode reviewer/area opinions without writing per-area specs (which would take "years"). | ❌ No — *negative signal: workload-specific* |
| 3 | May 21 | **Shreyansh Agrawal** | **ODSP** — Android app on Office Mobile Repo (monorepo, terabytes of code, ~500MB ship) | Built a custom **PR agent + "Architecture Explorer"**: a deterministic tool that builds a full call-graph from build artifacts (classes, APK, SOs) so the agent knows "who calls what" in the actual shipped app. PR agent runs as a **cron every 5–10 min**, picks up diffs, comments on PR. | **Docs lag code by 2–3 months.** *"Humans are bad to follow instructions to say, hey, every PR you need to keep the [docs] up to date and whatnot."* PR assistant on OMR runs at monorepo root → has no app-level context → comments are *"very, very shallow."* Bootstrapping structured docs (specs/figmas folders) on a legacy app is *"close to impossible."* | (1) **Docs = source of truth, code follows** — auto-update docs in same PR; (2) **PR Learn loop** (idea borrowed from ODSP web photos team): after every merge, a pipeline retrospects "what could have been better" and updates skills/agents/docs; (3) put docs at **agency workspace root**, not in folders — agents.md/cloud.md in folders pick up "very opportunistically". | ✅ Yes — and tightly coupled to a docs-sync problem |
| 4 | May 21 | **David Coulter** | **Talos** — autonomous dev workflows (he is the builder/sponsor) | Built **Talos**: a state-machine flow orchestrator that runs **deterministic steps outside the LLM** and only calls the LLM for non-deterministic work (writing code, evals). Approve work → it picks up, runs the loop, inserts evals as gates, pushes PR, manages PR lifecycle. Started with a **security/safety eval**, now expanding to intent + tech-debt. | **Trust gap in evals.** *"I've looked at 100 of these over the last month and maybe 60% of them are right. Maybe."* Intent eval is *"one of the hardest facets I've been trying to land."* PR velocity metrics are *"interesting but extremely gameable."* Tests can pass with **all source code deleted** — *"if you have passing tests when none of your source is present, there is a problem with your tests."* | **Discrete evals per facet, never one mega-prompt** (*"poisons the context"*). Add a **tech-debt eval** (same function written 12 times, passes tests, intent passes — *"but it's bad"*). **Test for anti-patterns**, not just patterns. Use **Gherkin + execution engine** so intent eval has something concrete to compare against. **Share Talos broadly**; align on a **June joint demo**. | ✅ Yes — and he's a potential partner / co-builder |
| 5 | May 21 | **Taylor Williams** | **Outlook** (joined ~2 weeks ago); speaking from prior team — **distributed systems framework** with heavy invariants & couplings | No formal intent-validation phase. **Tests are the de-facto enforcement**: 80%+ unit-test coverage, end-to-end tests, **fuzz tests** that walk the game tree of scenarios — invariants become implicit because the code can't merge unless tests pass. Already uses **agent swarms with personas** for PR review *before* humans. | Two categories of desynchronization causes: (a) **tacit-knowledge gaps** (un-documented invariants the model can't infer) — his team's answer is to *"crystallize that knowledge through tests"*; (b) **capabilities gap** (model is "jagged" — bad at concurrency, distributed systems, design taste). Orthogonally, two types of *failures* an intent-validation cycle would catch: **drift / natural probabilistic mistakes** (*loops can fix these*) and the **jagged-intelligence** failures (*"it doesn't matter how many loops you go in"*). Multi-agent flows can dangerously **obfuscate** the fact that the model is just bad at the task. | **Socratic / clarifying-questions phase** — instead of "is my plan good?", force the model to **build an open-question list** and interrogate the human to extract tacit knowledge upfront. *"That's actually how you induce humans to put their knowledge down in writing."* **Keep step 4 (human review) for years** — human-out-of-the-loop loses entire domains. | ✅ Yes — but solution must respect model capability limits |

---

## 4. Per-Customer Deep Dive

Each subsection follows the same structure: **Profile → Verbatim Highlights → Pain Points → Current Practices → Ideas & Asks → Highlights / Conclusion**.

### 4.1 Jyothi Marreddy & Rupam Mittal — PR Lifecycle Team (May 18)

#### 4.1.1 Profile
The PR Lifecycle team is building the platform-side capability for PR validation across Microsoft. They have multiple workstreams (AI code review on human PRs, intent validation, test coverage, scenario checks, eventing). Jyothi is leading the **code-validation plugin** for **auto-generated PRs**. Rupam was largely an observer in this call.

#### 4.1.2 Verbatim Highlights (Jyothi)

> *"PR lifecycle has multiple work streams in itself… last cycle, what we have done is for auto-generated PRs, like a CodeQL PR…we are verifying intent validation under test coverage."*

> *"Intent validation means, okay, let's say PR is meant to fix a particular SQL issue. So the fix is related to that only. Fix is not deviating from that intent."*

> *"For #3, there are some guidelines for CodeQL PR… let's say they'll specify some guidelines. Okay, my PR has to follow these followings. This is for all CodeQL PRs, not only for that particular PR."*

> *"So in cycle three, what we are planning is instead of giving comment in the first pass itself, we will do the validation and give that feedback to agency. So we will loop this for two to three times, then we will give the comment."*

> *"Once it is successful in one plugin, like a CodeQL plugin, once it is successful, then we will include this for like five more scenarios, five more agents. If it is running fine, then we are thinking to add it generically to the agency flow itself."*

> *"The one we created for our intent-based validation, that one we can create as a sub agent, so this can be used for any PR. So today it is being used for auto-generated PRs, right? So we can make it as a sub-agent and include it in the co-pilot code review."*

> *"But generally, in PR, developers doesn't provide spec details, right? So how will you know this is the spec doc for a PR? Generally whenever a PR is getting created, spec doc won't be provided in the description or comments, right? So, how will we know which spec doc we want to update?"*

> *"Maybe it has to come from work item, maybe it should be linked to work item, something like that."*

#### 4.1.3 Pain Points (customer voice)
- **No spec attached to PRs**, so the intent validator has nothing to compare against unless it can infer from the work item / issue.
- Spec **update** as a downstream PR action is technically wanted but operationally undefined (whose responsibility, what doc, where?).
- Validation as a one-shot comment is too weak — they want the model to **self-correct before commenting**.

#### 4.1.4 Current Practices
- Three validations run today on auto-generated PRs (CodeQL scenario): intent vs. issue, test coverage, scenario-rule conformance.
- Cycle-2 design: single-pass → comment on PR.
- Cycle-3 (in progress this week): **loop** validation → Agency feedback → 2–3 iterations → final comment.
- Delivery vehicle: an **Agency plugin** that integrates wherever Agency runs (Agency hub, CLI, Breeze).

#### 4.1.5 Ideas & Asks
- Convert the existing intent-validation plugin into a **sub-agent of Copilot Code Review** so it works on **any human PR**, not just auto-generated ones.
- Use **work items as the intent anchor** when no spec is attached to a PR.
- Generalize the cycle-3 loop (CodeQL today → 5 more scenarios → eventually Agency-wide).

#### 4.1.6 Highlights / Conclusion
- **This is the most validated, in-flight implementation of intent validation at MSFT today.** The PR Lifecycle plan to deliver a generalized sub-agent in Cycle 3 is a direct, immediate integration target for the workstream.
- The "no spec on PR" objection is **the single most important real-world constraint** any intent-validation product must accept. Falling back to work items, issue descriptions, or PR titles is mandatory.
- Their planned loop architecture (validate → feedback → iterate) **matches the architecture Shreyansh and Talos converged on independently** — strong cross-team signal.

---

### 4.2 Tiago Macarios — Engineering Systems / Office OMR (May 20)

#### 4.2.1 Profile
Tiago works on the Office Monorepo (OMR) — a federated codebase where Word, PowerPoint, Excel, and ~4,000 engineers contribute. His own work is **maintenance / mechanical** (compiler upgrades, fixing breakage), not business logic.

#### 4.2.2 Verbatim Highlights

> *"A lot of the changes I do are extremely mechanical."*

> *"Usually when I get a prompt or a set of prompts working, it's very rare that they drift… It usually works fine."*

> *"What I've been working lately… I want the model to update the prompt with PR feedback. I think it's different than the problem that you're trying to solve, but that is a problem that I have now."*

> *"The way I'm going about this is I tell it to do the change 20 times… So for example, right now I'm working on functions. So I tell it to pick 20 functions, do the change, create a PR. The first thing I actually asked the model to create a retrospective… and then I tell it to like pick the one with the most ROI."*

> *"I've been seeing cases where sometimes it tries to optimize for the corner case, and then it completely lose itself, and I lose itself, and I have to intervene. So, as of two days ago, it was basically trying to write a C parser to figure out one of the problems that I was trying to solve, so that's pretty annoying."*

> *"Devs sometimes disagree with how the code should be. And how do you back feed that into the model? Let's say you know you're working with a dev for a couple of weeks, and you get the workflow nailed down for how that dev works, but then you're doing the same change in another part of the code, and the reviewer for that part of the code has different opinions. So you kind of have to tune the whole workflow to how the dev expects the code to be on that part, even though the change is the same."*

> *"Like, you know, Office is millions of lines of code, we have about 4,000 devs working on it for years. Like, I know Alex has been doing a great job kind of writing down those guidances for AI in Excel. I don't see all the other apps doing the same. I think that will take probably years."*

#### 4.2.3 Pain Points (customer voice)
- **Drift is NOT a problem for him** — explicit falsification signal for the universal-pain hypothesis.
- The model **over-optimizes corner cases** and goes on tangents (e.g., started writing a C parser to solve the assigned problem).
- **Subjective reviewer opinions** vary by area of the monorepo; a workflow tuned to one reviewer fails for another.
- Producing per-area AI guidance (the "Alex in Excel" model) is **funded only for one team** — generalizing it across Office will take years.

#### 4.2.4 Current Practices
- Narrow, mechanical PRs only.
- Local review with **SDVDiff**.
- Strict one-commit-per-iteration discipline in ADO so reviewers can step through diffs cleanly.
- **20-variant batch + retrospective + ROI selection** as his core productivity pattern.

#### 4.2.5 Ideas & Asks
- A **feedback-learning loop**: prompt evolves from PR feedback so it gets better the more it ships.
- Some way to encode **subjective reviewer preferences** without per-area specs (because per-area specs aren't going to happen at OMR scale).

#### 4.2.6 Highlights / Conclusion
- **Negative signal is valuable.** Tiago is the existence proof that "AI PR intent drift" is **not a universal Microsoft problem** — it is concentrated in business-logic-heavy, novel-feature-heavy, complex-domain work. The starter pack / product narrative must not over-claim universality.
- His real pain — **subjective reviewer disagreement** — is adjacent to intent validation but distinct: it argues for a *style/preferences-as-rules* check alongside intent.
- His **20-variant batch** pattern is a creative use of throwaway parallelism that could itself become a starter-pack capability ("multi-shot + retrospective + ROI selector").

---

### 4.3 Shreyansh Agrawal — ODSP / Office Mobile Repo (May 21)

#### 4.3.1 Profile
Shreyansh works on an ODSP Android app inside the Office Mobile Repo (OMR — terabytes of code, ~500MB ships). His team has invested in a **custom PR agent + Architecture Explorer** because the platform-level PR assistant doesn't have enough app-level context.

#### 4.3.2 Verbatim Highlights

> *"The code should follow the docs. So if engineers are supposed to update the docs, the system is supposed to update the code. So when you're talking about catching these gaps in the PR, the doc should always be updated as part of the PR is what we are believing at this point of time."*

> *"At this point of time, our docs lag. Docs are a snapshot of probably like two or three months ago. Not being able to have the docs in sync with the code is the major problem that we are facing and definitely it kind of impacts the outcome in a big way."*

> *"Humans are bad to follow instructions to say, hey, every PR you need to keep the docs up to date and whatnot."*

> *"One of the architects in our team basically created something like an architecture explorer. It isn't an agent or a skill, but it is basically a deterministic tool. What that tool has is it has all the generated artifacts at build time… it creates a link graph that, hey, who's calling what? So when the APK, when the app is actually generated, the architecture explorer exactly knows who's calling what."*

> *"The PR agent that we have basically leverages the architecture explorer and it is able to generate outcomes like, hey, you are modifying this, but I don't even think that this is the right place to make these modifications without even having to look for the documentation."*

> *"Just a cron job running every 5 to 10 minutes, picking up diffs since the last run… leveraging the architecture explorer to get more context, build architecture context, and then add comments."*

> *"The PR assistant works on the root context of the mono repo. And whenever the PR assistant is working on the root context of the mono repo, it does not have any insight about the app it is reviewing. The comments that we get from PR assistant is very, very shallow, does not understand the app or the scenarios at all."*

> *"The agents.md and cloud.md hasn't worked very great for me. In the last three months, when we seeded those documents, the agents picking those up has been very opportunistically… I have found it much more easier for me when the documents or the repository of documents is at the agency root or wherever I'm starting that workspace."*

> *"For an existing app like us, it is next to impossible to populate those folders today. For all new feature hours, for all new Figma, all those things, it is easy to populate, but bootstrapping those folders with an existing standing app is close to impossible."*

> *"I believe that this [iterative loop] is more closer to what I would believe can keep both in sync. It can't be [one-shot], I mean, like even for long running sessions that we have locally, the docs and the implementation go back and forth. And it does take multiple iterations… definitely a system like this would basically reach to close to 100 [%] more often than the previous system."*

> *"One of our partner teams, ODSP web photos team, while they were keeping the docs and code in sync… had a pipeline which was called PR Learn. So irrespective that the docs was kept in sync in the PR, every PR will also trigger a PR Learn loop, which is not just about keeping the documents in sync, but also about, hey, what mistakes I made, what I could have done better, and come back and update the skills and agents, or at least capture the recommendations, but also definitely did the documentation gap updates."*

> *"Most of the teams which have achieved 10X velocity, as far as I know, have decided or taken a leap of faith to rewrite their entire code base. Are there examples in Microsoft that have achieved that velocity with an existing code base without rewriting it?… The challenge is that that legacy code, code that was written 20 years back, still part of the app — just lack of documentation and so much historical knowledge and context that one engineer in the team might have it, and it's not documented and very difficult to get to."*

#### 4.3.3 Pain Points (customer voice)
- **Docs lag code by 2–3 months** — direct quote, treated as the *major* problem.
- **Humans can't be trusted to keep docs in sync** — must be system-enforced.
- **Mono-repo blocks auto-merge / nightly pipelines** for doc sync.
- **Platform PR assistant is shallow** because it lacks app-level context (runs at monorepo root).
- **agents.md / cloud.md inside folders are picked up "very opportunistically"** — not consistently.
- **Legacy-app bootstrapping** of structured docs (specs, Figma, etc.) is *"close to impossible"*.
- Open question: **is the AI-first starter pack meant for legacy codebases at all, or only greenfield rewrites?**

#### 4.3.4 Current Practices
- Custom **PR agent** that runs as a **cron every 5–10 min**, picks up diffs, uses the architecture explorer for context, comments on the PR.
- **Architecture Explorer**: deterministic, build-artifact-driven call graph — gives the agent *grounded* knowledge of who calls what in the actual shipped APK.
- Docs stored at **workspace/agency root** rather than folder-local because of pickup reliability.

#### 4.3.5 Ideas & Asks
- **Docs as source of truth, code follows.** Every PR should also update docs (in the same PR, possibly via a docs sub-PR if separated repo).
- **PR Learn loop** (post-merge retrospective pipeline) — emerges from the partner team's success and is described as a *breakthrough idea* in this discovery.
- **Iterative coordinator + coder + PM loop** strongly preferred over static reports.
- Validate whether the AI-first starter pack covers **legacy / non-rewrite** scenarios.

#### 4.3.6 Highlights / Conclusion
- Shreyansh independently arrived at the **same iterative-loop architecture** as PR Lifecycle and Talos.
- His **Architecture Explorer** is a near-perfect example of the David Coulter principle: *use deterministic tools to ground LLM context, don't ask the LLM to infer*.
- The **PR Learn** loop is the standout new idea from this discovery — addresses skills/agents drift over time, not just one-shot PR drift.
- His **legacy-codebase question** is a strategic challenge the workstream should answer head-on.

---

### 4.4 David Coulter — Talos / Autonomous Dev Workflows (May 21)

#### 4.4.1 Profile
David's role is to "facilitate AI use and smartly building tools, not just smart tools." He built **Talos**, a state-machine flow orchestrator for end-to-end autonomous dev workflows. He has been running hundreds of test PRs through evals and has the most empirically grounded view of where evals fail.

#### 4.4.2 Verbatim Highlights

> *"I think autonomous is where we actually want to be able to go. Where I just drop a here's a feature ADO, and I mark it as approved, and I know that the pieces are going to just pick it up and go code the whole thing, and I'll check back in when it's done. I don't even want to have to go vibe engineer any of that… I don't want to be the bottleneck."*

> *"We can tell if you're using tokens, we can tell you're using CLI or agency or sweet agent or whatever, like there's some metrics, but it doesn't tell us if it's having impact… [PR velocity metrics] they're interesting, but they're also extremely gameable. And it also doesn't tell me how much AI is being used. It's just telling me velocity and velocity can be achieved in multiple ways."*

> *"We spent a year building smart tools. But it's not about building smart tools, it's about smartly building tools."*

> *"If you take a deterministic task and you stuff it through an LLM, you inherently make it non-deterministic. You reduce trust and you introduce variation in the response or the outcome… which is notionally not the direction we want to go. We want to improve trust, not reduce trust."*

> *"If you take deterministic work and you have the AI execute that deterministic work, you're actually burning a lot of money unnecessarily. Because if you're telling it to push the PR and check on the PR and do all these things, well, those are very deterministic outcomes."*

> *"We basically built a flow orchestrator. That is a state machine that helps us run those full flows. The deterministic steps are outside of calling, say, agency or co-pilot CLI. They sit at the point where I need to do something in a non-deterministic way, like write code… If it fails, I might want to take the outcome from it and send it back through a loop."*

> *"One of the final stages should be: did what it built without me involved, did what it built match what the task I assigned it to go build was? And so that's where I think the validations or the evals come in."*

> *"I mean, you could produce an eval today. I don't trust it. Like until I see it work a bunch and I know that it's catching the right stuff and it's actually catching things I as a human or a team of five humans doing reviews would catch, I don't have trust in it. And I don't want it actually blocking or allowing work to go through until I have a high degree of trust within that process."*

> *"As soon as you hit the stage where the evals are trusted, I stop having to look at the code and I can actually gate off of the evals. But we're not there yet."*

> *"I think it's not going to be 1 eval. I think it's going to be a number of evals that look at different facets. Trying to have today with the models having a singular evaluation of the entire code block against all of those facets poisons the context. And so you end up needing very discrete evals for each of the facets that are important to you."*

> *"If you try to mesh them all into one massive prompt, you lose the granularity, you blow the context, and it loses the thread of the story. But doing it very focused facets of areas, you get better results."*

> *"I've been messing with the intent one, and I'll tell you, that's one of the hardest facets I've been trying to land. I've looked at 100 of these over the last month and maybe 60% of them are right. Maybe."*

> *"One of the facets I think we probably want to introduce is probably a tech debt evaluation. Great, you built code. Are you putting us in a bad place for the next line of code I need to write? I've been running hundreds of these tests, and I've seen it write the same function 12 times. It passes the tests because the test tests the outcome, but that's a mess to maintain. It'll pass everything we've thrown at it today… intent passes, but it's bad."*

> *"Or you'll tell it to write tests to go with the code and it'll write the tests. The tests pass without the code and then it considers itself done. Delete 100% of your source code, leave the tests, run the tests, see what tests pass when there's nothing there. If you have passing tests when none of your source is present, there is a problem with your tests. We have to start thinking not in just the pro pattern, but we have to be thinking what the anti-pattern looks like so that we can test for it."*

> *"Using something like Gherkin plus an execution engine gives me a better position where I can actually do an intent validation at the end of it."*

> *"My entire role in this new team is basically to help facilitate AI use and smartly building tools… That's why I went and built Talos. We can't be the only ones that are doing this, so I'd love to share wider. I'm happy to share it until something better comes along."*

#### 4.4.3 Pain Points (customer voice)
- **Trust gap**: ~60% perceived accuracy on intent evals → can't be used as a gate.
- **PR velocity metrics are gameable** and don't measure AI impact.
- **LLMs make deterministic work non-deterministic, costly, and lower-trust** when they're used for tasks that should never have been LLM-driven.
- **Tests can pass with 0% of source code** — the testing paradigm itself is broken if we don't test for anti-patterns.
- **Models write the same function 12 times** → tech-debt accumulates invisibly because intent and tests both pass.
- **Monolithic single-prompt evaluation "poisons the context"**.

#### 4.4.4 Current Practices
- **Talos** = state-machine orchestrator.
- Strict separation: **deterministic steps run outside the LLM**; LLM only for genuinely non-deterministic work.
- Loops with self-healing (failed deterministic step → feed back into LLM step).
- Has shipped a **security/safety eval** as the first check; building intent + tech-debt next.
- Eval results structured in a **standard PR-comment format**.
- Approve work item → Talos picks up → runs full loop → manages PR lifecycle.

#### 4.4.5 Ideas & Asks
- **Eval taxonomy**: intent, security/safety, code quality, tech-debt, business/domain alignment — each as a **discrete narrow eval**.
- **Anti-pattern testing** (deletion test for test suites; same-function-N-times for code).
- **Gherkin + execution engine** as the structured intent input so the eval has a deterministic comparison target.
- **Trust-staged gating**: evals are advisory until they are independently verified to match human-team judgment, then can be promoted to merge gates.
- Wants to **share Talos** with PR Lifecycle and the workstream; aligned on a **June joint demo**.

#### 4.4.6 Highlights / Conclusion
- David is the single most aligned external partner identified in this discovery — both **philosophically** (discrete evals, deterministic-vs-non-deterministic split, trust-staged adoption) and **practically** (he wants to share Talos and co-demo).
- His **anti-pattern testing** insight is original and powerful — should become a starter-pack capability.
- His framing — *"smartly building tools, not building smart tools"* — could become the workstream's narrative anchor.
- The **60% intent-eval accuracy** number is the most concrete, quantified trust-gap evidence we have.

---

### 4.5 Taylor Williams — Outlook (prev. Distributed Systems Framework) (May 21)

#### 4.5.1 Profile
Taylor recently moved to Outlook but speaks from his prior team — a **distributed systems framework** with deep couplings and many invariants. Self-describes as *"fairly AGI-pilled"* but with hard limits.

#### 4.5.2 Verbatim Highlights

> *"There's no formal practice in a sort of very intentional way around a synchronization phase afterwards [for PR review]."*

> *"I kind of think of it as there's like 2 categories of things that can cause this kind of desynchronization. The first is a lack of tacit knowledge, meaning maybe it's team tribal knowledge, maybe it's invariants in the code that are not documented, maybe it's things that you didn't know you needed to explain to the model when giving it the initial plan. And then the second is sort of capabilities gap… they're really bad at certain things."*

> *"In my previous team, a lot of the challenge, at least in my area, was that our framework, which is a distributed systems library, is very complicated and it has a lot of couplings… So the way that we crystallize that knowledge is through tests. And I am, it's really just a fancy way of saying have a very, very rigorous suite of tests… 80 plus percent coverage in unit tests, in end-to-end tests, in fuzz tests. We had a very robust sort of fuzz test that would open up the game tree of possible scenarios and explore all of them. And we make sure that all those tests check the invariants that we need to not be violated. And so when you're describing a feature that you'd like from the model, you don't have to try to regurgitate and tell it about every invariant that you need it to uphold. Those are just implicit in the spec because it won't pass the tests unless it does so."*

> *"We have like review swarms, we'll have an agent swarm that goes over every PR before humans look at it. And you have various agent personas. Whether or not that's better than a single agent, I'm not trying to claim that, but at least we do a very deep multi-agent review step before humans look at it."*

> *"There are two types of failures that I see that I would wish an intent validation cycle would catch. The first is drift, which is just sort of like these things are somewhat probabilistic and they're prone to making mistakes. If you chain 100 tasks together, it's actually a very high likelihood that they make mistakes because each step has a 5% chance [of failing]. Those types of natural mistake making, that all you have to do is if you give it another couple times to look at it, it would catch the error. This is great for that."*

> *"But then there's the second category of things, which is things that the model has jagged intelligence and they're just strangely bad at certain things. A classic example I will use over and over again, it's never really gotten better… is thinking of reasoning about concurrency. Models are just really bad at certain types of multi-threading, parallelism. Distributed systems is multi-threaded problems on steroids. And they're really bad at it. And it doesn't matter how many loops you go through and how many times you ask it. The key is that you have a human in the loop."*

> *"You also don't need a human in the loop as much as I think a lot of people think. Step 4 here, I think is still very critical. As soon as you go human out of the loop, you will lose a bunch of domains. Greenfield web development… sure. But there's a bunch of kind of hard problems that they suck at. And I tend to think that multi-agent flows, if you're not careful, you're basically kind of obfuscating the fact that they're just inherently bad at this."*

> *"I have certain skill packs that I use, but those are very heavily biased towards upfront planning, super good test coverage, and then a step where the model will do like Socratic tutoring, where the model will essentially ask me very pointed questions. I will induce the model. I will convince the model to ask me repeated pointed questions to try to probe at my knowledge. And this isn't actually for my good. Socratic tutoring is one of the best ways to learn when an expert asks you questions and puts you on the spot. But it's actually in the other way. It's to get the model to ask questions that it doesn't know the answer to. And then for me to be able to provide that information in a very information-dense way in small amounts of time."*

> *"My recommendation is that for step 4 — instead of just saying, here's what I want to do, is it good? — instead, have it force it to ask open questions, like build an open question list or ask the user a series of questions so that it better understands. That's actually how you induce humans to put their knowledge down in writing."*

#### 4.5.3 Pain Points (customer voice)
- **Two-category drift model**: (1) tacit-knowledge gaps, (2) model capability gaps.
- **Capability gaps cannot be fixed with more loops** — concurrency, distributed systems, design taste.
- **Multi-agent flows can mask** the fact that the model is fundamentally bad at the task.
- **No formal intent-validation phase exists today** in his prior team — tests carry the load.

#### 4.5.4 Current Practices
- Rigorous test pyramid (80%+ unit + end-to-end + **fuzz tests** that explore the game tree of scenarios).
- **Agent swarms with personas** for PR review before humans look.
- Personal skill packs biased toward **upfront planning + test coverage + Socratic questioning**.

#### 4.5.5 Ideas & Asks
- **Socratic / clarifying-questions phase** — the model interrogates the human up-front to extract tacit knowledge, *before* writing code.
- Keep **step 4 (human review) for years** — it is the only thing protecting hard domains from quiet failures.
- Build skill packs with the bias: **upfront planning > catch-after-the-fact validation**.

#### 4.5.6 Highlights / Conclusion
- Taylor introduces the **most important conceptual addition** of the discovery: validation should partly **shift left to elicitation** (Socratic questioning *before* code generation), not just sit downstream.
- His **two-category drift taxonomy** (tacit-knowledge vs. capability gaps) is a sharp diagnostic frame and should be adopted in the workstream's vocabulary.
- His test-as-implicit-spec approach validates the broader "tests are the only real intent enforcement today" reality across multiple interviews.
- His **caution against multi-agent flows obfuscating capability gaps** is the most important counter-pressure to the otherwise dominant "more loops = more quality" narrative.

---

## 5. Cross-Cutting Theme Matrix

A "✓" = customer explicitly raised this; "~" = implied/adjacent; "✗" = explicitly disagreed.

| Theme | Jyothi (PR Lifecycle) | Tiago (OMR) | Shreyansh (ODSP) | David (Talos) | Taylor (Outlook) |
|---|---|---|---|---|---|
| Intent drift is a real, painful problem | ✓ | ✗ | ✓ | ✓ | ✓ |
| Discrete evals per check beat one mega-prompt | ✓ | ~ | ~ (architecturally; via separate tools, not labeled "evals") | ✓ | ~ |
| Iterative loops beat single-pass validation | ✓ | ✓ | ✓ | ✓ | ✓ (only for probabilistic drift, not capability gaps) |
| Specs/docs lag or don't exist for PRs | ✓ | ~ (per-area AI guidance gap, not PR-spec gap) | ✓ | ✓ | ~ (no formal post-coding sync; not framed as docs-lag) |
| Need a deterministic grounding tool (call graph / artifacts) | ~ | ~ | ✓ | ✓ | ✓ (tests as ground truth) |
| Trust gap blocks merge-gating today | ~ | ~ | ~ | ✓ | ~ |
| Tests carry the de-facto intent contract | ~ | ~ | ~ | ✓ (and broken) | ✓ |
| Human-in-the-loop is essential for hard domains | ~ | ✓ | ~ | ✓ | ✓ |
| Subjective reviewer/style preferences are a real source of "drift" | ~ | ✓ | ~ | ~ | ~ |
| Want Socratic / clarifying questions phase | ~ | ~ | ~ | ~ | ✓ |
| Want a post-merge "learn" loop | ~ | ✓ (prompt update) | ✓ | ~ | ~ |
| Tech-debt / anti-pattern eval is a missing check | ~ | ~ | ~ | ✓ | ~ |
| Legacy codebase / bootstrapping is hard | ~ | ✓ | ✓ | ~ | ~ |
| Want to partner / share / co-build | ✓ | ~ | ~ | ✓ | ~ |

**Reading the matrix:** **iterative loops**, **specs/docs gap**, and **discrete evals** are universally validated. **Trust gap, subjective preferences, Socratic-elicitation, PR Learn,** and **tech-debt eval** are uniquely strong signals from individual customers that the workstream should treat as first-class.

---

## 6. PROBLEMS, DOCTRINE, AND COORDINATION

> **Structure note (second-round audit, June 1, 2026).** An earlier draft listed 15 entries under a single "Problems to Be Solved" heading. On second pass, several entries were not customer-felt problems but architectural lessons (e.g., "mega-prompt poisons context") or workstream coordination items (e.g., "no shared integration surface"). Conflating these inflated the apparent problem count and produced misleading 1:1 problem-to-solution mappings — a "solution" that simply restates a design principle is not a solution. Three entries were also reframed because they pointed at constraints or symptoms rather than the underlying solvable issue (trust-gap → accuracy ceiling; loops-don't-fix-capability → capability-gap failures ship undetected; monorepo/legacy blockers → validators starved of app context). One new problem was added (P11 — no way to pick the best draft from N attempts) so that Tiago's productized pattern (S13) has a home.
>
> This section is now split into:
>
> - **6A. Customer Problems** — customer-felt pains that a feature or capability can resolve. Renumbered P1–P11.
> - **6B. Design Doctrine** — architectural commitments about *how* the workstream builds; not work items. Labelled D1–D3.
> - **6C. Workstream & Ecosystem Concerns** — coordination and measurement items that sit outside the product surface. Labelled W1–W2.
>
> A full renumbering map from the previous draft is in §9.4.

---

### 6A. Customer Problems

Ordered by signal strength (number of customer voices, severity, blocking power). Single-source items are tagged for follow-up validation.

#### P1. "Intent against what?" — there is no spec to compare a PR against
- **Signal strength:** 3 explicit voices + 2 adjacent (5/5 coverage).
- **Voices:** Jyothi (*"developers doesn't provide spec details… how will we know which spec doc?"*), Shreyansh (*"docs are a snapshot of probably 2–3 months ago"*), David (*"today, like if I just have a feature… it can try to infer the intent from it, but it's less specific"* — he uses Gherkin to make intent explicit *because* of this gap).
- **Adjacent voice:** Tiago — per-area AI guidance docs don't exist either (*"Alex's team is funded to do that… I think that will take probably years"*) — different from missing PR-level specs but a related upstream gap.
- **Why it matters:** Intent validation is meaningless unless there is a trusted, current statement of intent. Today, the spec is either missing, stale, or trapped in tribal knowledge.
- **Hardest sub-problem:** legacy codebases — Shreyansh: *"bootstrapping those folders with an existing standing app is close to impossible."*

#### P2. Intent-eval accuracy is too low to be load-bearing
- **Signal strength:** 1 voice, quantified — needs cross-team validation.
- **Voices:** David (*"I've looked at 100 of these over the last month and maybe 60% of them are right. Maybe"*).
- **Why it matters:** Earlier framings labelled the "trust gap" as the problem and pointed at S8 as the solution. That conflated *symptom* (low trust) with *cause* (accuracy ceiling). A deployment policy (S8) lets you operate around low accuracy without ever closing it; only an improvement mechanism (S14, new) actually moves the number. Both are needed.

#### P3. Capability-gap failures ship undetected because the loop reports success
- **Signal strength:** 2 voices.
- **Voices:** Taylor (*"it doesn't matter how many loops you go in"* — concurrency, distributed systems, design taste; *"multi-agent flows… you're basically kind of obfuscating the fact that they're just inherently bad at this"*), Tiago (*"loses itself… tried to write a C parser"*).
- **Why it matters:** Loop architectures repair probabilistic drift but can hide deep capability weaknesses. Without a way to flag and route high-risk domains to humans, the loop *increases* false confidence in domains the model is fundamentally bad at. Earlier framings made this an un-solvable constraint of the medium; reframed it is solvable — by domain-risk classification + mandatory human-in-the-loop for tagged domains, partially supported by S5 (Socratic) shifting the work left.

#### P4. Tests as the de-facto intent contract — but tests themselves can be wrong
- **Signal strength:** 2 voices (Taylor positive, David failure-mode).
- **Voices:** Taylor (tests crystallize invariants), David (*"tests pass without the code… delete 100% of your source code… if you have passing tests when none of your source is present, there is a problem with your tests"*).
- **Why it matters:** If we lean on tests for intent enforcement (which several customers already do), we inherit a different drift problem — tests that pass without the code, tests that test the model's own work.

#### P5. Tech-debt blindness — intent and tests can both pass while code rots
- **Signal strength:** 1 voice, but observed across hundreds of test PRs.
- **Voices:** David (*"I've seen it write the same function 12 times. It passes the tests because the test tests the outcome, but that's a mess to maintain… intent passes, but it's bad"*).
- **Why it matters:** A failure mode unique to AI-assisted workflows: redundant, low-quality but functionally correct code accumulates faster than humans can refactor.

#### P6. Subjective reviewer preferences vary by area
- **Signal strength:** 1 voice.
- **Voices:** Tiago (*"the reviewer for that part of the code has different opinions… you have to tune the whole workflow"*).
- **Why it matters:** Not classical "intent drift" but presents identically: the PR doesn't match the reviewer's mental model of "correct". Requires *style/preferences-as-rules* per area/owner (S9).

#### P7. Docs lag code, and humans cannot be trusted to keep them in sync
- **Signal strength:** 1 voice but treated as *the major problem* by that customer.
- **Voices:** Shreyansh (*"docs are a snapshot of probably like two or three months ago"*; *"humans are bad to follow instructions to say, hey, every PR you need to keep the [docs] up to date"*).
- **Why it matters:** The upstream cause of P1 in established codebases. Must be system-enforced, not policy-enforced.

#### P8. Validators starved of app-level context produce shallow output in monorepo / legacy settings
- **Signal strength:** 2 voices.
- **Voices:** Shreyansh (*"the PR assistant works on the root context of the mono repo… does not have any insight about the app it is reviewing. The comments… are very, very shallow"*), Tiago (OMR scale precludes per-area AI guidance — *"will take probably years"*).
- **Why it matters:** Earlier framings treated "monorepo and legacy" as environmental blockers (and therefore not solvable). The *solvable* form is: the validator lacks grounded, app-level facts about what the code actually does in the shipped artifact. Architecture grounding (S7) and one-time legacy bootstrap (S10) directly target this. Also absorbs the §5 matrix theme "need a deterministic grounding tool" (3 voices) which had no corresponding entry in the previous §6.

#### P9. Tacit knowledge gap — invariants the model can't infer from artifacts alone
- **Signal strength:** 2 voices.
- **Voices:** Taylor (*"tacit knowledge"* as one of the two causes of desynchronization), Shreyansh (legacy historical knowledge — *"one engineer in the team might have it, and it's not documented"*).
- **Distinction from P1:** P1 = no artifact of intent exists at all. P9 = an artifact may exist but is incomplete because the relevant constraints were never written down. P9 cannot be closed by better artifact resolution (S1) — it must be elicited (S5).

#### P10. Skills / agents / prompts themselves drift over time
- **Signal strength:** 2 voices.
- **Voices:** Shreyansh (PR Learn loop — *"capture the recommendations, but also definitely did the documentation gap updates"*), Tiago (*"I want the model to update the prompt with PR feedback"*).
- **Why it matters:** Intent validation is a moving target; the validator itself needs continuous improvement informed by post-merge outcomes (S6).

#### P11. No way to pick the best draft from N model attempts
- **Signal strength:** 1 voice — surfaced as a working practice, not as an articulated pain.
- **Voices:** Tiago (*"I tell it to do the change 20 times… The first thing I actually asked the model to create a retrospective… and then I tell it to like pick the one with the most ROI"*).
- **Why it matters:** Tiago has hand-built a workaround because single-shot generation hides quality variance — a high-quality draft can be lost among mediocre ones. Without a productized version (S13), this pattern stays trapped in one engineer's workflow. **New entry — added so S13 stops being an orphan solution.**

---

### 6B. Design Doctrine

Architectural commitments the workstream makes about *how* it builds. Not problems, not work items — doctrine adopted on the basis of customer evidence. Every solution in §7 must respect them.

#### D1. Discrete per-check evals over a single mega-prompt
- **Evidence:** David (*"singular evaluation of the entire code block against all of those facets poisons the context… very discrete evals for each of the facets… you get better results"* — David's word *"facets"* is what this doc now calls **checks**), Jyothi (three separate validations today), Shreyansh (separate Architecture Explorer step), Taylor (separate agent personas).
- **Commitment:** Every evaluation surface is built as a narrow **check** — one specific thing being evaluated (e.g. intent, security, tech-debt) — with its own prompt, schema, and trust track. S2 is the embodiment. Customer-felt accuracy effects of violating this doctrine surface as P2.

#### D2. Iterative loop over single-pass validation
- **Evidence:** Jyothi (Cycle-3 loop), Shreyansh (*"it does take multiple iterations… would basically reach close to 100[%] more often"*), David (state-machine loop), Taylor (*"if you give it another couple times to look at it, it would catch the error"* — for probabilistic drift specifically).
- **Commitment:** Validators self-correct before commenting; max-N iterations (Jyothi's 2–3 is the starting point). Constrained by P3: loops do *not* fix capability gaps, so loop output must be gated by domain-risk classification when those domains are in scope.

#### D3. Deterministic work runs outside the LLM
- **Evidence:** David (*"if you take a deterministic task and you stuff it through an LLM, you inherently make it non-deterministic… you reduce trust… you're actually burning a lot of money unnecessarily"*).
- **Commitment:** Build / test / push / comment / gate steps are deterministic and live outside the model. The LLM is invoked only for genuinely non-deterministic work (writing code, evaluating intent, summarizing). S3 is the reference orchestrator; S7 (artifact grounding) is the deterministic counterpart that feeds the LLM grounded facts instead of asking it to hallucinate them.

---

### 6C. Workstream & Ecosystem Concerns

Coordination and measurement items that sit outside the product surface. Important to track, but not addressed by user-facing features.

#### W1. Multiple teams are building parallel validation systems
- **Voices:** Jyothi (wants intent validator as a Copilot Code Review sub-agent), Shreyansh (platform PR assistant is *"shallow"* — so they built their own), David (wants Talos to plug into PR Lifecycle).
- **Why it matters:** Without a shared integration surface, every team rebuilds. The workstream's coordination opportunity is to provide that surface (S4 is the concrete proposal) and to align ongoing efforts (June joint demo with David).

#### W2. PR-velocity metrics don't measure AI impact and are gameable
- **Voices:** David (*"they're interesting, but they're also extremely gameable… it doesn't tell me how much AI is being used"*).
- **Why it matters:** Without honest measurement we cannot prove the workstream is helping — and we cannot set credible promotion thresholds for S8's trust stages. Addressed by S12.

---

## 7. POTENTIAL SOLUTIONS

### Priority order

Areas were ranked by (1) foundational dependency, (2) customer-coverage breadth, (3) presence of a named partner team ready to integrate this cycle, (4) whether the workstream can make any non-advisory claim without it.

| Rank | Area | One-line scope | Why ranked here |
|---|---|---|---|
| **#1** | **SR1 — PR Validation Agents & Skills** | A local-first set of agents, skills, and assets that resolves intent → runs a battery of narrow per-check evals in a self-correcting loop → emits a structured verdict. Runs in any host (local CLI, VS Code agent mode, future service wrappers). | Foundation: every other area composes against this. 5/5 customer coverage of the underlying need. PR Lifecycle Cycle 3 is a co-design partner, not a hosted-service dependency. |
| **#2** | **SR2 — Eval Trust Stages & Feedback Loop** | A small system that wraps each SR1 eval with (a) a *trust stage* — surface-only at first, allowed to take consequential action (halt a local loop, refuse a push, block a service-side merge) only after it has agreed with human reviewers often enough on a labelled dataset — and (b) a feedback loop that records every human override as a new labelled example the eval can be re-tuned against. Also emits the agreement-rate and impact metrics the promotion decision uses. | Without it, SR1 is permanently stuck in surface-only mode. The only path past David's *"60% maybe"* accuracy ceiling. Required before the workstream can claim any eval is good enough to gate any change (local push, service-side merge, or otherwise). |
| **#3** | **SR3 — Code Context Grounding Service** | Deterministic, build-artifact-driven facts (call graphs, symbol tables, ship-vs-source) that SR1's checks query instead of hallucinating. Includes a one-time legacy bootstrap. | Determines whether SR1 is useful at Microsoft scale (monorepo / legacy). Higher build cost — can trail SR1/SR2 in time, but critical for scope of relevance. |
| **#4** | **SR4 — Spec & Knowledge Lifecycle** | Addresses the *upstream cause* of validation failure: Socratic elicitation *before* code; PR Learn retrospective *after* merge. | Highest long-term leverage (closes P1 at the source), but depends on SR1+SR2 being deployed to generate the post-merge signal it retrospects on. |
| **#5** | **SR5 — Adjacent Generation Patterns** | Engineer-side patterns that improve the *quality of drafts* submitted to SR1, independent of SR1 itself. | Single-customer signal (Tiago). Useful and ready to productize, but off the critical path; could be absorbed into a partner skill-pack rather than a dedicated product. |

---

> **What counts as a "PR" in §7?** Throughout the rest of this section, *"PR"* is shorthand for a **candidate changeset** — a working branch plus a statement of intent (a linked spec, issue, or work item; the chat/agent session that produced the change; in-progress commit messages; a draft PR description). The changeset exists from the moment a developer (or an autonomous agent) starts editing the branch, **long before any PR is created in GitHub or ADO**, and the same validation logic applies to it across that entire lifecycle.
>
> What changes from local edit → pushed branch → draft PR → open PR is **(a) the host surfacing the verdict** (terminal, VS Code diagnostics, agent chat, PR-comment thread) and **(b) the consequence the runtime is authorized to take when a verdict fails** (print a warning, pause an autonomous loop, refuse to push, block a merge). The validation itself — intent resolution, the per-check evals, the self-correcting loop, the structured verdict — is identical at every step. SR1's S4 (host-neutral adapters) makes the surfacing host-agnostic; SR2's trust stages make the authorized consequence host-agnostic.
>
> §6 (problem statements) and §§3–5 (customer evidence) deliberately retain the customer-voice usage of *"PR"* — that is what customers said, and is preserved verbatim.

### SR1. PR Validation Agents & Skills

**Scope.** A **local-first** set of **agents, skills, and assets** that, for a given **candidate changeset** (a working branch + its intent statement, whether or not a service-side PR has been created yet in GitHub/ADO), resolves what the change *meant to do*, runs a battery of narrow **per-check evals** (one eval per thing being checked — intent, test coverage, security, tech-debt, etc.) in a self-correcting loop, and emits a **structured verdict**. Ships as plain building blocks — runnable from a local CLI, from VS Code agent mode, or wrapped by any host (Talos, a future Copilot Code Review sub-agent, a hosted PR Lifecycle plugin, an ADO task, a GitHub Action). The verdict schema and the agent/skill contracts are the API; the hosts are interchangeable.

**Primary first-cycle target:** developer inner loop (local CLI + VS Code agent mode) on the **changeset-in-progress** — well before a service-side PR exists. **Non-goals (first cycle):** Copilot Code Review sub-agent, hosted PR Lifecycle plugin, or any other service-hosted distribution. Those are *future adoption surfaces* — S4 captures the host-neutral design that keeps them cheap to add later.

**Boundary.** Does NOT improve its own accuracy (→ SR2), provide app-level grounding (→ SR3), author or maintain the artifacts it reads (→ SR4), or operate before the changeset is ready for validation (→ SR5). Does NOT take a hard dependency on any specific hosting service or on a service-side PR existing.

**Customer problems addressed.** P1, P4, P5, P6, P11. Workstream concern W1 (via S4's host-neutral adapter design).

**Components (existing S1, S2, S3, S4, S9, S11):**

#### S1. Intent-Source Resolver (→ P1, P7, P9)
A small upstream component that, given a candidate changeset, identifies the *most reliable available statement of intent* in a fallback chain (highest-confidence available source wins):

1. Linked spec doc (if attached to the work item or referenced in the changeset)
2. Linked work item / issue (Jyothi's fallback)
3. **Changeset-attached intent surface** — whichever exists, in order: a service-side PR title + description (when a PR has been opened); otherwise the branch's in-progress commit messages, a draft PR description, or the chat/agent session that produced the change (e.g., the VS Code agent transcript, a Talos work-item approval)
4. Inferred intent from architecture-explorer-style call graph (Shreyansh)
5. Inferred intent from existing tests (Taylor's *tests-as-invariants*)
6. **Trigger Socratic elicitation** (Taylor) if none of the above is sufficient

The Resolver returns both *what we think the intent is* and a *confidence* — downstream evals adjust strictness accordingly. The same resolver runs whether the changeset is a local branch with no PR yet or a fully-formed service-side PR; only the contents of source #3 change.

#### S2. Discrete Multi-Check Eval Framework (→ P4, P5; embodies D1)
A pluggable framework where each **check** is a narrow eval with its own prompt, schema, and trust track. A **check** is one specific thing being evaluated about the change. Initial checks:

- **Intent** vs. resolved intent source (S1)
- **Test coverage** of the changed surface (already shipping in PR Lifecycle)
- **Scenario-rule conformance** (Jyothi's #3 — generalize her CodeQL pattern)
- **Security / safety** (David has this shipping)
- **Tech-debt** (David's missing check — duplicate-function detection, complexity deltas, etc.)
- **Anti-pattern tests** (David — e.g., "delete all source, do tests still pass?")
- **Style / preferences** per area/owner (Tiago's pain)

Each check emits to a **standardized verdict schema** — roughly `{check, status, evidence, confidence, suggested_action}` (where `check` names which check produced this verdict — `"intent"`, `"security"`, `"tech_debt"`, etc.). The schema is the contract; **rendering is delegated to whichever host adapter (S4) is in use** — terminal output for the CLI, info/warning diagnostics or agent-chat replies in VS Code, a PR comment server-side. One schema, many surfaces. (S14 reuses the same acknowledgement hook on each surface to capture human override signal.)

> *Note (June 1 audit): previous mapping `→ P2, P5` corrected to `→ P4, P5`. P2 (accuracy ceiling) is owned by SR2 (S8+S14); S2 produces the verdicts that have the P2 problem but does not solve it. P4 (tests as the de-facto intent contract) is what the test-coverage check actually owns. See §9.5.*

#### S3. Loop Orchestrator with Deterministic-vs-Non-Deterministic Split (implements D2 + D3; supports P2 indirectly via self-correction)
Adopt David's Talos architecture as the reference loop:

- Deterministic steps (build, run tests, push PR, comment, gate evaluation) run *outside* the LLM.
- LLM is invoked only for genuinely non-deterministic steps (write code, evaluate intent, summarize).
- Failed deterministic step → feed back into LLM as context → re-loop, max-N iterations (Jyothi's 2–3).
- Each loop iteration logged for traceability (and for S6 below).

**Partnership opportunity:** explore reusing Talos directly. June joint demo already proposed by David.

#### S4. Host-Neutral Adapters (→ W1, P1)
Ship SR1 as a **host-neutral core** (agents, skills, verdict schema, and invocation contracts), with **thin adapters** exposing it through whichever host a given team uses. No business logic lives in any adapter — every adapter is a translation layer between a host's invocation/rendering conventions and SR1's contracts.

**First-cycle adapters (built):**

- **Local CLI** — primary target. Runs against a local diff or a fetched PR. Inner-loop usable on a developer machine, no service dependency.
- **VS Code agent mode / chat agent** — invoked from the editor against the current branch or open PR. Same core, different host.

**Future adapters (designed-for, not built first cycle):**

- **Copilot Code Review sub-agent** — wraps the same core so any human PR can be validated server-side. Jyothi / PR Lifecycle Cycle 3 is the co-design partner that keeps this wrapping cheap. *Co-design only; not a first-cycle deliverable.*
- **Hosted PR Lifecycle plugin** — generalizes Jyothi's auto-PR pipeline beyond CodeQL.
- **ADO PR-comment task / GitHub Action** — drop-in for teams running their own pipelines.
- **Talos orchestrator step** — David's flow orchestrator calls SR1 as a non-deterministic step. Reinforces SR1/SR2/Talos as the cross-org reference architecture.

The shared surface that closes W1 is not a single hosting service — it's the **host-neutral contract** that any of the above can wrap. Picking one host as the canonical path before the building blocks exist would force the others to wait or fork.

#### S9. Style/Preferences-as-Rules Layer (→ P6)
A per-area / per-owner rules file that captures subjective reviewer preferences (e.g., naming conventions, abstraction depth, error-handling style). Loaded into the LLM context when modifying that area. Addresses Tiago's "tune the workflow per reviewer" pain without requiring a full spec.

Could be auto-populated by S6 (PR Learn) over time.

#### S11. Anti-Pattern Test Suite (→ P4, P5)
A reusable, language-agnostic library of anti-pattern probes:

- *"Delete all source — do tests still pass?"*
- *"Same function signature appears N times — flag?"*
- *"Cyclomatic complexity delta exceeds baseline — flag?"*
- *"Test asserts on its own setup — flag?"*

Runs in the deterministic side of S3 and feeds S2's tech-debt check.

---

### SR2. Eval Trust Stages & Feedback Loop

> **Plain-English definition.** SR1 produces eval verdicts ("this change looks fine", "this change is missing tests", etc.) against a **candidate changeset** — which may be a local branch a developer (or an autonomous agent) is still iterating on, or a service-side PR already open in GitHub/ADO. Those verdicts are not 100% accurate — David put his current eval accuracy at *"60% maybe."* SR2 is the small system that sits **around** those verdicts and answers two questions every adopter immediately asks: *"What happens to my work when the eval is wrong?"* and *"How does the eval ever get less wrong?"*
>
> It answers the first by **starting every eval in surface-only mode** (verdicts are shown, never block progress) and only allowing it to graduate to consequential actions — halting an autonomous loop, refusing a push, blocking a service-side merge — after it has agreed with human judgment often enough on a labelled dataset. It answers the second by **recording every human override as a new labelled example**, so the eval can be re-tuned against its own real-world mistakes.
>
> Same wrapper works around any verdict producer, not just SR1 — David's Talos evals can plug in too.

**Scope.** Three small, cooperating pieces, deployed together: a per-eval **trust-stage policy** that the SR1 runtime reads before deciding how loudly to act on a verdict (S8); a **feedback-capture-and-promotion job** that turns human overrides into labelled data and decides when an eval has earned the next stage (S14); and a **metrics emitter** that produces the agreement-rate and impact numbers the promotion job depends on (S12).

**Boundary.** Does NOT produce eval verdicts (→ SR1) or ground them in code reality (→ SR3). Designed to wrap *any* validator — SR1's per-check evals, David's Talos evals, a partner team's custom check. **Implementation form, like SR3: a small library + a per-eval config file + a scheduled CLI/MCP job, not a hosted service** (consistent with the §7 local-first commitment).

**Customer problems addressed.** P2 (closes the accuracy ceiling that S8 alone only manages around). Workstream concern W2 (honest telemetry). Indirectly supports W1 by giving partner teams a shared adoption framework.

#### What a developer actually builds (high level)

A concrete first cut of SR2 is roughly the following four artifacts:

1. **A per-eval policy file (S8).** A YAML/JSON config — one entry per eval — that declares the eval's current trust stage (`advisory` | `shadow` | `soft_gate` | `hard_gate`) and the numeric thresholds required to advance to the next stage. The SR1 runtime reads this file before surfacing the verdict; the **authorized consequence** that the stage permits (surface only, pause for confirmation, refuse to advance) is then translated by the active host adapter (S4) into the concrete action available on that surface — a terminal warning, an IDE diagnostic, *"agent halts before pushing the branch"*, or *"PR merge button is blocked"*. New evals default to `advisory`.
2. **A feedback-capture path (S14).** The host adapter (S4) that renders the verdict makes it explicitly acknowledgeable on its surface (CLI thumbs-up/`--ack`, VS Code inline accept/reject or chat reply, a reaction or reviewer override on a service-side PR comment). Each acknowledgement is appended — together with the eval's input and verdict — to a per-eval **calibration set**: a small versioned labelled dataset, one file per eval. David's existing ~100 labelled intent samples are the starter dataset for the intent eval.
3. **A promotion-checker job (S14).** A scheduled CLI/MCP job that, for each eval, re-runs it against its calibration set, computes the eval's agreement rate with humans, and — if the rate clears the threshold declared in the policy file for the next stage — bumps the eval's stage. The check, the rate, and the promotion decision are emitted as events via S12.
4. **A metrics emitter (S12).** A thin library that the SR1 runtime and the promotion checker both call. Emits structured events for every verdict, every override, every promotion check, and the four AI-impact counters (LLM PRs merged without rework, cycle time before vs. after AI, agreement rate per eval, defect-escape rate per eval). Powers dashboards *and* the promotion decision.

That is the entire mechanism. No model training, no hosted service, no PR-platform integration — just config, a dataset, a periodic job, and an emitter library.

#### What this lets the workstream ship that it cannot ship without SR2

- A defensible answer to *"what does adopting your eval cost me?"* — adoption starts at `advisory`, where the worst case is an extra PR comment.
- A concrete promotion path past comment-only — answering David's *"how does it ever get out of advisory?"* objection with a number, a dataset, and a job, not a promise.
- A measured, per-eval *"how often does this agree with humans?"* number — replacing anecdote with a metric that the same SR1 building blocks emit everywhere they run.

#### S8. Trust-Staged Eval Deployment Policy (→ P2 — manages around the accuracy ceiling; does not close it)

The per-eval policy file. The SR1 runtime reads it before acting on a verdict. The stage declares the **authorized consequence** (what the runtime is permitted to do when the verdict fails); each host adapter (S4) translates that into the concrete action available on that surface. **Same stage, same authorization, different mechanism per host** — which is why one policy file works across local CLI, VS Code agent mode, and any service-side wrapper.

| Stage | Authorized consequence on failure | Local CLI / VS Code agent mode (changeset-in-progress) | Service-side PR (CCR / ADO task / hosted plugin) | Promotion criteria (checked by S14's job) |
|---|---|---|---|---|
| `advisory` | Surface the verdict; never block progress. | Print to terminal; show as info-level diagnostic in VS Code; agent reports it in chat. | Post a non-blocking PR comment. | Eval has run against ≥X changesets (baseline collected). |
| `shadow` | Run the eval but do **not** surface to the human; capture for offline analysis. | Log verdict + inputs to the calibration set; no terminal/IDE output. | Capture only; no PR comment posted. | Agreement rate with humans on the calibration set ≥ Y%. |
| `soft_gate` | On failure, **hold for human confirmation** before progress continues. | Autonomous loop (Talos-style) pauses for human reply; VS Code agent asks *"override and continue?"* before pushing or completing the task. | Post a "blocking suggestion" comment; merge button shows a warning but reviewer can override. | Agreement rate ≥ Z%, validated across ≥N changesets. |
| `hard_gate` | On failure, **refuse to advance**. | Autonomous loop halts; agent refuses to push the branch; CLI exits non-zero. | Merge is blocked until the eval passes (or an administrative override is granted). | Sustained agreement rate ≥ Z%, no critical false-positive class in the last M changesets. |

Directly addresses David's *"I don't want it actually blocking or allowing work to go through until I have a high degree of trust"* objection by **bounding the consequences** when an eval is still inaccurate — a wrong verdict in `advisory` mode is, at worst, a note the human can ignore (a terminal print, an IDE info diagnostic, or an unblocking PR comment). Does **not** improve accuracy on its own — that is S14's job. Together S8 + S14 give the workstream a defensible adoption story: ship behind the advisory rail on the **changeset-in-progress**, learn from disagreements (including pre-service ones), and promote toward consequential gating only when calibrated.

#### S14. Eval Calibration Loop (→ P2)

Closes the loop that S8 only *manages around*. Four concrete pieces a developer implements:

- **Disagreement capture.** Whichever host adapter (S4) renders the verdict makes it explicitly acknowledgeable, using whatever signal the surface supports: a thumbs-up/-down prompt or `--ack` flag in the CLI; inline accept/reject on the diagnostic, or a reply in agent chat, in VS Code; a reaction or a reviewer override on the gate, in a service-side PR comment. The acknowledgement signal (and any free-text reason), the eval's input, and the eval's verdict are appended to the per-eval calibration set. **Capture works pre-service as well as post-service** — local-loop overrides count just as much as PR-comment overrides.
- **Per-eval calibration set.** A versioned, labelled dataset — one file per eval (one per check). David's ~100 labelled intent samples are the starter dataset for the intent eval. New rows appear every time a reviewer disagrees with a verdict.
- **Improvement actions.** When disagreement clusters appear, trigger rubric refinement, few-shot updates, prompt-structure changes, and contributions to S11's anti-pattern library. (These are eval-team actions, not runtime behavior — but the calibration set is what makes them targeted instead of guesswork.)
- **Promotion checker.** A scheduled job that re-runs each eval against its calibration set, computes agreement rate, and — when the rate crosses S8's threshold for the next stage — advances the eval's stage in the policy file. The promotion event is emitted via S12.

**Without S14, S8 can shelter low-trust evals forever without ever earning trust.** With S14, the workstream has an explicit mechanism to attack David's *"60% maybe"* number — and a credible answer to *"how do you ever get out of advisory mode?"*.

#### S12. Honest AI-Impact Telemetry (→ W2; also powers S8 + S14 promotion thresholds)

A thin emitter library called by the SR1 runtime and by S14's promotion checker. Emits structured events that feed dashboards *and* the promotion check. Replaces gameable PR-velocity metrics with:

- Number of LLM-driven PRs reaching merge without rework
- Cycle time *after* AI vs. *before*
- Agreement rate with humans, per eval (this is the number S14's promotion checker compares against S8's thresholds)
- Defect-escape rate per eval

Reuses David's maturity-model framing.

---

### SR3. Code Context Grounding Service

**Scope.** A deterministic, build-artifact-driven knowledge base that SR1's checks query as a tool — call graphs, symbol tables, dependency graphs, ship-vs-source distinctions — replacing hallucination-prone code search with grounded fact retrieval. Includes a one-time bootstrap flow for legacy apps.

**Boundary.** Purely deterministic; no LLM in the path. Validation logic lives in SR1; SR3 only answers factual questions about the shipped artifact. **Implementation form: a local library, per-repo CLI, or MCP server invoked from the dev machine — not a hosted cloud service** (consistent with the §7 local-first commitment). The word "Service" in the area name is used in the SOA sense (*provides a service to the LLM*), not the hosted-cloud sense.

**Customer problems addressed.** P8, P9, P5. Partially P1 (powers the call-graph fallback in S1's resolver chain).

**Components (existing S7, S10):**

#### S7. Architecture/Artifact Grounding Service (→ P8, P9, P5)
Productize Shreyansh's Architecture Explorer concept as a generic capability:

- Build-time artifact ingestion (call graphs, symbol tables, dependency graphs, ship-vs-source distinctions).
- Deterministic API the LLM can query: *"who calls X?", "is Y in the shipped binary?", "what depends on Z?"*
- Replaces hallucination-prone code-search with grounded fact retrieval.

Critical for monorepo / large-codebase teams (Tiago, Shreyansh).

#### S10. Legacy-Mode Docs Bootstrapping (→ P8, P1)
A starter-pack flow that, for a legacy app, runs a **one-time dedicated bootstrap** (Shreyansh's *"dedicated effort for a week or two"*) using S7's grounding service to:

- Generate root-level docs from build artifacts + code structure.
- Surface high-coverage docs first (most-called modules) and leave low-coverage areas explicitly marked as *"unknown — Socratic elicitation needed"*.
- After bootstrap, S6 keeps it in sync.

Answers Shreyansh's strategic question: *"is the AI-first starter pack a fit for legacy codebases?"* Without S10 the answer is "no" — with S10 the answer is "yes, with a bootstrap step."

---

### SR4. Spec & Knowledge Lifecycle

**Scope.** The upstream and downstream work that creates and maintains the artifacts SR1 reads. Two stages: Socratic elicitation *before* code is written (extract tacit knowledge into spec), and PR Learn retrospective *after* merge (update docs, skills, and the eval's known-bad-pattern library based on what actually happened).

**Boundary.** Does NOT run on the changeset itself. Runs **before the changeset exists** (S5 elicits intent from the human before code is written) and **after merge** (S6 retrospects on what actually happened). Owns the artifacts; SR1 only consumes them.

**Customer problems addressed.** P1 (closes the upstream cause), P3 (shift-left mitigation), P7, P9, P10.

**Components (existing S5, S6):**

#### S5. Socratic Pre-Code Elicitation Phase (→ P1, P9, P3)
Before writing code (or before running the intent eval if code already exists), have the model:

- Build an **open-question list** based on the spec/work-item.
- Ask the human pointed questions designed to expose tacit knowledge.
- Capture the answers as a structured intent-augmentation file that becomes part of the spec.

This is Taylor's strongest recommendation and is the *cheapest way to shift validation left*.

#### S6. PR Learn — Post-Merge Retrospective Loop (→ P7, P10)
Borrow ODSP web photos' pattern (per Shreyansh):

- Trigger after every PR merge.
- Compare PR outcome vs. plan: what mistakes were made? What rework was needed? What docs were out of date?
- Update: (a) the affected docs/specs, (b) the team's skills/agents/prompts, (c) the eval's known-bad-pattern library (which feeds S2's anti-pattern check).

Pairs naturally with S3's traceability logs.

---

### SR5. Adjacent Generation Patterns

**Scope.** Engineer-side patterns that improve the *quality of the draft* submitted to SR1, independent of SR1 itself. Sits **before validation begins** on the changeset (whether or not the changeset later becomes a service-side PR).

**Boundary.** Complementary, not foundational. Could be productized as a dedicated capability or absorbed into a partner skill-pack — explicitly off the critical path.

**Customer problems addressed.** P11.

**Components (existing S13):**

#### S13. Multi-Shot + Retrospective + ROI Selector (→ P11)
Productize Tiago's working practice as a starter-pack capability: ask the model to do the change N ways, ask it to write a retrospective on each, ask it to pick the best by ROI. Useful for repetitive mechanical changes where one good draft hides among several mediocre ones.

---

## 8. Open Questions for Follow-Up Discovery

1. **Spec authority:** when work items, PR descriptions, and existing docs disagree, which wins? (raised by Jyothi)
2. **Style-rule capture:** is it realistic to auto-mine reviewer preferences from past comments (Tiago)? Or do we need explicit owner-authored rules?
3. **Test-as-spec dependency:** for teams that don't have Taylor's 80%+ coverage, is intent-eval-via-tests still viable, or does it collapse?
4. **Legacy starter-pack fit:** Shreyansh's open question deserves a direct answer in product positioning — is this for AI-native rewrites only, or do we commit to legacy?
5. **Trust calibration baselines:** what's the human-team-of-5 agreement rate that should anchor S8's promotion thresholds? (David proposed the framing; we need numbers.)
6. **PR Learn ownership:** post-merge retrospective lives between PR Lifecycle, the team's coding agent, and the docs system. Who owns it operationally?

---

## 9. Appendix

### 9.1 Source Recordings

| # | Date | Interviewee | Recording (SharePoint) |
|---|---|---|---|
| 1 | May 18, 2026 | Jyothi Marreddy (+ Rupam Mittal) | *Sync on PR Lifecycle and PR Intent Validation* |
| 2 | May 20, 2026 | Tiago Macarios | *Interview PR Intent Verifications … (Tiago Macarios)* |
| 3 | May 21, 2026 | Shreyansh Agrawal | *Interview PR Intent Verifications … (Shreyansh Agrawal)* |
| 4 | May 21, 2026 | David Coulter | *Interview PR Intent Verifications … (David Coulter)* |
| 5 | May 21, 2026 | Taylor Williams | *Interview PR Intent Verifications … (Taylor Williams)* |

All recordings are stored at `https://microsoft-my.sharepoint.com/personal/cesardl_microsoft_com/Documents/Recordings/`.

### 9.2 Glossary of Customer-Introduced Terms

| Term | Source | Meaning |
|---|---|---|
| Intent validation | Jyothi | Check that a PR's actual changes match the issue/spec it claims to fix. |
| Scenario-based validation | Jyothi | Per-scenario guideline conformance (e.g., all CodeQL-fix PRs follow a set of rules). |
| Architecture Explorer | Shreyansh | Deterministic, build-artifact-driven call-graph tool that grounds the agent in shipped reality. |
| PR Learn | Shreyansh (origin: ODSP web photos team) | Post-merge retrospective pipeline that updates docs, skills, agents from each PR's outcomes. |
| Talos | David | State-machine flow orchestrator for autonomous dev; separates deterministic from non-deterministic steps. |
| Smartly building tools | David | Strategic frame: AI value is in *how* we build, not in the smartness of the artifact built. |
| Agent swarm / personas | Taylor | Multiple model instances with different roles reviewing the same PR before humans. |
| Socratic phase | Taylor | Model interrogates the human up-front with pointed questions to extract tacit knowledge. |
| Tacit-knowledge gap vs. capabilities gap | Taylor | Two distinct *causes* of model desynchronization. Per Taylor: tacit knowledge is addressed by tests (which crystallize invariants) and/or a Socratic elicitation phase; capabilities gaps are not loop-fixable and require humans-in-the-loop. (Orthogonal axis: drift = probabilistic mistakes, *which* loops can fix.) |
| Tests-as-invariants | Taylor | Rigorous fuzz/end-to-end test suites encode invariants that the model can't violate without failing tests. |
| Anti-pattern testing | David | Test for what *should not* exist (e.g., passing tests when source is deleted). |
| 20-variant batch | Tiago | Ask the model to produce N variants of a mechanical change, write retrospectives, pick by ROI. |

### 9.3 Verification & Provenance

This document was generated from transcripts retrieved via WorkIQ (M365 Copilot) and was subsequently fact-check audited against the **full, unabridged customer-only verbatim transcripts** (every utterance the customer made, in chronological order, with the interviewer's lines stripped). The audit checked each quote and each attributed claim. Findings and corrections applied:

| Severity | Finding | Resolution |
|---|---|---|
| 🔴 Critical | Summary-table Taylor row mapped tacit-knowledge to loops; loops actually fix probabilistic drift per Taylor, while tests/Socratic phase address tacit knowledge | Rewrote the row to follow Taylor's actual two-axis framing (cause: tacit-knowledge vs. capabilities; failure type: drift vs. jagged-intelligence) |
| 🔴 Critical | Glossary said "only the first [tacit-knowledge] is loop-fixable" | Corrected to reflect Taylor's actual model |
| 🟡 Moderate | Exec summary generalized David's "~60%" to all evals | Scoped to **intent** eval specifically and attributed |
| 🟡 Moderate | Exec summary claimed PR Lifecycle + Talos + ODSP all converged on multi-check eval architecture | Narrowed to PR Lifecycle + Talos; ODSP achieves grounding via a separate tool (Architecture Explorer) rather than per-check evals |
| 🟡 Moderate | Cross-cutting matrix showed Tiago ✓ and Taylor ✓ for "specs/docs lag" | Demoted both to ~ with explanatory text; Tiago's quote is about per-area AI guidance not PR specs; Taylor's is about no post-coding sync phase |
| 🟡 Moderate | Cross-cutting matrix showed Shreyansh ✓ for "discrete evals per check" | Demoted to ~ with note: ODSP achieves it architecturally via separate deterministic tool, not as labeled evals |
| 🟡 Moderate | P1 ("Intent against what?") cited Tiago's "years" quote, which is about a different problem | Replaced Tiago with David's Gherkin/intent quote; kept Tiago as an "adjacent voice" |
| 🟢 Minor | "Tried to write a C parser unprompted" — Tiago didn't say "unprompted"; the model went off-track while solving the assigned problem | Reworded in three places |
| 🟢 Minor | Shreyansh's "Humans are bad to follow instructions to keep docs up to date" was a paraphrase presented as verbatim | Replaced with full verbatim quote (with [docs] correction for transcription artifact "dogs") |

**Cleanups preserved as acceptable (not flagged as issues):**

- Verbatim quotes were lightly cleaned of disfluencies ("yeah", "so", "I mean", "right?") and transcription errors ("peer"→"PR", "dogs"→"docs", "the can create"→"can create", "two followings"→"these followings"). All such cleanups preserve meaning.
- Ellipses are used to mark compressed quotes spanning multiple consecutive customer utterances.
- Editorial brackets `[...]` are used where context was added for readability (e.g., `[iterative loop]`, `[docs]`, `[for PR review]`).
- David's "June joint demo" reference is sourced from `customer-interviews.md` (Cesar's interview notes) rather than from David's recorded words; David's recorded acknowledgments ("Sure", "Oh yeah", "Great, sounds good") are consistent with the agreement noted in those notes.

**Full transcripts** retrieved via WorkIQ are available on demand from the SharePoint recording locations listed in §9.1.

### 9.4 Second-Round Structural Audit (June 1, 2026)

A second-pass review of §6 found that several entries were not customer-felt problems but architectural lessons or workstream coordination items, producing misleading 1:1 problem-to-solution mappings (a "solution" that simply restates a design principle is not a solution). The section was restructured into §6A (Customer Problems), §6B (Design Doctrine), §6C (Workstream & Ecosystem Concerns); three entries were reframed from constraints/symptoms to solvable customer pains; one new problem (P11) was added to give the previously orphaned S13 a home; and one new solution (S14, Eval Calibration Loop) was added to actually close the accuracy gap that S8 only manages around.

**Renumbering map (previous draft → current):**

| Previous | Current | Change |
|---|---|---|
| P1 ("Intent against what?") | P1 | unchanged |
| P2 (trust gap, ~60% accuracy) | P2 | **reframed** — symptom (trust) split from cause (accuracy ceiling); now paired with new S14 |
| P3 (mega-prompt poisons context) | **D1** | moved to doctrine (design lesson, not customer pain) |
| P4 (single-pass insufficient) | **D2** | moved to doctrine (design lesson, not customer pain) |
| P5 (capability gaps loops can't fix) | P3 | **reframed** — un-solvable constraint → solvable failure mode ("ships undetected because loop reports success") |
| P6 (tests can be wrong) | P4 | unchanged |
| P7 (tech-debt blindness) | P5 | unchanged |
| P8 (subjective reviewer preferences) | P6 | unchanged |
| P9 (docs lag code) | P7 | unchanged |
| P10 (monorepo / legacy blockers) | P8 | **reframed** — environmental constraint → solvable context-starvation problem; also absorbs §5 matrix theme "need a deterministic grounding tool" which had no §6 entry |
| P11 (tacit knowledge gap) | P9 | unchanged (clarified distinction from P1) |
| P12 (no platform integration target) | **W1** | moved to workstream concerns (ecosystem coordination, not a feature gap) |
| P13 (velocity metrics gameable) | **W2** | moved to workstream concerns (measurement, not a feature gap) |
| P14 (cost of LLM for deterministic work) | **D3** | moved to doctrine (design lesson, not customer pain) |
| P15 (skills / agents drift over time) | P10 | unchanged |
| (new) | **P11** | added — "no way to pick best draft from N attempts" (Tiago's pattern); gives S13 a home |

**Solution updates:**

| Solution | Change |
|---|---|
| S2 | Mappings updated to P2, P5; now described as *embodying* D1 |
| S3 | Repositioned as the implementation of D2 + D3 rather than addressing P-items directly |
| S8 | Renamed *Trust-Staged Eval Adoption* → *Trust-Staged Eval Deployment Policy*; clarified it bounds consequences of P2 but does not close it |
| S12 | Noted that it powers S8 + S14 promotion thresholds |
| S13 | Mapped to new P11 (was previously orphaned as "Tiago's pattern") |
| **S14** | **New.** Eval Calibration Loop — captures human-vs-eval disagreements as labelled data, feeds a per-check improvement pipeline, drives S8 promotion. Without it, S8 is a holding pattern. |

**Why this matters:** the previous §6 conflated four categories of "thing" (customer problems, design lessons, environmental constraints, workstream concerns), which made the solution set look broader than it was — three of the original P entries were "solved" by solutions that simply restated them affirmatively (P3↔S2, P4↔S3, P14↔S3). The restructure removes those false mappings, surfaces one genuinely missing solution (S14), and gives one orphan solution (S13) a real problem to point at. No customer evidence was added, removed, or reinterpreted in this audit — only the framing changed.

### 9.5 Third-Round Structural Audit (June 1, 2026) — Solution Grouping Pass

A third-pass review of §7 found that the 14 sibling solutions (S1–S14) were not peers. They sat at different lifecycle moments (before / during / after the PR), had different owners, and several only worked as a co-deployed unit — S8 without S14 is a holding pattern; S14 without S12 has no promotion signal; S10 is a use case built on top of S7; S4 is the distribution surface for S1+S2+S3. The flat list invited "more solutions = more value" reasoning and gave partner teams no clean answer to *"what would I actually adopt?"*.

§7 was restructured into five **Solution Areas (SR1–SR5)**. Original S1–S14 bodies are preserved verbatim as sub-sections under their owning area, so all §6 cross-references stay valid. One solution mapping was corrected (S2). No customer evidence was added, removed, or reinterpreted — only the grouping, ordering, and one mapping changed.

**Grouping map (S → SR):**

| Solution Area | Rank | Contains | Boundary |
|---|---|---|---|
| **SR1 — PR Validation Agents & Skills** | #1 | S1, S2, S3, S4, S9, S11 | The on-PR runtime. Does not improve accuracy, ground itself, or maintain its inputs. *(Rename lineage: "PR Validation Sub-Agent" → "PR Validation Toolkit" in the fourth-round audit (§9.6) → "PR Validation Agents & Skills" in the eighth-round audit (§9.10).)* |
| **SR2 — Eval Trust Stages & Feedback Loop** | #2 | S8, S14, S12 | Wraps any verdict producer (SR1 checks, Talos evals, partner checks) with a trust-stage policy + feedback loop + metrics. Does not produce verdicts. *(Rename lineage: "Trust & Calibration Platform" → "Eval Trust Ladder & Feedback Loop" in the fifth-round audit (§9.7) → "Eval Trust Stages & Feedback Loop" in the ninth-round audit (§9.11).)* |
| **SR3 — Code Context Grounding Service** | #3 | S7, S10 | Deterministic facts about the shipped artifact. No LLM in the path. |
| **SR4 — Spec & Knowledge Lifecycle** | #4 | S5, S6 | Runs before and after the PR — not on it. Maintains the artifacts SR1 reads. |
| **SR5 — Adjacent Generation Patterns** | #5 | S13 | Improves draft quality before validation begins on the changeset. Off the critical path. *(Boundary rephrased from "before the PR is opened" in the sixth-round audit — see §9.8.)* |

**Priority rationale.** Areas were ranked by (1) foundational dependency, (2) customer-coverage breadth, (3) presence of a named partner team ready to integrate this cycle, (4) whether the workstream can make any non-advisory claim without it. Result: SR1 → SR2 → SR3 → SR4 → SR5. One-line rationales in §7's *Priority order* table.

**Solution mapping correction:**

| Solution | Change | Why |
|---|---|---|
| **S2** | Heading mapping `→ P2, P5` → `→ P4, P5` | P2 (accuracy ceiling) is owned by SR2 (S8+S14), not by the per-check eval framework. S2 *produces* the verdicts that have the P2 problem; it doesn't *solve* it. Adding P4 reflects what the test-coverage check actually owns (tests as the de-facto intent contract). |

**Two judgment calls noted for the record.**

- **SR5 kept as a separate area** rather than dropped to a footnote. Single-customer signal (Tiago), but the pattern is real and ready to productize. Keeping it visible avoids losing the idea; flagging it as #5 / off-critical-path avoids overstating its importance.
- **SR2 ranked ahead of SR3** because every customer who would adopt SR1 hits the trust ceiling immediately, while only monorepo / legacy customers (ODSP, Office) are blocked on SR3. If weighting shifts toward monorepo coverage, SR2 ↔ SR3 ordering should be revisited.

**No other bodies were changed.** S1–S14 text, P1–P11 problem statements, D1–D3 doctrine, W1–W2 workstream concerns, §8 (Open Questions), and §9.1–§9.4 are unchanged. Only the framing, grouping, and S2 mapping in §7 changed.

### 9.6 Fourth-Round Structural Audit (June 1, 2026) — Local-First Reorientation

A fourth-pass review of §7 found that the previous draft framed SR1 as a *"PR Validation Sub-Agent"* with S4 (*"Sub-Agent for Copilot Code Review"*) presented as the canonical distribution surface. This baked a hard dependency on a specific hosted service into the workstream's first-cycle scope, before the building blocks themselves had been validated, and it understated the value of the same building blocks running in the developer inner loop.

The workstream commits to a **local-first, service-agnostic** posture for the first cycle: ship the toolkit as **plain assets — agents, skills, MCP servers, eval rubrics, CLIs** — that run on a developer machine or in any host that can invoke an agent. Service-hosted distribution becomes a *future adoption surface*, not a first-cycle deliverable.

**What changed in §7:**

| Item | Change |
|---|---|
| SR1 heading | Renamed *PR Validation Sub-Agent* → *PR Validation Toolkit* |
| SR1 scope | Reframed as a local-first set of agents/skills/assets emitting a *structured verdict* (host renders it). Primary first-cycle target = developer inner loop (local CLI + VS Code agent mode). Explicit non-goals added for service-hosted distribution. |
| SR1 boundary | Added: "Does NOT take a hard dependency on any specific hosting service." |
| SR1 priority-order row | Replaced "PR Lifecycle Cycle 3 is the live integration target" with "Cycle 3 is a co-design partner, not a hosted-service dependency." One-line scope rewritten in the same spirit. |
| **S4** | Rewritten in place from *Sub-Agent for Copilot Code Review on Human PRs* → *Host-Neutral Adapters*. First-cycle adapters = local CLI + VS Code agent mode. Future adapters (designed-for, not built first cycle) = CCR sub-agent, hosted PR Lifecycle plugin, ADO/GitHub task, Talos orchestrator step. Same S4 label, same `→ W1, P1` mapping. |
| SR3 boundary | One-line clarification added: implementation form is a local library / per-repo CLI / MCP server, not a hosted cloud service. Name retained ("Service" used in the SOA sense). |
| §7 preamble | Added a "Local-first commitment" blockquote documenting the non-goals stance and the co-design (not service-dependency) framing for Jyothi/PR Lifecycle and David/Talos. |

**What did NOT change:**

- S1–S14 numbering and bodies (S4 was rewritten in place under the same label and same problem mapping; all other S# bodies untouched).
- S → SR grouping (S4 stays in SR1; all other assignments unchanged).
- Priority order (SR1 → SR5; local-first reframe is orthogonal to dependency ranking).
- P1–P11, D1–D3, W1–W2 (no problem-statement edits).
- §6, §8, §9.1–§9.5 (untouched).
- The S2 mapping correction from §9.5 (still applies).
- SR2 and SR4 scopes (already service-agnostic by construction — SR2 explicitly "usable around any validator including Talos"; SR4 runs locally by definition).

**Why this matters.** Three customers want a shared integration surface (W1: Jyothi, Shreyansh, David), but they each name a *different* preferred host — CCR sub-agent, custom PR agent, Talos orchestrator step. Picking one host as the canonical distribution path before the building blocks exist would force the other two to wait or fork. Shipping the building blocks first — and the adapter contracts second — lets all three plug in on their own timelines, keeps the workstream's first-cycle scope honest, and preserves the option to add a CCR (or any other) wrapper later when the core has earned the trust to justify it (per SR2's trust stages).

### 9.7 Fifth-Round Structural Audit (June 1, 2026) — SR2 Clarity Rewrite

A fifth-pass review of §7 found that SR2's framing — *"Trust & Calibration Platform"*, *"trust envelope"*, *"bounds blast radius of low-trust verdicts"* — was too abstract and jargon-laden for a developer reader to know what to build. The underlying mechanism is concrete (a config file, a labelled dataset, a periodic job, and an emitter library), but the prose was hiding it behind SRE/security idioms.

**What changed in §7:**

| Item | Change |
|---|---|
| SR2 heading | Renamed *Trust & Calibration Platform* → *Eval Trust Ladder & Feedback Loop*. |
| SR2 priority-order row | Rewritten in plain language — dropped *"trust envelope"* and *"blast radius"*; describes the mechanism directly (each eval starts as a comment, can only graduate to blocking after agreeing with humans often enough on a labelled dataset; every human override becomes a new labelled example). |
| SR2 section opener | Added a **Plain-English definition** blockquote that names the two questions every adopter asks (*"what happens when the eval is wrong?"* and *"how does it get less wrong?"*) and answers them in one paragraph before any S# detail. |
| SR2 scope paragraph | Rewritten to describe the three cooperating pieces in concrete terms (per-eval policy file, feedback-capture-and-promotion job, metrics emitter) instead of abstract roles. |
| SR2 boundary | Added explicit implementation-form line, parallel to SR3: *"a small library + a per-eval config file + a scheduled CLI/MCP job, not a hosted service."* |
| New: *What a developer actually builds (high level)* | Added a numbered four-artifact list (policy file, feedback-capture path, promotion-checker job, metrics emitter) that names exactly what a developer would implement in a first cut, with the closing line: *"No model training, no hosted service, no PR-platform integration — just config, a dataset, a periodic job, and an emitter library."* |
| New: *What this lets the workstream ship that it cannot ship without SR2* | Added a three-bullet "capabilities unlocked" block to make the value concrete (adoption cost answer, promotion path past comment-only, measured per-eval agreement rate). |
| S8 table | Stages renamed to backtick-quoted identifiers (`advisory`, `shadow`, `soft_gate`, `hard_gate`) matching the policy-file values. Behavior column rewritten to describe what the runtime *does*, not abstract "behavior". Promotion-criteria column made concrete ("≥ Y% agreement on calibration set" instead of "> threshold"). |
| S8 prose | Replaced *"bounding the consequences of low accuracy"* with *"a wrong verdict in advisory mode is, at worst, a misleading PR comment"*. |
| S14 | Restructured into four explicitly labelled pieces (Disagreement capture, Per-eval calibration set, Improvement actions, Promotion checker), each described in implementation terms. |
| S12 | Reframed as *"a thin emitter library called by the SR1 runtime and by S14's promotion checker"* rather than a generic telemetry track. Same four metrics retained, with one annotation pointing out which metric S14's promotion checker compares against S8's thresholds. |

**What did NOT change:**

- S8, S14, S12 numbering and → P#/W# mappings.
- The S → SR grouping (S8 + S14 + S12 still constitute SR2; no membership change).
- Priority order or any other SR's content.
- The local-first commitment from §9.6 (SR2 was already service-agnostic; the new "implementation form" line just makes it explicit alongside SR3).
- P1–P11, D1–D3, W1–W2, §6, §8, §9.1–§9.6 (untouched).
- The customer evidence cited by SR2 (David's *"60% maybe"*, his ~100 labelled intent samples, his maturity-model framing) — same evidence, clearer prose around it.

**Why this matters.** The §6/§7 audience includes engineers on partner teams (David's Talos team, Jyothi's PR Lifecycle team, Shreyansh's ODSP team) who will judge whether to adopt SR2 by reading the section. If they cannot tell from the prose what they would have to build or integrate, they will pattern-match SR2 to *"another ML platform pitch"* and pass. The rewrite makes the mechanism unmistakable: it is a config file, a dataset, a job, and an emitter — small, local, and wrappable around their own evals — not a platform commitment.

### 9.8 Sixth-Round Structural Audit (June 1, 2026) — *"PR"* Reframed as Candidate Changeset

A sixth-pass review of §7 found that the solution prose conflated two different referents under the same word *"PR"*:

1. **The service-side artifact** — the record in GitHub/ADO with a number, a comment thread, and a merge button. Exists only after `git push` + PR creation.
2. **The candidate changeset** — a working branch plus the intent statement attached to it (issue, spec, work item, in-progress commit messages, chat/agent session), which exists from the moment the developer (or an autonomous agent) starts editing, well before any service-side PR is created.

Customer evidence overwhelmingly describes work that happens *before* the service-side PR exists. David's Talos runs the entire validation loop locally and only **pushes** the PR at the end as a deterministic step. Tiago iterates locally on 20 variants and picks one to push. Shreyansh's Architecture Explorer runs locally to ground reasoning on the in-flight change. Service-side PR vocabulary in §7 — *"block the PR"*, *"PR-comment renderer"*, *"never surfaces in the PR UI"* — was implicitly excluding the inner-loop adoption surface that §9.6 declared the first-cycle target.

The reframe is purely vocabulary precision in §7's solution prose. **No solution scope, no S→SR grouping, no priority order, and no problem-statement framing changed.** §6 and §§3–5 retain customer-voice usage of *"PR"* verbatim — that is what customers said.

**What changed in §7:**

| Item | Change |
|---|---|
| §7 preamble | Added a *"What counts as a 'PR' in §7?"* definitional blockquote before SR1, naming the candidate changeset as the actual referent and noting that §6 and §§3–5 retain customer-voice usage. |
| SR1 scope | *"for a given diff or PR"* → *"for a given candidate changeset (a working branch + its intent statement, whether or not a service-side PR has been created yet in GitHub/ADO)"*. Primary first-cycle target sentence extended with *"on the changeset-in-progress — well before a service-side PR exists"*. |
| SR1 boundary | *"operate before the PR is opened"* → *"operate before the changeset is ready for validation"*. Added: *"Does NOT take a hard dependency on … a service-side PR existing."* |
| S1 fallback chain | Replaced source #3 (*"PR title + description"*) with a generalized *"Changeset-attached intent surface"* that enumerates pre-service sources (in-progress commit messages, draft PR description, chat/agent session, Talos work-item approval) alongside the service-side PR description. Closing sentence: *"only the contents of source #3 change"* across local vs. service-side. |
| S2 output line | *"Standardized output schema → PR-comment renderer is shared"* → explicit verdict schema `{facet, status, evidence, confidence, suggested_action}` with rendering delegated to whichever host adapter (S4) is in use. |
| SR2 priority-order row | *"block PRs"* → *"take consequential action (halt a local loop, refuse a push, block a service-side merge)"*. *"gate a PR"* → *"gate any change (local push, service-side merge, or otherwise)"*. *"comment-only mode"* → *"surface-only mode"*. |
| SR2 plain-English definition | Reworded for the candidate-changeset frame; *"blocking PRs"* → *"consequential actions (halting an autonomous loop, refusing a push, blocking a service-side merge)"*; *"comment-only mode"* → *"surface-only mode"*. |
| SR2 *What a developer builds* point #1 | Replaced *"post an FYI comment, post a non-blocking suggestion, block low-risk PRs, block all PRs"* with the **authorized-consequence-translated-by-host-adapter** pattern, naming the same four concrete examples per surface. |
| **S8 stage table** | **Restructured from 3 columns to 5**: *Stage* / *Authorized consequence on failure* / *Local CLI / VS Code agent mode (changeset-in-progress)* / *Service-side PR (CCR / ADO task / hosted plugin)* / *Promotion criteria*. Each stage now reads as *one authorization, per-host translation* — making explicit that `soft_gate` / `hard_gate` are not service-side-only concepts. Promotion-criteria column counts in *changesets*, not *PRs*. |
| S8 closing paragraph | *"a misleading PR comment"* → *"a note the human can ignore (a terminal print, an IDE info diagnostic, or an unblocking PR comment)"*. Added *"on the changeset-in-progress"* and *"(including pre-service ones)"* to underline pre-service applicability. |
| S14 disagreement-capture bullet | *"The PR-comment renderer (S2) makes each verdict explicitly acknowledgeable"* → enumeration of per-host acknowledgement surfaces (CLI thumbs-up/`--ack`; VS Code inline accept/reject or chat reply; service-side PR-comment reaction or gate override). Added: *"Capture works pre-service as well as post-service."* |
| SR4 boundary | *"Does NOT run on the PR itself. Runs before and after the PR moment"* → *"Does NOT run on the changeset itself. Runs before the changeset exists (S5) and after merge (S6)"*. |
| SR5 scope | *"Sits before the PR is opened"* → *"Sits before validation begins on the changeset (whether or not the changeset later becomes a service-side PR)"*. |

**What did NOT change:**

- §6 (P1–P11, D1–D3, W1–W2) — customer-voice problem statements retain *"PR"* exactly as customers said it.
- §§3–5 (per-customer table, deep dives, cross-cutting matrix) — verbatim customer language preserved.
- §1 Executive Summary — uses *"PRs"* in the customer-grounded sense; not edited.
- The workstream / document name (*PR Intent Validation*) — *"PR"* is the established shorthand the workstream is known by externally.
- S3 (loop orchestrator) — *"push PR"* in the list of deterministic Talos steps is correct: pushing IS a service-side action that happens at the end of the loop, after local validation has passed.
- S6 (PR Learn) — runs after merge, always service-side; no edit needed.
- All S# numbers, the S→SR grouping, the SR1→SR5 priority order, and every problem mapping (including the §9.5 S2 mapping correction).
- The local-first commitment from §9.6 — this reframe **strengthens** it by removing implicit service-side assumptions from the prose.
- The SR2 mechanism from §9.7 (policy file + calibration set + promotion-checker job + metrics emitter) — the four artifacts are unchanged; only the way the policy file's stage values are *described* (and the columns of the S8 table) changed.

**Why this matters.** The §9.6 local-first commitment said the first-cycle target is the developer inner loop. But the prose around it still talked as if the validated object was a service-side PR. That gap produced two specific failure modes for the document's audience:

- An engineer on David's Talos team, reading SR2, would see *"blocks PRs"* and conclude SR2 doesn't apply to their flow (Talos validates *before* pushing, so there is no PR to block yet). The new S8 table makes the Talos translation of `hard_gate` explicit: *"agent refuses to push the branch."*
- A developer evaluating *whether to invoke the toolkit on a local branch with no PR* would not find a clear answer in the old text. The new SR1 scope and S1 intent-source chain make explicit that the toolkit works against the changeset whether or not a PR has been opened, and that source #3 in the intent chain falls back gracefully to in-progress commit messages or the chat/agent session that produced the change.

The reframe also reveals an architectural symmetry that was hidden by vague language: **S4 is the host-neutral adapter layer for invocation and rendering; S8's stages are the host-neutral consequence layer.** Together they give the toolkit a complete *"same logic, many hosts"* contract that is the actual mechanism behind the §9.6 host-neutral promise — and the reason a workflow like *"GitHub Copilot CLI + VS Code custom agents on a feature branch"* is a valid first-cycle adoption shape, not a degenerate one.

### 9.9 Seventh-Round Vocabulary Audit (June 1, 2026) — *"Facet"* Renamed to *"Check"*

A seventh-pass review of the doc found that the load-bearing term *"facet"* — adopted from David Coulter's interview language (*"a number of evals that look at different facets"*) — was a barrier to comprehension for readers outside the eval-research vocabulary. The word is precise but abstract; new readers were having to infer from context what a "facet" is.

The rename swaps *"facet"* for **"check"** everywhere the doc speaks in its own voice. A **check** is one specific thing being evaluated about the change — *intent*, *test coverage*, *security*, *tech-debt*, *anti-patterns*, *style/preferences*. Each check gets its own narrow eval rather than being folded into one mega-prompt. The doctrine (D1) is unchanged; only the noun for *the thing* changed.

**What changed:**

| Item | Change |
|---|---|
| §1 executive summary | First bullet rewritten so the *"discrete evals"* point uses *"check"* and adds a one-time gloss: *"A check here means one specific thing being evaluated about the change — e.g. 'does the code match the stated intent?', 'does it introduce a known security issue?', 'does it add the kind of tech-debt that hurts the next change?'."* This is the doc's single definitional appearance — no other instance needs to re-define. Second bullet (*intent facet* → *intent check*) and bottom-line sentence (*multi-facet* → *multi-check*) updated for consistency. |
| §4.2.6 (Tiago conclusion) | *"a style/preferences-as-rules facet alongside intent"* → *"a style/preferences-as-rules check alongside intent"*. |
| §4.4 (David deep dive) | *"first facet"* → *"first check"* (one occurrence, in the per-customer summary bullet). Verbatim quotes from David at lines 242–248 retain *"facet"* / *"facets"* untouched — those are interview text, not editable. |
| §5 cross-cutting matrix | Two row labels: *"Discrete evals per facet beat one mega-prompt"* → *"Discrete evals per check beat one mega-prompt"*; *"Tech-debt / anti-pattern eval is a missing facet"* → *"Tech-debt / anti-pattern eval is a missing check"*. |
| §6B D1 | Heading *"Discrete facet evals over a single mega-prompt"* → *"Discrete per-check evals over a single mega-prompt"*. Commitment sentence reworded around *"check"*. Evidence quote from David retained verbatim (still says *"facets"*) with a parenthetical bridge: *"David's word 'facets' is what this doc now calls **checks**"*. |
| §7 priority-order table | SR1 row and SR3 row scope columns updated (*facet evals* → *per-check evals*; *SR1's facets* → *SR1's checks*). |
| §7 preamble blockquote | Third sentence: *"facet evals"* → *"per-check evals"*. |
| SR1 scope | *"a battery of narrow facet evals"* → *"a battery of narrow per-check evals (one eval per thing being checked — intent, test coverage, security, tech-debt, etc.)"*. The parenthetical re-anchors the reader on what a check is without re-stating the gloss. |
| **S2 framework name and prose** | Heading *"Discrete Multi-Facet Eval Framework"* → *"Discrete Multi-Check Eval Framework"*. Framework prose, the initial-list intro, and the four uses inside the list (*David's missing facet* → *David's missing check*, etc.) updated. |
| **S2 verdict schema field name** | `{facet, status, evidence, confidence, suggested_action}` → `{check, status, evidence, confidence, suggested_action}`. Added inline gloss: *"where `check` names which check produced this verdict — `\"intent\"`, `\"security\"`, `\"tech_debt\"`, etc."* This is a schema-contract change *in name only* — the field's role (it identifies which check produced the verdict) is unchanged. |
| S11 closing line | *"feeds S2's tech-debt facet"* → *"feeds S2's tech-debt check"*. |
| SR2 boundary | *"SR1's facet evals"* → *"SR1's per-check evals"*. |
| S14 calibration-set bullet | *"one file per eval/facet"* → *"one file per eval (one per check)"*. |
| SR3 scope | *"SR1's facets query as a tool"* → *"SR1's checks query as a tool"*. |
| S6 update bullet | *"S2's anti-pattern facet"* → *"S2's anti-pattern check"*. |
| §9.5 corrections table | Two rows describing earlier audits: *"multi-facet eval architecture"* → *"multi-check eval architecture"* and *"discrete evals per facet"* → *"discrete evals per check"* in the row labels (the historical claims those rows describe are unchanged). |
| §9.5 grouping table SR2 row | *"SR1 facets, Talos evals, partner checks"* → *"SR1 checks, Talos evals, partner checks"*. |
| §9.5 S2 mapping-correction row | *"facet framework"* → *"per-check eval framework"*; *"test-coverage facet"* → *"test-coverage check"*. |
| §9.5 S14 row | *"per-facet improvement pipeline"* → *"per-check improvement pipeline"*. |

**What did NOT change:**

- **Verbatim customer quotes** (David at §3 line 59 quote-cell, §4.4 deep-dive quotes at lines 242–248, the evidence quote inside D1 at line 443, and any other italicized customer text). Customers said *"facet"* / *"facets"* and the document preserves what they said.
- The doctrine itself (D1). The commitment to one-narrow-eval-per-thing is unchanged; only the noun changed.
- The §9.8 audit-log table row that quotes the old `{facet, …}` schema. That row is a historical record of what Phase 8 actually did — quoting a schema that was real at the time — and editing it would falsify the audit history. The schema-rename `facet → check` is recorded here, in §9.9.
- All S# numbers, the S→SR grouping, the SR1→SR5 priority order, every problem mapping, and every other piece of structure or scope. This is a vocabulary pass, not a structural one.
- The workstream / document name (*PR Intent Validation*) and all *"PR"* usage — untouched by this pass.

**Why this matters.** *"Check"* is the word every reader — PMs, engineers on partner teams, customer counterparts — already uses for *one thing CI evaluates*. The doc was paying a comprehension tax on every page so it could echo a single interview quote. The customer evidence is preserved (every quote intact; the bridge in D1 makes the linkage explicit), and the doc's own prose now uses the term every reader already has. The schema field rename `{facet → check}` is the most consequential edit: it makes the field's job self-describing in code as well as in prose, and it matches how implementers will actually name it when they build S2.

### 9.10 Eighth-Round Naming Audit (June 1, 2026) — SR1 Renamed to *"PR Validation Agents & Skills"*

An eighth-pass review of the SR1 area name found that *"Toolkit"* — adopted in the fourth-round audit (§9.6) when the workstream committed to local-first, host-neutral distribution — was too generic to convey what SR1 actually ships. *"Toolkit"* can mean almost anything (a CLI, a library, a bundle of templates) and does not signal the **agent-and-skill** shape that the rest of §7 (S1's resolver agent, S2's eval framework, S3's loop orchestrator, S4's host adapter, S9's style/preference rules, S11's anti-pattern library) is built around.

The rename swaps *"PR Validation Toolkit"* for **"PR Validation Agents & Skills"** — a name that:

- **Says what ships.** Every artifact SR1 produces is either an *agent* or a *skill* (in the agent/skill sense already used by GitHub Copilot CLI, VS Code agent mode, and the Agency platform), plus the assets the agents and skills consume (verdict schema, invocation contracts, intent-source adapters). Engineers reading the priority-order table can immediately translate the name into *"a folder of agent and skill files I install into my host."*
- **Aligns with how partners will invoke it.** David's Talos calls SR1 as a non-deterministic step; Jyothi's PR Lifecycle plugin wraps SR1 as a Copilot Code Review sub-agent; a developer in VS Code invokes SR1 through agent mode. In every case, the unit of invocation is *an agent* (and the agent's *skills*). The name now matches the invocation surface.
- **Reinforces local-first.** *"Agents & Skills"* is recognizable plain-asset vocabulary — not platform vocabulary. Reads as something a developer installs, not a service they subscribe to.

No scope, structure, S→SR grouping, or priority order changed. This is a name change in the doc's own voice; the underlying solutions (S1, S2, S3, S4, S9, S11) and their boundaries are identical.

**What changed:**

| Item | Change |
|---|---|
| §7 priority-order table SR1 row (name column) | *"PR Validation Toolkit"* → *"PR Validation Agents & Skills"*. Scope and ranking columns unchanged. |
| §7 SR1 heading | *"### SR1. PR Validation Toolkit"* → *"### SR1. PR Validation Agents & Skills"*. |
| S4 *"Host-Neutral Invocation Contract"* prose | *"Ship the toolkit as a host-neutral core (agents + skills + verdict schema + invocation contracts)… the toolkit's contracts"* → *"Ship **SR1** as a host-neutral core (agents, skills, verdict schema, and invocation contracts)… **SR1's** contracts"*. Switched the noun phrase to **SR1** to avoid the awkward double mention (*"ship the agents & skills as a host-neutral core (agents + skills + …)"*) and to give the prose a clean generic noun. The parenthetical list of components is unchanged. |
| S4 *Talos orchestrator step* bullet | *"David's flow orchestrator calls the toolkit as a non-deterministic step"* → *"calls **SR1** as a non-deterministic step"*. |
| §9.5 grouping table SR1 row | *"(Renamed from 'PR Validation Sub-Agent' in the fourth-round audit — see §9.6.)"* → chained-rename annotation: *"(Rename lineage: 'PR Validation Sub-Agent' → 'PR Validation Toolkit' in the fourth-round audit (§9.6) → 'PR Validation Agents & Skills' in the eighth-round audit (§9.10).)"* Preserves the full history in one place. |

**What did NOT change:**

- **§9.6 audit-log entry** (the *Local-First Commitment*) and its accompanying narrative — every occurrence of *"toolkit"* in §9.6, including the *"ship the toolkit as plain assets — agents, skills, MCP servers, eval rubrics, CLIs"* sentence and the explicit *"Renamed *PR Validation Sub-Agent* → *PR Validation Toolkit*"* row, is left intact. Same rule as the facet → check pass: audit-log entries quote the doc's vocabulary as it stood at the time of that audit, and editing them would falsify the audit history. The current name is recorded here, in §9.10.
- **§9.8 audit-log entry** — the two occurrences of *"the toolkit"* in §9.8's *Why this matters* paragraph (lines describing the candidate-changeset reframe) are likewise left intact for the same reason.
- All S# numbers, the S→SR grouping, the SR1→SR5 priority order, every problem mapping, and every other piece of structure or scope. This is a name change, not a structural one.
- SR1's scope, boundary, customer-problem mapping, primary first-cycle target, and the four-source intent fallback chain (S1) — untouched.
- The local-first commitment from §9.6 — the rename **strengthens** it by removing the generic *"Toolkit"* placeholder and naming the actual plain assets the workstream ships.
- All other SR area names (SR2 *Eval Trust Stages & Feedback Loop*, SR3 *Code Context Grounding Service*, SR4 *Spec & Knowledge Lifecycle*, SR5 *Adjacent Generation Patterns*) — untouched. Each SR uses the noun that fits *that* area; consistency across SRs is not a goal.

**Why this matters.** *"Toolkit"* was a safe, generic placeholder while the SR1 boundary was still being settled. With the boundary now stable (§9.6 local-first, §9.7 SR2 boundary, §9.8 candidate-changeset reframe, §9.9 *check* vocabulary), the name can afford to be more specific. *"Agents & Skills"* matches both **what SR1 actually ships** (files in a repo: `.agent.md`, `.skill.md`, an MCP server, a verdict schema) and **how engineers on partner teams will invoke it** (`copilot install`, the VS Code agent picker, an Agency plugin). A reader scanning the priority-order table for the first time should not have to guess what SR1 is; the new name tells them.

### 9.11 Ninth-Round Naming Audit (June 1, 2026) — SR2 *"Ladder"* Renamed to *"Stages"*

A ninth-pass review of the SR2 area name found that *"Ladder"* — adopted in the fifth-round audit (§9.7) as a plain-language replacement for *"Trust & Calibration Platform"* — was a **metaphor** sitting in the middle of an otherwise literal name (*Eval Trust ____ & Feedback Loop*). The metaphor was also inconsistent with the SR2 prose it governed: S8 is titled *"Trust-**Staged** Eval Deployment Policy"*, the S8 table column header is *"**Stage**"*, the YAML field is *"current trust **stage**"*, the SR2 priority-order row reads *"a trust **stage**"*, and the four positions are spoken of as *stages* throughout. *"Ladder"* was the only place in the entire SR2 area calling the same concept anything else.

The rename swaps *"Eval Trust Ladder & Feedback Loop"* for **"Eval Trust Stages & Feedback Loop"** — a name that:

- **Matches the vocabulary the rest of SR2 already uses.** Zero new terms introduced; one inconsistent term removed. A reader who lands on the SR2 heading now sees the same word the S8 heading, the S8 table, the policy-file field, and the priority-order row all use.
- **Stays plain.** *"Stages"* is everyday deployment vocabulary (deployment stages, release stages, pipeline stages). Engineers do not need a metaphor decoder to read it.
- **Preserves the discrete-and-sequential meaning.** Four ordered positions — `advisory` → `shadow` → `soft_gate` → `hard_gate` — carry the upward-progression sense in the S8 table itself, which is where it belongs. The area name no longer has to do that work via metaphor.

No scope, structure, S→SR grouping, priority order, or problem mapping changed. This is a one-word vocabulary fix in the doc's own voice.

**What changed:**

| Item | Change |
|---|---|
| §6C W2 *Why it matters* line | *"promotion thresholds for S8's trust ladder"* → *"promotion thresholds for S8's trust stages"*. |
| §7 priority-order table SR2 row (name column) | *"SR2 — Eval Trust Ladder & Feedback Loop"* → *"SR2 — Eval Trust Stages & Feedback Loop"*. Scope and ranking columns unchanged. |
| §7 SR2 heading | *"### SR2. Eval Trust Ladder & Feedback Loop"* → *"### SR2. Eval Trust Stages & Feedback Loop"*. |
| §9.5 grouping table SR2 row | *"(Renamed and rewritten from 'Trust & Calibration Platform' in the fifth-round audit — see §9.7.)"* → chained-rename annotation: *"(Rename lineage: 'Trust & Calibration Platform' → 'Eval Trust Ladder & Feedback Loop' in the fifth-round audit (§9.7) → 'Eval Trust Stages & Feedback Loop' in the ninth-round audit (§9.11).)"* Preserves the full history in one place. |
| §9.6 closing paragraph | *"per SR2's trust ladder"* → *"per SR2's trust stages"* in the local-first commitment narrative's closing sentence (the cross-reference is live narrative prose, not an audit-log quotation of a historical phrase). |

**What did NOT change:**

- **§9.7 audit-log entry** — every occurrence of *"Ladder"* in §9.7, including the explicit *"Renamed *Trust & Calibration Platform* → *Eval Trust Ladder & Feedback Loop*"* row, is left intact. Same rule as the facet → check and Toolkit → Agents & Skills passes: audit-log entries quote the doc's vocabulary as it stood at the time of that audit, and editing them would falsify the audit history. The current name is recorded here, in §9.11.
- **§9.10 audit-log entry** — the one occurrence of *"Eval Trust Ladder & Feedback Loop"* in §9.10's *"What did NOT change"* list (the line that enumerates the other SR names as untouched-by-that-audit) is likewise left intact for the same reason.
- All S# numbers, the S→SR grouping, the SR1→SR5 priority order, every problem mapping, and every other piece of structure or scope. This is a vocabulary fix, not a structural one.
- SR2's scope, boundary, customer-problem mapping, the four-artifact mechanism (policy file + calibration set + promotion-checker job + metrics emitter), and the four-position policy (`advisory` / `shadow` / `soft_gate` / `hard_gate`) — untouched.
- The plain-language sense of the area name. *"Stages"* carries the same meaning *"Ladder"* carried; only the register changed (metaphor → literal).
- All other SR area names (SR1 *PR Validation Agents & Skills*, SR3 *Code Context Grounding Service*, SR4 *Spec & Knowledge Lifecycle*, SR5 *Adjacent Generation Patterns*) — untouched.

**Why this matters.** Area names show up in the priority-order table, the grouping table, and the SR section headings — they are the doc's high-signal labels. *"Ladder"* forced every reader to translate the metaphor before reading SR2's prose; once inside SR2, they then read *"stage"* in eight different places and had to mentally reconcile the two words. Removing the metaphor removes the translation step and aligns the label with the field it governs in the policy file an implementer will actually write.

---

*End of document.*
