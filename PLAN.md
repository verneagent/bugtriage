# bugtriage — Plan

> GitHub issue triage agent. Reads code, decides verdict, posts analysis comment, assigns owner. **Never talks to Lark. Never edits code. Never spawns fixes.**

## Background

### Why this exists

The fived repo (`/Users/dinghaozeng/clover/fived`) ships a triage skill — `/Users/dinghaozeng/clover/fived/.claude/skills/ghissue/SKILL.md` (T1–T6 section) — that runs inside the Claude Code agent CLI. It works, but:

- **Skill execution is a black box**: the skill prompt is markdown invoked via the agent CLI; we cannot unit-test "given this issue, does the prompt land on the right verdict?" without launching the full harness.
- **Tool budget is loose**: the skill is invoked with the user's full tool set. Nothing prevents it from accidentally calling `Edit` or `gh pr create`. We rely on prose instructions ("don't edit code") for safety.
- **Coupled with intake**: the skill lives next to `larkbug` and reads its outputs implicitly. Splitting it out forces the contract (HTML comment markers) to be explicit.

bugtriage is a **standalone Python app** that calls the [Claude Agent SDK](https://docs.anthropic.com/en/api/claude-agent-sdk) directly. Its tool whitelist is enforced at SDK call time, not in prose. It runs against any GH repo that carries the cutover label.

### Why two repos (bugpatrol + bugtriage)

bugpatrol is the user-facing voice (Lark intake, screenshots, video). bugtriage is the engineer-facing analysis (read code, blame, assign). Same project, opposite ends. They never share code or process; their only contract is the GitHub issue itself, mediated by HTML comment markers.

Local layout: `/Users/dinghaozeng/code/verneagent/{nemo,bugpatrol,bugtriage}` — three sibling repos, all under the `verneagent` GitHub org, all public.

### Scope vs. fived's existing pipeline

| Concern | bugtriage | fived `/ghissue triage` (legacy) |
|---|---|---|
| Read issue + comments | yes | yes |
| Read source via Read/Grep/Glob | yes | yes |
| Read git history (`git blame`, `git log`) | yes | yes |
| Cross-stack repo reading (weaver) | yes (M6) | yes |
| Post `<!-- TRIAGE_V1 -->` analysis comment | yes | yes |
| Assign owner via `gh issue edit --add-assignee` | yes | yes |
| Add `triaged` / `lark/needs-fix-agent` labels | **no** | yes |
| Spawn fix agent | no | downstream of skill |
| Talk to Lark | no | no (delegated to bugpatrol) |

The fix-spawn step stays in fived. bugtriage just leaves the comment; humans (or fived's `dispatch_pending_fixes.py` if re-pointed) decide what happens next.

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

## Reference materials

Read these before writing code. Most of bugtriage is a re-shape of an existing skill, not greenfield.

### Reference implementation (Claude Agent SDK usage pattern)

- **`/Users/dinghaozeng/code/verneagent/nemo`** — sibling repo. Read in this order:
  - `nemo/pyproject.toml` — Python version (>=3.14), mypy/ruff config, deps.
  - `nemo/nemo/claude_agent.py` — the canonical `ClaudeSDKClient` + `ClaudeAgentOptions` invocation. bugtriage's `sdk.py` is a simplified copy (one-shot, no resume).
  - `nemo/nemo/turn.py`, `nemo/nemo/sdk_thread.py` — anyio task lifecycle. bugtriage daemon does NOT need this complexity (no resumable sessions); skim only.

### fived skills (the legacy triage being replaced)

- `/Users/dinghaozeng/clover/fived/.claude/skills/ghissue/SKILL.md` — primary reference. Read the **T1–T6 triage section** (lines ~449–680). It documents the exact prompt structure, verdict labels, and ownership rules bugtriage must replicate. The plain-prose rules become bugtriage's `triage_initial.md` prompt + `TriageResult` pydantic schema.
- `/Users/dinghaozeng/clover/fived/.claude/skills/bugpipeline/SKILL.md` — orchestrator context. Mostly informational for bugtriage; useful to understand what `<!-- LARK_SYNC -->` and the `bugpatrol` label mean.

### fived helpers (read-only references; do NOT shell out)

bugtriage may parse logic out of these but should not depend on the fived checkout being present at runtime.

- `/Users/dinghaozeng/clover/fived/scripts/triage_backlog.py` — fived's batch-triage driver. Source of the "iterate over open bug issues, skip already-triaged" pattern that becomes bugtriage's `backlog` command.
- `/Users/dinghaozeng/clover/fived/scripts/gh-as-bot.sh` — sobit-bot token wrapper. Vendor a copy into `bugtriage/scripts/gh-as-bot.sh` for any GH write.
- `/Users/dinghaozeng/clover/fived/docs/team-map.json` (if present) — reference for the blame author → GH login mapping. Path is configurable via `[ownership] team_map`.

### External APIs / SDKs

- **Claude Agent SDK (Python)** — `claude-agent-sdk` on PyPI. Docs: https://docs.anthropic.com/en/api/claude-agent-sdk. Used for the single `triage_issue` step.
- **`gh` CLI** — wrapped through `scripts/gh-as-bot.sh` for writes; plain `gh` for reads.
- **`git`** — read-only operations only (`log`, `blame`, `show`, `diff`). bugtriage never calls `git commit`, `git push`, `git checkout` (other than detached read-only checkouts, which we don't currently need).

### Repos bugtriage reads (configurable)

- `[github] repo` — primary repo, e.g. `TheCloverLab/fived`. Cloned at a fixed local path; bugtriage `cd`s into it for `Read`/`Grep`/`Glob`/`git blame`.
- `[siblings]` — additional checkouts for cross-stack triage, e.g. `weaver = "/Users/dinghaozeng/clover/weaver"`. Tool whitelist still applies; the SDK is told via system prompt that it may read those paths.

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

## Execution playbook

Each step is independently shippable and ends with a concrete verification command. Do not start step N+1 until step N's verification passes against a real GitHub issue.

### M0: Bootstrap (no LLM, no GH writes)

**Deliverable**: empty package that lints, type-checks, and runs `bugtriage --help`.

1. `cd /Users/dinghaozeng/code/verneagent/bugtriage` (already cloned).
2. Write `pyproject.toml` modeled after `nemo/pyproject.toml`:
   - `requires-python = ">=3.12"` (3.14 if available).
   - Deps: `pydantic>=2.7`, `claude-agent-sdk`, `typer` (or `click`), `anyio`, `httpx`.
   - Dev deps: `mypy`, `ruff`, `pytest`, `pytest-asyncio`.
   - `[tool.mypy] strict = true`. `[tool.ruff] target-version = "py312"`.
3. Create the directory tree under `bugtriage/bugtriage/` per [Module layout](#module-layout). Stub each leaf with `# TODO(M<n>)`.
4. `bugtriage/cli.py` — Typer app with `one`, `backlog`, `watch`, `doctor` subcommands. Each prints `NotImplementedError` and exits 2.
5. `bugtriage/__main__.py` — calls `cli.app()`.
6. `bugtriage/config.py` — `Config` BaseModel loaded from TOML via `tomllib`. Schema matches [Configuration shape](#configuration-shape). No defaults for paths/secrets; fail loudly on missing config.
7. Vendor `gh-as-bot.sh`: copy `/Users/dinghaozeng/clover/fived/scripts/gh-as-bot.sh` to `bugtriage/scripts/gh-as-bot.sh`, mark executable.
8. CI: GitHub Actions workflow running `ruff check`, `ruff format --check`, `mypy bugtriage`, `pytest`.

**Verify**:
```
uv run bugtriage --help
ruff check . && mypy bugtriage && pytest -q
```
Both must exit 0. No network, no LLM call.

### M1: `bugtriage one <n>` — single-issue, initial mode

**Deliverable**: given an open issue carrying `bugpatrol` + `bug` labels and no prior `<!-- TRIAGE_V1 -->` comment, run the full triage pipeline and post one analysis comment + assign owner.

Files to write (in order):

1. `schemas.py` — define `TriageResult`, `CodeLocation`, `GitHubOwner`, `TestSuggestion` per the [LLM step](#llm-step-single) section. All pydantic v2 with `Field` constraints.
2. `sources/gh_poll.py` — `fetch_issue(num) -> IssueWithComments`. Wraps `gh issue view --json body,title,labels,assignees,createdAt,closedAt,state` + `gh api repos/.../issues/N/comments`. Returns a typed model.
3. `sinks/gh.py` — `add_comment(issue, body)`, `assign(issue, login)`. All writes go through `scripts/gh-as-bot.sh`.
4. `sdk.py` — one-shot Claude Agent SDK runner. Same shape as `nemo/nemo/claude_agent.py` but simpler: no resume, no `setting_sources`, no `can_use_tool`. `permission_mode="bypassPermissions"`. Hard-fail on missing JSON output. Allowed tools come from a callsite-supplied whitelist; the runner sanity-checks against a hardcoded blocklist (`Edit`, `Write`, `NotebookEdit`, `Agent`, `Skill`, `Task`) and raises if a caller tries to slip them in.
5. `prompts/triage_initial.md` — prompt template. Inputs: issue body, comments (with markers preserved), repo paths the SDK may read. Output instruction: a JSON block matching `TriageResult`. Mirror the T1–T6 logic from `fived/.claude/skills/ghissue/SKILL.md`.
6. `pipelines/triage_one.py`:
   ```python
   async def triage_one(issue_num: int, cfg: Config, *, force: bool = False) -> TriageOutcome:
       issue = await gh.fetch_issue(issue_num)
       if not force and has_triage(issue):
           return TriageOutcome.ALREADY_TRIAGED
       result = await sdk.run(
           prompt=render_prompt("triage_initial", issue=issue, cfg=cfg),
           model=cfg.llm.model,
           allowed_tools=["Read", "Grep", "Glob", "Bash"],
           system_prompt=TRIAGE_SYSTEM_PROMPT,
           response_model=TriageResult,
       )
       comment = render_triage_comment(result)  # marked with <!-- TRIAGE_V1 -->
       await gh.add_comment(issue_num, comment)
       if result.owner:
           await gh.assign(issue_num, result.owner.login)
       return TriageOutcome.TRIAGED(result)
   ```
7. `render/comment.py` — `render_triage_comment(result: TriageResult) -> str`. Fixed format, embeds `<!-- TRIAGE_V1: ts=<iso8601> -->` on first line plus the JSON-encoded `TriageResult` between hidden delimiters so M4's delta mode can parse it back.
8. Owner resolution helper (`pipelines/owner.py`): implements the [Owner assignment](#owner-assignment) preference order. Standalone module, fully unit-tested without LLM.
9. `cli.py` — wire up `bugtriage one <num> [--force]`.

Tests:
- `tests/fixtures/issue_pristine.json` — captured GH issue JSON, no triage comment.
- `tests/test_render.py` — round-trip: `TriageResult` → markdown → parse → equal `TriageResult`.
- `tests/test_owner.py` — preference-order matrix.
- `tests/test_triage_one_e2e.py` — stub `sdk.run`, stub gh sinks, assert one comment + one assign.

**Verify**:
```
bugtriage one --config config.toml <issue_num>
gh issue view <issue_num> --json comments,assignees
```
Issue must have exactly one comment starting `<!-- TRIAGE_V1`. Assignee must be set if `result.owner` is non-null. Re-running without `--force` must exit early with `ALREADY_TRIAGED`.

### M2: detection logic (no LLM)

**Deliverable**: pure-function module that decides whether an issue needs triage and in what mode (initial / delta / skip).

Files:
1. `sources/gh_poll.py` — extend with `classify_comment(body) -> CommentClass` enum (`SELF_TRIAGE`, `LARK_SYNC`, `LARK_LIFECYCLE`, `LARK_STATE`, `HUMAN`).
2. `pipelines/detect.py`:
   ```python
   def needs_triage(issue: IssueWithComments) -> TriageMode | None:
       triage_comments = [c for c in issue.comments if classify_comment(c.body) == CommentClass.SELF_TRIAGE]
       fresh = [c for c in issue.comments if classify_comment(c.body) in (CommentClass.HUMAN, CommentClass.LARK_SYNC)]
       if not triage_comments:
           return TriageMode.INITIAL
       if fresh and fresh[-1].created_at > triage_comments[-1].created_at:
           return TriageMode.DELTA(previous=triage_comments[-1])
       return None
   ```

Tests:
- `tests/fixtures/issue_with_lark_sync.json` — has TRIAGE_V1 then LARK_SYNC.
- `tests/fixtures/issue_with_human_followup.json` — has TRIAGE_V1 then plain comment.
- `tests/fixtures/issue_just_lifecycle.json` — has TRIAGE_V1 then LARK_LIFECYCLE only (must skip).
- `tests/test_detect.py` — exhaustive truth table over the four `CommentClass` values × triage-present/absent.

**Verify**: `pytest tests/test_detect.py -v` — all branches covered.

### M3: `bugtriage backlog`

**Deliverable**: iterate every open issue matching `label:bugpatrol label:bug`, run M2 detection, call M1 (or M4 once it lands) for each.

Files:
1. `sources/gh_poll.py` — `iter_candidate_issues(cfg) -> AsyncIterator[IssueRef]`. Uses `gh issue list -l bugpatrol -l bug --state open --json number`.
2. `pipelines/backlog.py`:
   ```python
   async def run_backlog(cfg: Config, *, limit: int | None = None) -> BacklogReport:
       sem = anyio.Semaphore(cfg.max_concurrent)
       async with anyio.create_task_group() as tg:
           async for ref in gh_poll.iter_candidate_issues(cfg):
               async def handle(ref=ref):
                   async with sem:
                       issue = await gh.fetch_issue(ref.number)
                       if mode := needs_triage(issue):
                           await triage_one(ref.number, cfg, mode=mode)
               tg.start_soon(handle)
   ```
3. `cli.py` — wire up `bugtriage backlog [--limit N]`.

Tests:
- `tests/test_backlog_idempotent.py` — fixture: one issue already triaged, one pristine, one with LARK_SYNC. Stub LLM. Assert exactly two LLM calls (pristine → initial; LARK_SYNC → delta-once-M4-lands; for now, skip).

**Verify**: run twice in quick succession; second run logs "0 triaged" and emits no GH writes.

### M4: delta mode

**Deliverable**: when M2 returns `TriageMode.DELTA`, run a delta prompt that takes the previous `TriageResult` JSON + new comments and decides whether the verdict still holds.

Files:
1. `prompts/triage_delta.md` — receives previous JSON + new comments. Output is again a `TriageResult` (full, fresh) plus a `delta_summary: str` field describing what changed. Prompt instruction: "if the verdict didn't change and the new info is incidental, copy the prior result and set `delta_summary` to a one-line note; otherwise produce a fresh full triage."
2. Extend `schemas.py` with optional `delta_summary: str | None = None` on `TriageResult`.
3. `pipelines/triage_one.py` — branch on `mode`:
   ```python
   if mode is TriageMode.INITIAL:
       prompt = render_prompt("triage_initial", ...)
   elif isinstance(mode, TriageMode.DELTA):
       previous = parse_triage_v1(mode.previous.body)
       prompt = render_prompt("triage_delta", previous=previous, new_comments=...)
   ```
4. `render/comment.py` — when `delta_summary` is set, prepend a "Δ since previous triage:" header.

Tests:
- `tests/test_triage_delta.py` — fixture: issue with prior triage + LARK_SYNC. Stub LLM to return same verdict + `delta_summary`. Assert new comment carries the delta header and embeds fresh `TRIAGE_V1` JSON.

**Verify**: in production, post a `LARK_SYNC` comment to a triaged issue; run `bugtriage one <n>`; observe a second `TRIAGE_V1` comment with a delta header.

### M5: `bugtriage watch` — daemon

**Deliverable**: long-running daemon that polls backlog every `--interval` seconds.

Files:
1. `daemon.py`:
   ```python
   async def run(cfg: Config, *, interval: int):
       async with anyio.create_task_group() as tg:
           tg.start_soon(install_signal_handlers, tg.cancel_scope)
           while True:
               await run_backlog(cfg)
               await anyio.sleep(interval)
   ```
2. Signal handling: SIGINT/SIGTERM → cancel current backlog gracefully, exit 0.
3. Logging: structured JSON to stdout; one line per issue triaged with `{issue, verdict, owner, duration_s}`.

Tests:
- `tests/test_daemon_lifecycle.py` — start, fake one tick, send SIGINT, assert clean shutdown.

**Verify**: `bugtriage watch --interval 60`; create a new pristine bug issue with the cutover labels; observe triage within ≤120s; `kill -TERM <pid>`; observe graceful shutdown.

### M6: cross-repo (weaver, etc.)

**Deliverable**: when triaging an issue whose suspected location is in a sibling repo, the SDK can `Read`/`Grep`/`git blame` that repo's checkout.

Files:
1. `config.py` — `[siblings]` section already loaded; expose `cfg.siblings: dict[str, Path]`.
2. `pipelines/triage_one.py` — when running the SDK, set CWD to the primary repo, but include sibling paths in the system prompt: "You may also Read and Grep within: /Users/.../weaver".
3. `schemas.py` — `CodeLocation.repo` validates against `["fived", "weaver", ...cfg.siblings.keys()]`.

Tests:
- `tests/test_cross_repo_paths.py` — assert that when an LLM result references `repo="weaver"` with a path inside the sibling, the renderer produces a working `https://github.com/.../blob/...` link.

**Verify**: pick an issue whose root cause is known to live in weaver; run `bugtriage one`; observe `TRIAGE_V1` JSON with `locations[].repo == "weaver"` and a sane `path`.

---

**Rule**: do not start step N+1 until step N's verify command passes in production. Do not add cross-cutting "improvements" mid-step. The cost of a fresh session re-deriving "did M1 actually work?" is small; the cost of a half-built M3 is large.

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
