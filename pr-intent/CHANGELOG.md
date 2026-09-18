# PR-Intent Artifacts Changelog

This changelog versions only the PR-intent automation artifacts in this repository.

Versioning model:

- **Bundle semver (tagged as `vX.Y.Z`)**: tag + changelog entry (for example `v0.0.1`)
- **Per-file versions**: each included artifact carries `bundle`, `bundle_version`, and `artifact_version` frontmatter fields
- **Sync rule**: in normal releases, all three included artifacts keep the same `bundle_version` and `artifact_version`

## v0.0.1 - 2026-06-12

Initial scoped version tag for PR-intent assets.

Included artifacts:

- `.github/agents/pr-intent-verifier.agent.md`
- `.github/agents/pr-intent-resolution-coordinator.agent.md`
- `.github/skills/intent-verification/SKILL.md`

Out of scope (intentionally excluded):

- Dev-Trio related agents/skills
- PM agents/skills (except when used by coordinator at runtime)
- Researcher and other non PR-intent agents
