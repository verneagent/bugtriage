# bugtriage — Plan

> GitHub issue triage agent. Reads code, decides verdict, posts analysis comment, assigns owner. **Never talks to Lark. Never edits code. Never spawns fixes.**

## Mission

One sentence: **for every open `bug` issue, post a deep analysis comment with a verdict, root cause, suspected file locations, and an assignee — based on the issue body + comments + repo history alone.**

## In scope

- Poll a configured GitHub repo for `is:issue is:open label:bug label:bugpatrol` (and `enhancement` if enabled). The `bugpatrol` label scope is mandatory — it prevents bugtriage from double-handling issues owned by the legacy `/ghissue triage` skill during cutover. To triage a non-bugpatrol issue, a human must add the `bugpatrol` label first.
- Detect when triage is needed: never triaged before, **or** new human/synced comment arrived after the last triage.
- For each triage candidate:
  - Read issue body + comments (including `LARK_SYNC` ones from bugpatrol — treated as fresh human input).
  - Read source via `Read` / `Grep` / `Glob`. Read git history via `Bash` (`git log`, `git blame`).
  - Optionally read a sibling repo (e.g. `weaver`) for cross-stack triage.
  - Output: a single `<!-- TRIAGE_V1 -->` comment with verdict, root cause, locations, owner, suggested tests.
  - Add `--add-assignee` to the most likely owner from blame.
  - **No label writes.** Triage status is encoded by the presence of the `<!-- TRIAGE_V1 -->` comment, not a label. Fix dispatch is out of scope.

## Out of scope (hard line)

- **No Lark.** No imports from `lark-oapi`. No `lark_bot.py`. No reading or writing the Lark thread.
- **No source edits.** Tool whitelist excludes `Edit`, `Write`, `NotebookEdit`.
- **No Agent / sub-agent spawn.** Tool whitelist excludes `Agent`, `Skill`, `Task`.
- **No PR creation.** No `gh pr create`. Tool whitelist excludes any `gh pr` write.
- **No fix dispatch.** bugtriage does not signal downstream fix agents. Whoever wants to act on a `confirmed_bug` verdict reads the `<!-- TRIAGE_V1 -->` comment directly.

## Detection rule (the single source of "should I run?")

Comments are classified by author + body marker:

| Comment shape | Class |
|---|---|
| Body starts `<!-- TRIAGE_V1` | self (bugtriage) |
| Body starts `<!-- LARK_SYNC` | proxied human (Lark reporter via bugpatrol) |
| Body starts `<!-- LARK_LIFECYCLE` or `<!-- LARK_STATE_V1` | bot housekeeping; ignore for detection |
| Otherwise | direct human |

For each open issue carrying `label:bug`:

```python
triage_comments = [c for c in issue.comments if c.body.startswith("<!-- TRIAGE_V1")]
fresh_input = [c for c in issue.comments
               if not c.body.startswith(("<!-- TRIAGE_V1",
                                          "<!-- LARK_LIFECYCLE",
                                          "<!-- LARK_STATE_V1"))]

if not triage_comments:
    # initial triage; treat full body + all fresh_input as input
    run_triage(mode="initial")
elif fresh_input and fresh_input[-1].created_at > triage_comments[-1].created_at:
    # delta retriage; feed previous verdict + new comments as context
    run_triage(mode="delta", previous=triage_comments[-1])
else:
    # nothing new; skip
    pass
```

**No label writes.** Detection is purely comment-timestamp comparison: presence of `<!-- TRIAGE_V1 -->` (any) plus its timestamp vs. fresh-input comment timestamps. This is robust against any label edits by humans.

## LLM step (single)

One `claude-agent-sdk` call per issue. Fresh client, no resume, hard tool whitelist, JSON output validated by pydantic.

| Step | Tools | Model | Output |
|---|---|---|---|
| `triage_issue` | `Read`, `Grep`, `Glob`, `Bash` (read-only `git log`, `git blame`, `gh issue view`, `gh pr list`) | `claude-opus-4-7` | `TriageResult` |

Tool enforcement is by `allowed_tools` whitelist + system prompt forbidding any write. There is **no** `can_use_tool` permission handler — `permission_mode="bypassPermissions"` for runtime simplicity. All safety lives in the whitelist.

```python
class TriageResult(BaseModel):
    verdict: Literal["confirmed_bug", "spec_gap", "works_as_designed",
                     "duplicate", "needs_more_info"]
    layer: Literal["frontend", "backend", "both", "infra", "unknown"]
    root_cause_summary: str = Field(..., max_length=500)
    locations: list[CodeLocation] = Field(..., max_items=8)
    owner: GitHubOwner | None         # None → no assignee, log a warning
    suggested_tests: list[TestSuggestion]
    related_issues: list[int]
    related_prs: list[int]
    spec_path: str | None             # openspec capability if relevant
    caveats: str = ""

class CodeLocation(BaseModel):
    repo: str                         # "fived" | "weaver" | other configured
    path: str
    line_range: tuple[int, int] | None
    why_suspicious: str

class GitHubOwner(BaseModel):
    login: str
    confidence: Literal["blame", "codeowners", "label", "fallback"]
    rationale: str
```

Renderer turns `TriageResult` into the markdown body of the `<!-- TRIAGE_V1 -->` comment. Comment format is fixed in code, not LLM-controlled.

## Module layout

```
bugtriage/
  pyproject.toml              # python>=3.12, pydantic v2, mypy strict, ruff
  bugtriage/
    __main__.py
    cli.py
    config.py                 # repos, sibling checkouts, models
    sdk.py                    # one-shot Claude Agent SDK turn runner
    schemas.py                # TriageResult, CodeLocation, GitHubOwner, comment markers
    prompts/
      triage_initial.md
      triage_delta.md
    sources/
      gh_poll.py              # list bug issues, fetch comments, classify by marker
    sinks/
      gh.py                   # comment, edit (label / assignee) via gh-as-bot.sh
    pipelines/
      triage_one.py           # full per-issue flow
    render/
      comment.py              # TriageResult → markdown
    daemon.py                 # asyncio poll loop with semaphore
  tests/
    fixtures/                 # recorded GH issues (initial + with delta comment)
    test_detect.py
    test_render.py
    test_triage_one_e2e.py    # stub LLM
```

## CLI

```
bugtriage one <issue_num>             # single issue, force re-run even if no delta
bugtriage backlog [--limit N]         # process every issue that needs (re)triage now
bugtriage watch [--interval 300]      # daemon loop over backlog
bugtriage doctor [--quick] [--json]
```

All commands accept `--config /path/to/bugtriage.toml`.

## State storage

**No state files.** Everything derives from GH on each tick:

- "Has this been triaged?" → `<!-- TRIAGE_V1 -->` comments on the issue.
- "Was the triage stale?" → comment-timestamp comparison.
- "Who should own it?" → live `git blame` lookup at triage time, then `gh issue edit --add-assignee`.
- "Where is the configured repo cloned?" → from `bugtriage.toml`.

Restart-safe by construction.

## Configuration shape

```toml
# ~/.bugtriage/config.toml or --config
[github]
repo = "TheCloverLab/fived"
gh_as_bot_script = "/path/to/fived/scripts/gh-as-bot.sh"
require_labels = ["bugpatrol"]        # ALL must be present (cutover gate)
include_labels = ["bug"]              # ANY suffices (or ["bug", "enhancement"])
exclude_labels = []

[siblings]                            # extra repos to read for cross-stack triage
weaver = "/Users/dinghaozeng/clover/weaver"

[ownership]
team_map = "/path/to/docs/team-map.json"  # for blame author → GH login fallback
codeowners = ".github/CODEOWNERS"

[llm]
model = "claude-opus-4-7"
fallback_model = "claude-sonnet-4-6"   # used if opus rate-limited
max_turns = 10                         # SDK turn cap
```

## Re-triage semantics

When a `<!-- LARK_SYNC -->` or direct human comment arrives after the last triage, bugtriage runs in **delta mode**. The prompt receives:

1. The previous `TRIAGE_V1` content (parsed JSON from the marker, not just the rendered markdown).
2. The new comments since that triage, in chronological order.
3. Instruction: "decide whether the previous verdict still holds. If yes, post a brief 'Triage unchanged' comment referencing the previous one. If no, post a fresh full triage with a leading note about what changed."

The new comment also carries `<!-- TRIAGE_V1 -->`. bugpatrol mirrors the new comment to Lark using the same summarizer it uses for initial triage.

## Owner assignment

Order of preference (first non-empty wins):

1. `git blame` of the most-suspicious location → map author → GH login via `team_map.json`. Confidence: `blame`.
2. CODEOWNERS for the most-suspicious path. Confidence: `codeowners`.
3. Existing `assignee` on the issue (don't overwrite). Confidence: `label`.
4. Per-label default from config (e.g. `infra: @ops-lead`). Confidence: `fallback`.
5. None. Log warning, do not assign, record `owner=None` in TriageResult.

Never assign the reporter as owner.

## Milestones

1. **M1: `bugtriage one <n>`** — single-issue, initial mode only. Validates: schema, prompt, blame lookup, comment render, assignee write.
2. **M2: detection logic** — comment marker parser, "needs triage" predicate, unit tests against fixtures. Standalone, no LLM call.
3. **M3: `bugtriage backlog`** — loops detection over all open bug issues, calls M1 for each. Idempotent.
4. **M4: delta mode** — `triage_delta.md` prompt, parse previous TRIAGE_V1 JSON, render delta comment.
5. **M5: `bugtriage watch`** — daemon. Polling loop with concurrency cap. Signal handling, graceful shutdown.
6. **M6: cross-repo** — read sibling checkouts (weaver). Validate: tool permission stays within configured paths.

## Open questions

- **Bot identity vs bugpatrol**: same GH App or separate? Same simplifies token mgmt; separate gives free attribution (no markers needed for self-detection on GH side). Default same — markers are cheap.
- **Cost ceiling per issue**: triage today costs ~$0.10–0.50 in opus tokens depending on repo size read. Should we add a per-issue cap and downshift to sonnet on overrun? Default no cap initially; revisit after a week of metrics.
- **`needs_more_info` verdict**: current `/ghissue triage` skill never returns this — it always picks a verdict. Should bugtriage be able to defer? Pro: avoids confidently wrong triage. Con: produces an issue stuck in limbo. Default: include in schema but document a strong bias against using it.
- **Spec gap → who acts?**: skill currently posts `Spec gap` and stops. With bugpatrol mirroring, the reporter sees "Triage 完成 — Spec gap" and is stuck. Should we auto-route to a PM-facing channel, or is leaving it to humans the right call? Default: humans, document the gap.
- **`enhancement` triage**: enable from M1 or wait? Default wait until M3 — the bug path is the priority.

## Non-goals reminder (refresh on every PR review)

If a PR for bugtriage introduces any of these, **reject**:

- An import from `lark_oapi` or anywhere in `bugpatrol/`.
- Calling an LLM with `Edit`, `Write`, `Agent`, `Skill`, or `Task` in `allowed_tools`.
- Calling `gh pr create`, `gh pr edit`, `gh pr merge`, `git push`, `git commit`, or `git checkout` (other than read-only `git checkout` of detached commits for blame, which we don't need).
- Spawning sub-agents or worktrees.
- Reading files outside the configured `repo` and `siblings` paths.
- Sending HTTP to anything other than GitHub API + the configured Anthropic endpoint.
