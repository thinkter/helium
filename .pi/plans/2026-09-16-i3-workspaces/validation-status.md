# Workspace implementation validation policy

2026-09-16 — explicit user override for continued implementation from I3-T07.

- **Do not run tests** (including existing pure/model tests, browser tests and
  interactive tests). Test sources may be added and test objects compiled.
- **Do not restore or download test resources.** The existing native test build
  blocker remains; do not change its dependency graph to bypass missing inputs.
- Continue using production compilation, generated-command test-object
  compilation where feasible, static review and source/patch provenance.
- Object compilation is not a linked browser build or runtime verification.
  No browser launch, native interaction or runtime acceptance is claimed.
- T06's isolated disabled host is an approved implementation foundation despite
  its pending runtime acceptance. T06 and T07 runtime acceptance are deferred by
  user direction, not passed. Keep production routing and feature exposure off.
- Later tasks may proceed under this policy while recording implementation
  completion separately from runtime/ISC acceptance. Release readiness still
  requires an explicitly authorized runtime-validation phase.

2026-09-17 — T10 implementation closed after the finite modal attachment P1
fix; see `T10-closure-result.md`. Runtime remains **UNVERIFIED**; T11 may proceed.
Both disabled gates and the no-public-exposure restriction remain in force.

This supersedes the earlier plan/workflow instruction to run applicable tests
and the blocker report's proposed resource-restoration gate. It does not grant
permission to restore resources, execute tests, or enable the incomplete mode.
