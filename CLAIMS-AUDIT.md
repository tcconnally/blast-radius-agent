# Claims Audit — blast-radius-agent

**Date:** 2026-06-12 · **Audited:** README.md, AGENTS.md vs code on `main`

## Findings (ranked by judge visibility)

### HIGH — Live GitLab Duo / Orbit integration is unverifiable from the repo

- **Claim:** "Mention `@blast-radius` in any issue or MR comment, and the agent traverses Orbit's dependency graph…"
- **Reality:** `agent.yml`, the skill, and the Orbit query contract (`docs/ORBIT_CONTRACT.md`) are all present and coherent, but nothing in the repo proves a deployment ever answered an `@blast-radius` mention. The manifest itself says field names are unconfirmed against the platform version. If a judge asks for a live demo, only the offline mode is demonstrable.
- **Fix:** capture one screenshot/transcript of the deployed agent responding in an MR, or label the platform integration "deployable, validated offline".

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
