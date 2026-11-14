## Status

2026년 11월 8일 `arbiter/` 디렉터리는 Khala 모노레포(`khala-labs/khala`)에서 28개 커밋의 git 이력을 보존한 채 본 저장소로 분리되었습니다. 본 저장소는 아카이브되지 않은 활성 상태이며, Python 3.13 환경에서 pytest를 실행하는 자체 CI를 갖추고 있습니다. Arbiter는 현재 Khala 저장소의 ADR 및 SPEC 승인 게이트로 계속 사용되고 있습니다. Arbiter가 기록하는 원장(`specs/`, `adr/`, `.reviews/`, `INDEX.md`)은 Khala 저장소에 유지되며, `ARBITER_DOCS` 환경 변수를 Khala 저장소의 `records/` 디렉터리로 설정하여 참조합니다. Arbiter의 공식 문서는 Khala 문서 사이트(`https://khala-labs.github.io/khala/tools/arbiter/`, 한국어: `https://khala-labs.github.io/khala/ko/tools/arbiter/`)가 계속 기준 문서로 유지됩니다. Arbiter 자체에 대한 이슈는 본 저장소에 등록해 주시고, Khala 원장 관련 이슈는 `khala-labs/khala`에 등록해 주시기 바랍니다.

On 2026-11-08, the `arbiter/` directory was extracted from the Khala monorepo (`khala-labs/khala`) into this repository with its 28-commit git history preserved. This repository is live, not archived, and now has its own CI running pytest on Python 3.13. Arbiter is currently in use as the approval gate for ADRs and SPECs in the Khala repository. The ledger Arbiter writes (`specs/`, `adr/`, `.reviews/`, `INDEX.md`) stays in `khala-labs/khala` and is located by pointing the `ARBITER_DOCS` environment variable to the `records/` directory of the Khala repository. Arbiter's page in the Khala documentation stays the reference (`https://khala-labs.github.io/khala/tools/arbiter/`, Korean: `https://khala-labs.github.io/khala/ko/tools/arbiter/`). Issues about Arbiter itself should be filed in this repository, while issues about the Khala ledger belong in `khala-labs/khala`.

---

# Arbiter

Arbiter is a Python MCP server and Claude Code `PreToolUse` hook that records AI-generated ADRs and design specs in a consistent Markdown+frontmatter format, enforces accountable review (AI critique → human issue-disposition → sign-off) before any code edits are written, and optionally publishes approved documents to an external Nexus sink. The gate is active during implementation: `begin_implementation` arms it, `end_implementation` disarms it. Until a spec is approved and stamped with a content hash, all `Write`/`Edit`/`MultiEdit` calls targeting non-exempt source paths are blocked.

---

## Install

Development install from a checkout uses `constraints.txt` to pin the resolved dependency set that CI uses. CI runs pytest on Python 3.13; `pyproject.toml` requires Python >=3.11:

```bash
pip install -e ".[dev]" -c constraints.txt
```

A consumer installs from git pinned to a commit (the Khala repository's CI installs it this way). The install provides the `arbiter` command (CLI) and the module `khala.arbiter.server` (MCP server):

```bash
pip install "khala-arbiter @ git+https://github.com/khala-labs/arbiter@<commit>"
```

---

## CLI (run the gate by hand)

The MCP server is for agents wired through `.mcp.json`. **A human — or an agent without that
wiring — runs the same gate from the terminal:**

```bash
arbiter record spec "결제 정책 SPEC"          # register a draft → prints its id
arbiter critique SPEC-결제-정책-spec           # run the critic → prints issues
arbiter approve  SPEC-결제-정책-spec \         # apply dispositions + sign off
    --dispositions disp.json --approver eisen
arbiter status                                # list all artifacts (or pass one id)
arbiter begin-implementation SPEC-결제-정책-spec # arm the gate for this spec
arbiter check-gate src/app.py                 # is this path allowed right now?
arbiter end-implementation                    # disarm the gate
```

The CLI has seven commands: `record`, `critique`, `approve`, `status`, `begin-implementation`, `end-implementation`, and `check-gate`. `begin-implementation` checks that the id exists first and exits 1 on an unknown id. The marker it writes records `set_by: "cli"`; the MCP tool records `"agent"`. `end-implementation` on a closed gate is not an error; it prints that no implementation was open. `status` prints the `needs_review` and `tampered` flags next to each artifact. `supersede`, `index`, `publish`, `promote_external`, and `guide` exist only as MCP tools.

`--dispositions` is a JSON file, one object per open issue:

```json
[
  { "issue_id": "I-001", "disposition": "accepted" },
  { "issue_id": "I-002", "disposition": "rejected", "reason": "관측이 아니라 추측이다" }
]
```

`rejected` and `deferred` require a `reason`; `accepted` requires the document body to have
actually changed since the critique (an unchanged body with an accepted issue is refused). The CLI
calls the *same* `ledger` / `critique` / `review` functions the MCP tools call — it is a thin
surface, not a second implementation.

Root and docs directory come from `ARBITER_ROOT` / `ARBITER_DOCS` (default: `.` and `./docs`).
On Windows, the CLI converts an MSYS-style path such as `/c/work/project` (what Git Bash gives as `$PWD`) to `C:/work/project`; a root or docs directory that does not exist stops the CLI with exit code 2.

---

## Critic backends

The `critique` step uses a critic chosen by the `ARBITER_CRITIC` environment variable (default `anthropic`); an unknown value is rejected with an error so a typo is not hidden.

| `ARBITER_CRITIC` value | Class | What it needs |
| --- | --- | --- |
| `anthropic` | `AnthropicCritic` | `ANTHROPIC_API_KEY` |
| `claude-code` | `ClaudeCodeCritic` | `claude` command on `PATH` (no API key) |

`AnthropicCritic` calls the Anthropic API and reads `ANTHROPIC_API_KEY` only when a critique actually runs, so the MCP server boots and other tools run without a key (default model `claude-opus-4-8`, `max_tokens` 4096). `ClaudeCodeCritic` is keyless, running `claude -p` (Claude Code) on the host with flags that turn off built-in tools, MCP servers, user setting sources, and session persistence, so text inside a reviewed document cannot turn the review into host execution. `ARBITER_CRITIC_TIMEOUT` sets the timeout in seconds (default 300); `ARBITER_CRITIC_MODEL` optionally passes `--model`. Both critics send the same prompt and return the same shape (category, severity, description); any failure (non-zero exit, timeout, unparseable output) raises `CritiqueError`: critique fails closed.

---

## MCP Server Registration (`.mcp.json`)

Add to your project's `.mcp.json` (or the global `~/.claude/mcp.json`):

```json
{
  "mcpServers": {
    "arbiter": {
      "command": "python",
      "args": ["-m", "khala.arbiter.server"],
      "env": {
        "ARBITER_ROOT": "/abs/path/to/your/project",
        "ARBITER_DOCS": "/abs/path/to/your/project/docs",
        "ARBITER_CRITIC": "anthropic",
        "ANTHROPIC_API_KEY": "sk-ant-..."
      }
    }
  }
}
```

`ARBITER_ROOT` — project root where `.arbiter/` state lives (defaults to `.`).
`ARBITER_DOCS` — directory where spec and ADR Markdown files are written (defaults to `$ARBITER_ROOT/docs`).
`ARBITER_CRITIC` — `anthropic` (default) or `claude-code`; see [Critic backends](#critic-backends).
`ANTHROPIC_API_KEY` — required only when `ARBITER_CRITIC=anthropic` and the `critique` tool runs. Leave it out for `claude-code`.

---

## PreToolUse Hook Registration (`settings.json`)

Register the gate hook in `.claude/settings.json` (project-level) or `~/.claude/settings.json` (global):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python /abs/path/arbiter/hooks/pretooluse_gate.py"
          }
        ]
      }
    ]
  }
}
```

The hook reads the tool payload from stdin and exits `0` (allow) or `2` (block). It respects the same `ARBITER_ROOT` / `ARBITER_DOCS` environment variables.

If `ARBITER_ROOT` / `ARBITER_DOCS` are unset, the hook defaults them to the `cwd` in the hook payload and `<cwd>/docs`. If the docs directory does not exist, the hook allows the edit (the project is treated as not governed). Paths matched by `exempt_paths` in `.arbiter/config.yaml` are allowed and logged to `.arbiter/exempt.log`.

> **MVP note:** The `Bash` tool is not intercepted — only the first-class file-edit tools (`Write`, `Edit`, `MultiEdit`) are gated.

---

## Exposed MCP Tools (12 total)

| Tool | Purpose |
|---|---|
| `record` | Create a new spec or ADR |
| `critique` | Run AI critique → open issues in sidecar |
| `approve` | Disposition issues + sign off → stamp content hash |
| `status` | Report status; flags `needs_review` / `tampered` without editing the files |
| `supersede` | Mark one ADR superseded by another |
| `begin_implementation` | Arm the gate for a specific spec |
| `end_implementation` | Disarm the gate |
| `check_gate` | Query whether a list of paths is currently allowed |
| `index` | Regenerate `INDEX.md` |
| `publish` | Push an approved doc to the Nexus sink (no-op if not configured) |
| `promote_external` | Record an external document (inline, canonical spec format) as a governed draft SPEC/ADR, keeping its provenance in frontmatter |
| `guide` | Return the tier and operating guidance for a document type (`ADR`, `DESIGN`, `RFC`, `PRD`, ...) |

---

## Quickstart (MCP flow)

```
1. record       → Claude calls `record("spec", "Playlist Self-Update")`
                  Returns a spec ID, e.g. "SPEC-playlist-self-update"

2. critique     → Claude calls `critique("SPEC-playlist-self-update")`
                  AI reviews the body and opens issues (e.g. I-001: missing-invariant)

3. fix + disposition
                → Human edits the spec body to address each issue.
                  Then calls `approve` with dispositions:
                  [{"issue_id": "I-001", "disposition": "accepted"}]

4. approve      → Claude calls `approve("SPEC-playlist-self-update", dispositions, "reviewer")`
                  Verifies body was modified, stamps content hash, sets status=approved.

5. begin_implementation
                → Claude calls `begin_implementation("SPEC-playlist-self-update")`
                  Arms the gate. Subsequent Write/Edit calls on src/ paths are now allowed.

6. end_implementation
                → Claude calls `end_implementation()` when coding is complete.
```

---

## Nexus Integration (optional)

`publish` delivers an approved doc to Nexus over **A2A** (the `ingest_governed_doc` skill), with
the doc's content hash riding along as provenance. Add to `.arbiter/config.yaml`:

```yaml
nexus:
  url: "https://your-nexus-instance"   # Nexus base URL (A2A card is discovered under it)
  token: "<write-capability token>"     # or set ARBITER_NEXUS_TOKEN
```

The write token can also come from the `ARBITER_NEXUS_TOKEN` environment variable, which takes precedence over the `token` config key.
Then call the `publish` tool. Ingest is **capability-gated** server-side: the token must carry
Nexus's `ingest_governed` write capability — a read-only token is denied. A denied/failed task
maps gracefully to `{"published": false, "reason": ...}`. Without any config, `publish` is a safe
no-op returning `{"published": false, "reason": "nexus not configured"}`.

> The bespoke HTTP sink was retired; A2A is the sole transport (SPEC-specledger-a2a-publish-phase3 §16).

---

## MVP Boundaries

- **Path-agnostic gate:** paths are normalized relative to `ARBITER_ROOT`; no project-structure assumptions beyond the `docs/**` and `tests/**` allow-globs (configurable via `allow_globs` in config).
- **Bash not gated:** only `Write`, `Edit`, and `MultiEdit` tools are intercepted by the PreToolUse hook.
- **Solo-user:** the approve flow is designed for a single human approver; no multi-reviewer workflow is implemented.
- **No database:** all state lives in Markdown files under `ARBITER_DOCS` (`specs/`, `adr/`, `.reviews/`, `INDEX.md`) and in `.arbiter/` under `ARBITER_ROOT` (`active.json` gate marker, `exempt.log`, optional `config.yaml`).
