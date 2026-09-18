# Iberostar Restaurant Assistant: from vision to working POC

_2026-09-18_
_This story captures the full arc of the project: product scope, functional specification, technical plan, implementation, live validation, and intent-verification alignment._

---

## Opening scene: the idea and the constraints

The project started with a clear business problem: Iberostar needed a lightweight, guest-friendly assistant that could help a customer decide what to eat and drink, offer recommendations, and confirm a simple order flow without introducing heavy operational complexity. The original vision document established the goal, the target users, and the POC boundary: it should be a mobile-ready web experience, multilingual, and focused on a single restaurant or buffet context.

The first artifact in the project was the vision scope, which framed the problem and set the expected phase 1 outcome. It was intentionally realistic about scope: the prototype was not a production ordering system, and it would not integrate with PMS, POS, or kitchen systems. We also had a strong architectural decision from the start: avoid a backend in the first POC unless the browser-to-agent connection was impossible.

**Key files:**
- [Specs/Vision-Caso-Uso/vision-scope-iberostar-specs.md](../Specs/Vision-Caso-Uso/vision-scope-iberostar-specs.md)
- [Specs/Vision-Caso-Uso/especificacion-funcional-asistente-gastronomico-iberostar.md](../Specs/Vision-Caso-Uso/especificacion-funcional-asistente-gastronomico-iberostar.md)
- [Specs/Vision-Caso-Uso/implementation-plan.md](../Specs/Vision-Caso-Uso/implementation-plan.md)

> **User:** "Si podemos conectar el agente de Foundry desde JavaScript, mejor: no backend para la POC. Lo dejamos para fases futuras."

That decision became the anchor of the engineering approach. It kept the prototype fast, easy to demo, and helpful for validating user flow without taking on the complexity of a backend layer that was not needed yet.

---

## The project moved from product intent to detailed product requirements

Once the vision was set, the next step was turning the idea into a concrete functional specification. The functional doc clarified the exact flows: guest can ask for recommendations, filter by preferences or budget, inspect details of dishes and wines, request clarification when data is missing, and confirm an order summary before submission.

The specification also captured the product guardrails: no direct charging, no production integrations, no guessed allergen claims, and strict handling of missing or uncertain information. These constraints mattered because the project was designed as a demo system, not a production operation. The functional spec became the contract for the subsequent implementation decisions.

This is where the product story becomes more disciplined: the system had to not only chat, but also support a defined conversational flow, present recommendations based on a catalog, and protect the user from unsafe assumptions.

**Key files:**
- [Specs/Vision-Caso-Uso/especificacion-funcional-asistente-gastronomico-iberostar.md](../Specs/Vision-Caso-Uso/especificacion-funcional-asistente-gastronomico-iberostar.md)

---

## The implementation plan decided the actual architecture

The implementation plan made the POC strategy explicit. It specified a frontend-first approach in Vue, with direct browser calls to Azure AI Foundry when permitted, and a local fallback if direct browser authentication or endpoint restrictions prevented the call. This was a deliberate tradeoff: the app would stay lean and still produce a working demo, while the .NET backend would be treated as a future phase when there was a real business reason for central validation, stronger secret handling, or enterprise integration.

The document also defined the architecture in clear layers: the web experience, Azure AI Foundry agent, and demo data/backoffice layer. This structure allowed the app to be built without a dedicated backend while still keeping the POC realistic enough to demonstrate the user journey.

**Key files:**
- [Specs/Vision-Caso-Uso/implementation-plan.md](../Specs/Vision-Caso-Uso/implementation-plan.md)

> **User:** "Si es posible conectar al agente desde JavaScript, hagámoslo en lugar de crear backend para la POC. Explica eso en el plan."

The plan was then revised to make that decision explicit: browser-first Azure connection for the POC, backend deferred to future phases.

---

## The app was built as a POC frontend-first experience

The implementation was done under the project’s web app folder, in a minimal Vue + Vite setup. The first working version was focused on the user journey: allow the guest to type a request, receive a response, see suggestions, and pick items into a demo order. The app read Azure Foundry endpoint settings from environment variables and added a graceful fallback if those were missing or if the browser call failed.

The code used a browser-first approach with a direct `fetch` call to the Azure AI Foundry OpenAI-compatible endpoint. The agent persona and business safeguards were encoded in the system prompt, and the app also included local catalog logic so it remained functional even when the live agent was not available. That was important for demo reliability.

**Key implementation files:**
- [src/package.json](../src/package.json)
- [src/src/App.vue](../src/src/App.vue)
- [src/.env](../src/.env)

The implementation included:
- a conversational interface,
- multilingual messages,
- recommendation logic,
- order summary and item selection,
- local persistence for session state,
- and fallback behavior in case the Foundry endpoint was unavailable.

---

## The first big balancing loop: documentation vs code

As soon as the app took shape, the team noticed a gap: the code was functional, but the product narrative had to be updated to reflect the actual implementation decisions. The initial UI was simple, but the specification and plan were more ambitious in their business description. The fix was not to reduce the product scope, but to align the documents with the implemented POC.

This is the first real “balance loop” of the journey: we adjusted the docs so they reflected what the code was actually doing, and also improved the code so it more closely matched the intended product behavior.

The result was a better match between the experience and the spec: the project was now a realistic POC around a frontend-first architecture, not a half-finished backend-driven design.

**Loop records:**
- [Specs/Vision-Caso-Uso/loop-reports/loop-01.md](../Specs/Vision-Caso-Uso/loop-reports/loop-01.md)
- [Specs/Vision-Caso-Uso/loop-reports/loop-02.md](../Specs/Vision-Caso-Uso/loop-reports/loop-02.md)
- [Specs/Vision-Caso-Uso/loop-reports/loop-03.md](../Specs/Vision-Caso-Uso/loop-reports/loop-03.md)

---

## The app evolved into a more complete demo flow

The implementation matured from basic chat into a richer recommendation and ordering demo. The app gained a more realistic product flow:

- catalog with dishes, wines, and desserts,
- recommendation cards ranked by match,
- simple budget and preference handling,
- order summary and selected items,
- order history and backoffice-like view,
- Spanish and English toggle,
- voice input via browser APIs where supported,
- a secure fallback to local catalog logic when Azure is unavailable.

This is exactly where the code and the documents became genuinely aligned. The app was now not just a chat toy; it was a demo of how the guest journey could work in the POC: ask, get recommended options, confirm selection, and inspect a preliminary order summary.

**Key logic within the app:**
- `callFoundry()` for the Azure browser call
- `buildLocalResponse()` for fallback behavior
- `getRecommendationMatches()` for catalog-driven ranking
- `addItem()`, `removeItem()`, and `confirmOrder()` for order handling
- `startVoiceInput()` and locale toggling for user experience

The live implementation is in [src/src/App.vue](../src/src/App.vue).

---

## Validation: build and browser execution

Once the app was in a stable state, the project moved to validation. We verified the app could be built and launched locally. The package scripts were checked, dependencies were installed, and a build pass was confirmed.

The build command succeeded, and the app was then served locally in the browser. This provided direct evidence that the POC was operational from a frontend-first architecture. The browser view showed the expected assistant experience and interaction flow in a live environment.

**Evidence of success:**
- `npm install` and `npm run build` succeeded in the project root
- Vite served the app in the browser at the local preview URL
- The app rendered the main UI and could run without a backend

This was a critical milestone because it proved the technical strategy was viable: a browser-first POC can be validated without adding the .NET layer immediately.

---

## Intent verification and alignment review

The repo also contains the intent-verification work that provides a disciplined way to check whether code and docs still align. That process is designed to compare the intent documents (vision, spec, plan) against the implementation and classify any drift as:

- CODE_BEHIND
- CODE_AHEAD
- CONFLICT
- ALIGNED

This matters here because the project evolved quickly, and documentation drift is a normal consequence of iterating on a POC. The verification discipline gives a systematic way to check whether the code is doing what the spec says, or whether the docs need to be updated to match the actual product reality.

This repo includes the verification strategy, reports, and workflow definitions that explain the underlying method. Those are not just generic docs; they are part of the project’s own process for keeping the implementation honest.

**Relevant verification artifacts:**
- [Specs/01-Global/02-pr-intent-verification-strategy/pr-intent-verification-strategy.md](../Specs/01-Global/02-pr-intent-verification-strategy/pr-intent-verification-strategy.md)
- [Specs/02-Features/F1-pr-intent-verifier-agent/pr-intent-verifier-agent-specs.md](../Specs/02-Features/F1-pr-intent-verifier-agent/pr-intent-verifier-agent-specs.md)
- [Specs/02-Features/F2-pr-intent-verification-workflow/pr-intent-verification-workflow-design.md](../Specs/02-Features/F2-pr-intent-verification-workflow/pr-intent-verification-workflow-design.md)

The intent-verification loop is the same pattern we followed informally in this project’s iteration cycles: compare product expectation with implementation reality, update whichever side was behind, and avoid shipping a misleading story.

---

## What we learned

### About the technology

- A Vue SPA can validate the POC quickly and remain demonstrable without a backend.
- Browser-side Azure Foundry calls are viable only when the endpoint accepts the security model and the browser is allowed to use the credentials or policy required by the environment.
- Local fallback logic is essential for live demos and for supporting environments where the AI endpoint is unavailable or restricted.
- Web Speech API is a useful enhancement, but text input remains the safe fallback.

### About the product process

- The best way to keep a POC credible is to maintain a tight loop between the UX and the docs.
- Vision and implementation must evolve together; otherwise the product story becomes disconnected from reality.
- A backend is not always the first move. For discovery and validation, a browser-first architecture can be the right choice.
- Documentation of tradeoffs matters: the project should explain clearly when a backend is deferred to future phases and why.

### About project discipline

- Validation is not just a build check; it is also a product alignment check.
- The POC is stronger when it can explain the “why” behind the architecture decision, not only the features it contains.
- Intent verification creates a disciplined way to catch drift early and prevent the code and product story from diverging for long.

---

## What’s next

The immediate next step is to continue refining the balance between the specification and the actual implementation. The project is now in a healthy place: the docs express the correct POC strategy, the code reflects a working frontend-first experience, and the app has been validated by build and browser execution.

The remaining work is optional and depends on the project objective:

1. validate the real Azure Foundry live request in the actual endpoint environment,
2. strengthen the business rules and order-validation logic,
3. refine the recommendation engine and demo scenarios,
4. and continue using the verification flow to keep product documents and implementation aligned as the POC evolves.

---

_Written: 2026-09-18_
