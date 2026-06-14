# Claims Audit — blast-radius-agent

**Date:** 2026-06-12 · **Audited:** README.md, AGENTS.md vs code on `main`

## Findings (ranked by judge visibility)

### HIGH — Live GitLab Duo / Orbit integration is unverifiable from the repo

- **Claim:** "Mention `@blast-radius` in any issue or MR comment, and the agent traverses Orbit's dependency graph…"
- **Reality:** `agent.yml`, the skill, and the Orbit query contract (`docs/ORBIT_CONTRACT.md`) are all present and coherent, but nothing in the repo proves a deployment ever answered an `@blast-radius` mention.
- **STATUS 2026-06-14:** The personal namespace project (`prestige-worldwidest/blast-radius-agent`) has `duoAgenticChatAvailable: false`. AI agent creation requires the hackathon-provisioned namespace (`gitlab-ai-hackathon/transcend/`). The full repo (36 files, all tests passing) is synced. Agent creation is UI-only with no REST/GraphQL API. The system prompt, agent manifest, and skill are ready. The user needs to complete Contributor Platform provisioning to get the project in the correct namespace, then create the agent in the browser UI. See `docs/AGENT_SETUP.md`.
- **Fix:** Once the hackathon namespace project is provisioned, the user creates the agent via GitLab UI (Automate → Agents → New Agent) using the pre-written system prompt. Screenshot the deployed agent working in a real MR.

### FIXED TODAY — `validate` CI was red on every branch

The workflow asserted `agent-manifest.json`, which was renamed to `agent.yml`;
the check failed on every push since. Fixed on `ci/smoke-tests` (workflow now
validates `agent.yml`). Red CI on the repo front page is exactly the kind of
discovered fiction the playbook warns about.

## Verified claims

- Offline mode works end-to-end: `blast-radius src/auth/tokens.py --function checkToken --graph tests/fixtures/sample_graph.json` produces direct + transitive impact with depths, exit 0; `--json` mode works. ✓
- Zero runtime dependencies as claimed in pyproject. ✓
- 29 tests pass (CLI, engine, orbit SQL, risk scoring). ✓
- Cycle handling demonstrably present (fixture includes a cycle; output traverses it without hanging). ✓
