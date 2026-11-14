# Changelog

All notable changes to **arbiter** will be documented in this file.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Added

- `ClaudeCodeCritic`, a keyless critic that runs `claude -p` on the host. Select it with `ARBITER_CRITIC=claude-code` (default `anthropic`; an unknown value raises). `ARBITER_CRITIC_TIMEOUT` (default 300 s) and `ARBITER_CRITIC_MODEL` configure it. (2026-07-12)
- CLI commands `begin-implementation <spec-id>` and `end-implementation`, so the gate that `check-gate` reports on can be opened and closed from the terminal. (2026-09-05)

### Changed

- `status()` no longer writes. A SPEC whose content hash is missing or stale is reported with `needs_review` and its file is left alone; `arbiter status` prints the `needs_review` and `tampered` flags. `index()` does not list flagged artifacts under the approved heading. (2026-09-05, SPEC-arbiter-status-is-read-only)
- Migrated to the `mcp` 2.x API (`mcp.server.MCPServer`); the requirement is `mcp>=2.0.0,<3`. (2026-08-02)

### Fixed

- Writers no longer translate LF to CRLF on Windows. (2026-09-05)
- The CLI converts a Git Bash path (`/c/...`) in `ARBITER_ROOT` / `ARBITER_DOCS` to its Windows form, and exits with code 2 naming the variable when the directory does not exist. (2026-08-16)

- `Ledger` walks `specs/` and `adr/` in sorted order. `glob` returns filesystem order (alphabetical on NTFS, arbitrary on ext4), so `index()` wrote a different `INDEX.md` on Windows and Linux for the same ledger.

### Breaking Changes

- **Public class renames:** The two public sink types in `specledger.publish` are renamed:
  - `KhalaSink` (Protocol) → `NexusSink`
  - `KhalaHttpSink` → `NexusHttpSink`

  Any code implementing or instantiating these types must be updated. Type annotations using `KhalaSink` in function signatures are now invalid.

- **Config key:** The publish-config dictionary key `khala` → `nexus`. Callers passing a `config` object must rename the key (e.g., `{"khala": {"url": ...}}` → `{"nexus": {"url": ...}}`).

- **Error string:** The not-configured sentinel reason changed from `"khala not configured"` to `"nexus not configured"`. Code matching on this string (e.g., `result["reason"] == "khala not configured"`) must be updated.

- **Transport — `NexusHttpSink` removed; A2A is the sole transport.** The bespoke point-to-point HTTP POST sink is gone. `publish()` always builds an `A2ANexusSink` (A2A JSON-RPC: discover agent card → `ingest_governed_doc` skill → governed-doc DataPart with a write token). The `SPECLEDGER_NEXUS_TRANSPORT` env flag and the `nexus["transport"]` config key are no longer consulted. The publish-config needs a Nexus base URL (`nexus["url"]` or `nexus["base_url"]`) and a write-capability token (`ARBITER_NEXUS_TOKEN` env or `nexus["token"]`). Done once the A2A provenance loop was proven end to end against a real DB (SPEC-specledger-a2a-publish-phase3 §16).
