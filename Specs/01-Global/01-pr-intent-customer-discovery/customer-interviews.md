

### **PR Intent Validations — Customer Discovery (this week)**

This is a parallel workstream independent from the AI-First Starter Pack, to address **spec↔code drift in AI-assisted PRs** — verifying that PR changes actually match the original intent (spec, architecture, task plan). It will integrate with the PR Lifecycle / AI Code Review roadmap. Active customer discovery this week to validate the problem and shape the product.

| Date | Interviewee | Team / Context | Key Signal |
|------|-------------|----------------|------------|
| May 18 | Rupam Mittal, Jyothi Marreddy | *PR Lifecycle team* | Deep-dive sync — intent validation aligns with their Cycle 3 plan and the AI Code Review 1-pager |
| May 20 | Tiago Macarios | *Engineering Systems* | Intent drift is **not** a major issue for his workload — signal that the pain is workload-specific, not universal |
| May 21 | Shreyansh Agrawal | *ODSP* | Docs lag code by months; mono-repo complexity makes automation hard; strong preference for iterative coordinator+coder+PM loops over static reports |
| May 21 | David Coulter | *Autonomous dev workflows (Talos)* | Validates discrete evals (intent, security, tech debt) over monolithic prompts; current evals "low trust, high variability"; agreed to share Talos and plan a joint demo in June |
| May 21 | [Taylor Williams](https://microsoft-my.sharepoint.com/personal/cesardl_microsoft_com/_layouts/15/stream.aspx?id=%2Fpersonal%2Fcesardl%5Fmicrosoft%5Fcom%2FDocuments%2FRecordings%2FInterview%20PR%20Intent%20Verifications%20and%20your%20potential%20Feedback%20%20Pain%20Points%20%20Wanted%20Features%20%28Taylor%20Williams%29%2D20260521%5F140229%2DMeeting%20Recording%2Emp4&referrer=StreamWebApp%2EWeb&referrerScenario=AddressBarCopied%2Eview%2E57839d36%2Ddc3c%2D461f%2D8672%2D32a02d82efd2&share=cQrKIQKHaz6fRYx31tx8eaolEgUCd60zRQ3th9i4xDbSxvXbqQ) | *Outlook* (recently moved; prior distributed systems work) | No formal intent-validation today (tests are the de-facto enforcement); already uses agent-swarm PR review before humans; key idea — have the agent **generate clarifying questions** instead of pure validation; iterative loops alone won't fix model reasoning gaps (e.g., concurrency, distributed systems) — **human-in-the-loop is essential** |

**Emerging pattern across interviews:** discrete, evidence-based evals + orchestrated multi-agent loops are preferred over single-pass validation. The problem is real and high-value for complex AI-assisted workflows but is **not universal** across all dev scenarios.

---