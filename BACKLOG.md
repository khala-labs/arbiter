# Arbiter Backlog

Post-MVP items deferred from the initial build (see the design spec's Non-goals and
the final review), plus the open Arbiter items from the khala-labs organization roadmap
(`khala-labs/.github` `ROADMAP.md`). None block the MVP; all are tracked here.

Last checked against the code on 2026-11-14 (`3e6e6ac`). No item below was closed by a
merged change; each one is still open in the code.

## Performance
- **`check_gate` full-scan (review I-2):** `Gate.check_gate` calls `Ledger.status()`
  with no argument, which loads + hashes every artifact on every gated edit. Fine at
  solo scale; for a large governed repo, switch to `ledger.status(active_spec_id)`
  (scoped scan) and only opportunistically full-scan on `index()`. Since 2026-09-05
  `status()` is read-only (SPEC-arbiter-status-is-read-only): it flags `needs_review` /
  `tampered` and no longer repairs any file, so a scoped scan changes only cost.

## Artifact format fidelity (spec §4)
- **`review_ref` field (review I-3):** add a `review_ref: .reviews/<id>.md` pointer to
  artifact frontmatter on `approve()` so a human/tool can navigate artifact → review
  evidence without knowing the naming convention.
- **`version` field (review I-3):** increment a `version` on living-spec evolution
  (re-review after edit). Requires bump logic in `approve()`/`status()`.

## Enforcement precision
- **`governs:` path-glob mapping:** the MVP gate is path-agnostic (any approved active
  spec allows any source edit). Add optional `governs: [src/**]` globs in spec
  frontmatter so the gate maps edits to the spec that actually governs them.

## Review depth
- **Comprehension questions (Q4-B):** optional, high-risk-spec-only "what's the riskiest
  assumption?" gate on top of issue-disposition. Designed but not built.
- **Re-critique after edit:** `approve()` proves a spec was *edited* after an `accepted`
  disposition (hash change) but not that the edit *correctly addresses* the issue.
  Optionally force a re-`critique()` before final approval.

## Drift
- **`stale` auto-detection:** detect spec↔code drift (e.g. spec-derived tests failing,
  or Nexus design-vs-observation diff) and auto-flag artifacts `stale`.

## Open items from the organization roadmap
- **`resolves:` frontmatter key:** no code reads it. It is a convention, not a mechanism
  (ADR-0012 in the Khala ledger).
- **Re-critique overwrites the sidecar:** `critique()` writes a new review sidecar
  `.reviews/<id>.md` each run and numbers issues from `I-001` again. Dispositions must
  be rewritten against the last round's issue ids.
- **`ARBITER_CRITIC=claude-code` missing-body report:** `ClaudeCodeCritic` sometimes
  reports that the body is missing for a complete document; running it again returns
  real issues. The cause is not yet found.
- **Khala repository hook registration:** in the Khala repository,
  `.claude/settings.json` does not register Arbiter's PreToolUse gate hook
  (`hooks/pretooluse_gate.py`), so there the gate runs only through the CLI and CI. This is configuration in that repository, not code in
  Arbiter.

## Team
- **Multi-user / real approver identity:** `approved_by` is recorded but unauthenticated
  (solo MVP). Add identity + a shared ledger backend for team use.

## Server test seam
- **MCP wiring integration test (review M-4):** add a test that drives
  `begin_implementation → check_gate → end_implementation → check_gate` through
  `build_app` to cover shared `Gate` state across tool calls (currently only the core
  objects are exercised end-to-end). `tests/test_server.py` still only checks that the
  12 tools are registered.
