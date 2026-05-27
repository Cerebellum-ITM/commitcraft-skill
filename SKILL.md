---
name: commitcraft
description: Generate a structured git commit message via the CommitCraft CLI (Groq-powered) and create the commit. Use this skill whenever the user asks to commit work the assistant just produced — it plans atomic commits per functionality, ensures CHANGELOG.md is updated by the assistant before generation, stages the relevant files, picks the right tag/scope, runs the multi-stage AI pipeline, reviews the output for AI residue and re-runs the broken stage if needed, promotes the draft to completed, and runs `git commit` itself. It never pushes.
---

# CommitCraft skill

This skill drives the `commitcraft` CLI in headless mode (`commitcraft ai …`).
The CLI runs a 3-stage AI pipeline (change-analyzer → commit body → commit
title, plus an optional changelog refiner) against the staged diff and
persists the result as a `draft` row in CommitCraft's local SQLite DB. The
skill's job is to drive that CLI end-to-end and create the actual git
commit, without involving the user beyond the initial intent.

## How I work — read this first

Two non-negotiables that shape how every task ending in a commit must be
planned and executed:

### Atomic commits per functionality, planned up front

Commits in this project are **atomic per functionality**. That decision
is made during planning, not at the end. When laying out work that will
end in a commit (or commits), identify up front the distinct
functionalities / phases — each one is its own commit boundary.

In practice, when a task naturally splits into several pieces (e.g.
"add subcommand X" + "refactor helper Y" + "bump version"), plan to
produce one commitcraft cycle per piece, in order, instead of staging
everything together and producing a single mixed commit. If the task is
truly one piece of work, one commit is correct.

Usually all touched files belong to the same functionality and end up in
the same commit, but unrelated edits (a stray typo fix, an unrelated
config tweak) **must** be split into their own commit — never
piggybacked. Stage only the files that belong to the current
functionality, even when that means leaving other modified files in the
working tree for the next cycle.

### CHANGELOG.md is the assistant's responsibility

If the project has a `CHANGELOG.md`, **the assistant must update it as
part of the code changes for each functionality, before invoking
commitcraft**. Add the entry under the appropriate section (Unreleased /
current version) following the file's existing style.

This is important: if the assistant does not update `CHANGELOG.md`,
commitcraft's changelog stage will invent an entry from the diff, and
that entry is often wrong, duplicated, or stylistically off. The right
flow is: assistant writes the changelog entry → stages it with the rest
of the functionality → commitcraft refines it. Treat `CHANGELOG.md` like
any other source file the change requires.

If the project does not have a `CHANGELOG.md`, skip this — commitcraft
will not invent one.

### Preferred invocation: delegate to a sub-agent

This skill does mechanical work — parse JSON, follow a checklist, run
CLI commands, decide between `ai edit` and `ai regenerate` based on
clear rules. It does **not** need the parent agent's reasoning depth,
and running it inline drags every staged diff and `final_message` into
the parent's context window (which is the same context that just held
the code changes — exactly what we don't want to bloat).

When the parent agent is the one that just produced the code (the
normal case), it should **delegate this skill to a sub-agent** instead
of running it inline. The sub-agent receives the keypoints as part of
its prompt, executes the whole pipeline, makes any corrections needed,
and returns only the commit hash + title to the parent.

```
Agent(
  subagent_type: "general-purpose",
  model: "haiku",
  description: "Commit current changes",
  prompt: """
    Run the commitcraft skill end-to-end for the staged tree.

    Keypoints (use verbatim, do not paraphrase):
    - <keypoint 1>
    - <keypoint 2>
    - ...

    Functionality covered by this commit: <one line>
    Files in scope (already staged): <list>

    If `commitcraft ai context --strict` exits non-zero, stop and
    report — do not proceed with a payload that overflows the model.
    If `final_message` has AI residue, hallucinated paths, or wrong
    language, fix with `ai edit` or `ai regenerate` (cap 2 retries —
    you have full freedom to read the staged diff).

    Report back ONLY: <commit_hash> <title line of final_message>.
  """
)
```

Why Haiku: the work is rule-driven, not creative. Haiku costs ~10×
less per token than Opus and keeps the parent's context clean. If
Haiku gets confused in a corner case (rare with a tight prompt and the
`ai context` gate in place), escalate to Sonnet — still cheaper than
Opus.

**When NOT to delegate**: when the user is driving the skill directly
(invoked `/commitcraft`, asked "commitea esto" in a fresh session, or
otherwise has no parent context to protect), running inline is fine.

## Prerequisites

The skill assumes `commitcraft` is on `$PATH`. If `command -v commitcraft`
fails, stop and tell the user to install it (see `README.md` in this repo).
A `GROQ_API_KEY` must be configured in `~/.config/CommitCraft/.env` — when
the API call fails with an auth error, surface it verbatim and stop.

## When to invoke

- The assistant just made code changes in this session and the user says
  "commit this", "commitea", "haz el commit", "create the commit message",
  or anything that means *produce a commit for what we just did*.
- The user asks for a commit message review of staged changes.

Do **not** invoke this skill when the user explicitly wants to write the
message themselves, when there are no changes to commit, or for unrelated
git operations (push, rebase, etc).

## Workflow

Follow these steps in order. Stop and report back to the user on any
failure — don't paper over errors.

### 1. Confirm the commit boundary and stage the right files

Before staging anything, decide what *this* commit covers. If the work
done in the session spans multiple functionalities, this skill runs
**once per functionality** — pick the first one and only stage its files
now; the rest get their own cycles afterward.

Make sure the `CHANGELOG.md` entry for this functionality has been
written and is part of the files about to be staged (see the "How I
work" section above). If the project has a changelog and the entry is
missing, write it now before continuing.

Then check `git status --short`:

- If the files for this functionality are already in the staged column
  (`M `, `A `, `D ` in column 1) and nothing unrelated is staged, proceed.
- If they're only in the working-tree column (` M`, `??`), `git add` them
  explicitly by path. **Never** run `git add -A` or `git add .` — only
  add what belongs to the current functionality. Unrelated modified
  files stay in the working tree for the next cycle.
- If `git diff --cached --quiet` succeeds (nothing staged), stop and tell
  the user there's nothing to commit.

### 1.5. Pre-flight context check

Before spending any Groq quota, verify the staged diff actually fits
inside the change-analyzer model's context window:

```sh
commitcraft ai context --strict
```

This is offline — no Groq call, no DB write. It rebuilds the exact
payload `ai generate` would send (system prompt + `DEVELOPER_POINTS:` +
the same 80 KB-capped diff) and compares the chars/4 token estimate to
the model's cached `context_window`.

Exit codes and decision tree:

- **Exit 0**, `fits: true`, `diff_truncated: false` → proceed to step 2.
- **Exit 0**, `fits: true`, `diff_truncated: true` → the diff was
  capped by `ChangeAnalyzerMaxDiffSize` before reaching the model.
  Stage 1 will see a partial diff and lean heavily on the keypoints.
  Either split the commit further (preferred — atomic-commits
  principle) or strengthen the keypoints (step 4) so the analyzer
  doesn't extrapolate the missing parts.
- **Exit 3**, `fits: false` → stop. The payload overflows the model
  even before truncation. **Do not proceed** with `ai generate` — that
  is the primary source of hallucinations. Report to the user and
  propose splitting the staged set into smaller atomic commits. Re-run
  this skill from step 1 with the reduced staging.
- **Exit 0**, `fits: null`, `context_window: 0` → the configured model
  (`Prompts.ChangeAnalyzerPromptModel`) is not in the local
  `groq_models_cache`. The gate is advisory only; proceed but mention
  to the user that the context window is unknown. The cache is
  populated by the TUI's model picker — running `commitcraft` once
  refreshes it.
- **Exit 1** with `no_staged_diff` → step 1 was misjudged; re-check
  `git status` and stage.

### 2. Pick the tag

Run `commitcraft ai list-tags` and parse the JSON. The output is an array
of `{tag, description, source}` with `source ∈ {"default", "global", "local"}`,
listing only tags that are eligible for `generate --tag`. Pick the tag
whose `description` best matches the nature of the staged diff:

- New functionality → `ADD`.
- Bug fix → `FIX`.
- Internal refactor with no user-visible change → `REF` or `IMP`.
- Pure docs → `DOC`.
- Etc.

Bias toward `local`-source tags when they exist — those reflect the
project's own taxonomy.

If none of the tags returned by `list-tags` describes the change well,
run `commitcraft ai list-addable-tags` to see the builtin tags the CLI
knows about but that aren't yet registered in the local
`.commitcraft.toml`. The output is an array of `{tag, description}`
(independent of the local config's `behavior`). If one of those addable
tags fits, register it first with:

```sh
commitcraft ai add-tag --tag <TAG>   # repeatable; -t shorthand
```

`add-tag` only accepts tags returned by `list-addable-tags` (or already
local); anything else exits with `invalid_input`. It's idempotent and
creates the local config from template if needed. After a successful
`add-tag`, the tag will appear in `list-tags` with `source: "local"` and
becomes valid for `generate --tag`.

**Never** pass to `--tag` a value that isn't in `list-tags` (after any
`add-tag` you ran).

### 3. Pick the scope

Inspect the staged paths with `git diff --cached --name-only`. The scope
is **a single name** — the folder or file most representative of where
the change lives, in the spirit of Odoo module names. **Never a path with
slashes**: it's one token, no `/`.

Heuristics:

- All files live under a common subtree like `internal/cli/ai/*` → scope
  is the leaf folder name, e.g. `ai`. Pick the deepest folder that
  carries semantic meaning; skip generic wrappers like `src`, `pkg`,
  `lib`, `internal` and use the next-level component.
- All changes are inside a single file at any depth → scope is that
  file's basename without extension (e.g. `README.md` → `README`,
  `database.go` → `database`).
- Multiple unrelated trees touched → pick the leaf folder name of the
  dominant subtree (most files / most lines changed) and mention the
  rest in keypoints. Don't combine multiple names with separators.

Examples (good vs. bad):

| Staged paths                                             | Good       | Bad                  |
| -------------------------------------------------------- | ---------- | -------------------- |
| `internal/cli/ai/{generate,promote,list_tags}.go`        | `ai`       | `internal/cli/ai`    |
| `internal/storage/{queries,types}.go`                    | `storage`  | `internal/storage`   |
| `cmd/cli/main.go`                                        | `main`     | `cmd/cli`            |
| `README.md`                                              | `README`   | `docs`               |
| `internal/tui/{ai_pipeline,commands}.go` (mostly tui)    | `tui`      | `tui,commands`       |

### 4. Build the keypoints

The keypoints are the bridge between the assistant's session memory
and CommitCraft's AI pipeline. They are the **only** way the
non-obvious *intent* behind the diff reaches the model — without them,
stage 1 has to infer everything from the raw diff, which is exactly
where hallucinations originate.

Rules:

1. **Concise but informative.** Short noun phrases or single
   sentences, not paragraphs. In **Spanish**.
2. **Each keypoint must name something concrete** — a symbol,
   filename, flag, decision, or version bump. A keypoint that could
   apply to "any commit in this repo" is useless.
3. **Cover the *why*, not the *what***. The diff already shows what
   changed. Keypoints should add the reasoning, the trade-off picked,
   or the link between pieces that the diff can't show.
4. **3–6 keypoints**. Fewer and the model has nothing to anchor on;
   more and the signal gets diluted.

Good (each names a concrete artifact + decision):

- `"Nuevo subcomando ai promote para marcar drafts como completed"`
- `"Bump v0.35.1 -> v0.36.0 (minor: subcomando user-visible)"`
- `"add-tag valida contra list-addable-tags para evitar tags inventados"`

Bad (vague or pure description of the diff):

- `"Se agregó un nuevo archivo internal/cli/ai/promote.go que
  implementa el subcomando promote para que el usuario pueda marcar un
  draft como completado en la base de datos"` (paragraph, not keypoint)
- `"Cambios en el CLI"` (says nothing)
- `"Refactor"` (says nothing)

Use the assistant's own memory of what it just did — that's the
unique value the agent brings that the diff cannot reproduce.

### 5. Generate

Run:

```sh
commitcraft ai generate \
  -k "<keypoint 1>" -k "<keypoint 2>" … \
  -t <TAG> \
  -s <scope>
```

Capture stdout (a JSON object). Note the `id` and `final_message`
fields. If the command exits non-zero, parse the stderr JSON
`{error,code}` and surface a clear message:

- `no_staged_diff` → step 1 was misjudged; re-check status.
- `invalid_input` with "unknown tag" → step 2 was wrong; re-pick.
- `api_error` → Groq call failed; show the message verbatim.

### 6. Review the output

Step 6 runs in two layers: a **deterministic gate** that catches the
mechanical defects, and a **semantic review** for the judgements the
gate cannot make.

#### 6a. Deterministic gate — `ai verify`

```sh
commitcraft ai verify --id <ID>
```

This is offline, no Groq call, no DB write. The verifier checks the
composed `final_message` against eleven rules and emits a JSON
`VerifyReport`. Exit codes:

- **0** — clean (or warnings only). Move on to 6b.
- **4** — at least one error finding. Each finding has a `rule` slug
  + `severity` + `message` + `location`. Use the rule to decide:
  - `ai_residue_phrase`, `template_placeholder`, `code_fence_wrapper`,
    `title_format_missing_tag`, `empty_title`, `title_equals_body`,
    `title_too_long_hard` → patch with `ai edit` (the right text is
    knowable from the diff + keypoints; no need to re-run the model).
  - Multiple rules failing at once → consider `ai regenerate --stage
    title` or full `regenerate` instead of patching field by field.
- Warnings (`title_format_missing_scope`, `title_too_long_soft`,
  `empty_body`, `duplicate_line_in_body`) don't block the commit by
  default. Decide per case — duplicate `Updated CHANGELOG.md` lines
  warrant an `ai edit --changelog CLEAR` + manual body edit; a 75-char
  title in a body-heavy commit is usually fine.

After any `ai edit` / `regenerate`, re-run `ai verify` to confirm
the gate now passes.

#### 6b. Semantic review — judgements `ai verify` cannot make

Read `final_message` and check for:

- **Truncation**: title cut off mid-word, body referring to bullets
  that don't exist, sentences ending in "...".
- **Hallucinated paths/symbols**: file or function names that aren't
  in the staged diff. Cross-check against `git diff --cached --name-only`.
- **Wrong language**: title in Spanish when the project's body is
  English (or vice versa). The verifier doesn't classify language.
- **Misframed intent**: the title is technically valid but describes
  a different change than the keypoints asked for.

These four are the ones a reviewer's intuition still beats heuristics
on. The `ai verify` gate frees up your attention for exactly these.

Decide whether to **patch** the draft directly or **re-run** the model:

- **Patch directly with `ai edit`** when the text is almost right and
  you already know exactly what should be there: typos, residual AI
  phrasing to strip, a wrong tag/scope, a small tone tweak, or a
  changelog entry that needs to be cleared. No Groq call, no quota
  spend, telemetry from the original run is preserved.
- **Re-run a stage with `ai regenerate --stage`** when you want the
  model to *think again* about that stage: body has hallucinations or
  wrong structure, title doesn't reflect the body, changelog stage
  invented something off. Body re-runs also re-run title and changelog.
- **Full `regenerate`** (no `--stage`) when multiple stages are broken
  or the analyzer summary itself looks wrong.
- **`regenerate --refresh-diff`** when the staged file set changed
  after `generate`, or you see hallucinated version numbers / missing
  files / stale references — those are usually a stale diff snapshot,
  not a bad stage. Mutually exclusive with `--stage`; always implies a
  full pipeline run because only the analyzer consumes the diff.

#### `ai edit` — direct patch

```sh
commitcraft ai edit --id <ID> [--title <s>] [--body <s>] \
                              [--changelog <s>] [--tag <T>] [--scope <s>]
```

At least one flag besides `--id` is required. Each text flag accepts
`-` to read from stdin (a single stdin read is shared if several flags
use `-`). For `--changelog`, the literal `CLEAR` empties the field.
The command recomposes `final_message` from the new title + body and
preserves the previous `Changelog: …` trailer when applicable, then
returns the same JSON shape as `ai show` / `ai regenerate`.

Examples:

```sh
# Strip AI residue from the title
commitcraft ai edit --id 42 --title "FIX: corregir parseo de branches"

# Replace the body from stdin
printf 'Refactor del parser para evitar prefijo `+ `.\n\nDetalle.' \
  | commitcraft ai edit --id 42 --body -

# Fix tag/scope without re-running stages
commitcraft ai edit --id 42 --tag FIX --scope git

# Drop a changelog entry that doesn't apply
commitcraft ai edit --id 42 --changelog CLEAR
```

#### `ai regenerate` — re-run the pipeline

```sh
commitcraft ai regenerate --id <ID> [--stage <body|title|changelog>]
commitcraft ai regenerate --id <ID> --refresh-diff
```

Re-review the new output. Cap at **2** retries — if it's still wrong
after that, stop and surface the latest `final_message` to the user with
a brief note about what's still off; let them decide.

### 7. Promote

When the message passes review, run:

```sh
commitcraft ai promote --id <ID>
```

This flips the draft's status to `completed` in CommitCraft's DB. It
does **not** execute `git commit` — that's intentional.

### 8. Create the git commit

Once the draft is promoted, **run `git commit` yourself** with the
final message. Use a heredoc so the title and body are preserved
verbatim (no escaping pitfalls):

```sh
git commit -m "$(cat <<'EOF'
<final_message verbatim>
EOF
)"
```

If the commit fails (pre-commit hook, etc.), surface the error and stop
— do **not** retry with `--no-verify` and do **not** amend. Fix the
underlying issue, re-stage if needed, and create a new commit.

Do **not** add `Co-Authored-By` trailers or any other automated
signature unless the user explicitly asks for it.

### 9. Hand back and continue

Report to the user using **exactly** this two-line format, nothing else
(no draft id, no extra prose, no bullets, no fences):

```
- Commit: <short_hash> <title line of final_message>
- Resumen: <one-line description of what was committed>
```

The `<title line of final_message>` is the first line of the final
message, including the `[TAG] scope:` prefix (e.g. `[DOC] contract:
document blog composition and lifecycle`). The `Resumen:` line is the
assistant's own one-sentence summary in Spanish — not a copy of the
commit body.

Example:

```
- Commit: f9e2d73 [DOC] contract: document blog composition and lifecycle
- Resumen: Documenta cómo se compone un blog y su ciclo de vida en el contrato.
```

If there are remaining functionalities pending from the original plan
(unstaged changes still in the working tree that belong to the next
commit boundary), continue with the next cycle from step 1. When all
planned commits are done, stop. **Never** push — that's the user's
call.

### Recovering the keypoints after the commit

The keypoints used for a commit are persisted in CommitCraft's local
SQLite (the same row the draft lived in, now `status: "completed"`).
If the user asks "what keypoints were used for commit X?", recover
them with:

```sh
commitcraft ai show --id <draft_id>   # returns JSON with `keypoints` field
commitcraft ai list                    # enumerates recent drafts/commits
```

The `draft_id` is the `id` returned by `ai generate` and printed in
step 5 — it stays valid after `promote`. When invoking via sub-agent,
include the draft id in the report back to the parent if recovery may
be needed later:

```
- Commit: <short_hash> <title>
- Resumen: <one-line summary>
- Draft: <id>     ← optional, only when explicitly requested
```

## Merge commits

When the user asks to **merge a feature branch into main** (or another
target branch), the workflow is different from the staged-diff flow
above but uses the same CommitCraft surface. The schema is fixed by
project convention:

```
[MERGE] <source-branch>: <AI-generated title>

<AI-generated body summarizing the branch>
```

### When to invoke the merge flow

- The user says "merge X into main", "haz el merge", "cierra esta
  rama", or anything that means *merge a feature branch*.
- The current branch is a feature branch and the user wants to land
  it on `main` (or whichever target they name).

Do **not** invoke this for fast-forward-only situations (the merge
commit would be redundant), or when the user wants to rebase instead
of merge.

### Workflow

The pre-flight context gate (step 1.5 in the staged-diff flow) is
**not** applicable here — `ai merge` doesn't feed the diff through
the change-analyzer model; it feeds the commit log through the
release pipeline, which is sized differently.

```sh
# 1. Generate the merge draft from the branch's commit range.
commitcraft ai merge --branch <source> [--into main]

# 2. Verify the composed final_message (same gate, same rule set).
commitcraft ai verify --id <id>

# 3. Patch if needed — most common finding is title_too_long_soft
#    because release-pipeline titles tend to run long.
commitcraft ai edit --id <id> --title "..."

# 4. Promote.
commitcraft ai promote --id <id>

# 5. Execute the actual git merge using the composed final_message.
git checkout <target>            # usually `main`
git merge --no-ff <source> -m "$(commitcraft ai show --id <id> | jq -r .final_message)"
```

### Notes on `ai merge` vs. `ai generate`

- **No `--keypoint` flag**: merge messages derive content from the
  branch's commits, which already encode the keypoints used at
  commit time. If extra steering is needed, use `ai edit` after.
- **`ai regenerate` does NOT yet support merge drafts**. It would
  route the draft through the commit pipeline (wrong). For a clean
  re-run, invoke `ai merge` again from scratch; for tweaks, use `ai edit`.
- **No staging step**. The merge commit's content comes from the
  branch's commit history, not the working tree. The branch can be
  fully clean or have unrelated WIP — neither affects the draft.
- **Title length warnings are common**. The release pipeline's title
  prompt produces marketing-y titles that frequently exceed 72 chars.
  Don't auto-`ai edit` them — sometimes a 90-char title is the right
  call for a substantive branch. Use judgement.

## Release notes

When the user asks to **draft release notes** for a version (typically
right before tagging and publishing on GitHub), use the release flow.
Like the merge flow, it doesn't touch the staged tree or the
change-analyzer model — it summarizes a range of commits through the
same release pipeline.

### When to invoke the release flow

- The user says "draft the release notes", "release v1.2.3", "haz
  el release", "saca las notas de la versión", or anything that means
  *prepare the GH release body text*.
- Typically run right after merging to `main` and before tagging.

Do **not** invoke this for a normal commit, a merge, or a hotfix
where the user just wants a tag without notes.

### Workflow

```sh
# 1. Draft the release notes. --from defaults to the most recent tag,
#    --to defaults to HEAD. --version is required.
commitcraft ai release --version v1.2.3

# 2. Verify (same gate; title_too_long_soft is common — release-pipeline
#    titles run long).
commitcraft ai verify --id <id>

# 3. Trim or rephrase manually if needed. The body often includes
#    every commit in the range; the user may want to drop noisy ones
#    (typo fixes, internal refactors that don't matter externally).
commitcraft ai edit --id <id> --body -    # paste new body via stdin

# 4. Promote.
commitcraft ai promote --id <id>

# 5. Hand the title + body to the user (or to a future
#    `commitcraft ai release publish` once that lands). For now,
#    extracting the body from the JSON:
commitcraft ai show --id <id> | jq -r .body

# 6. The user runs `gh release create` themselves (or you do it
#    with explicit authorization from them).
```

### Notes on `ai release` vs. `ai merge`

- **Required `--version`**. Release notes are versioned by definition;
  if the user hasn't picked a version, push back before running.
- **Default range**: `--from` is the most recent tag, `--to` is HEAD.
  Override either when drafting notes for a non-linear cut (e.g. a
  hotfix branch off an old tag).
- **No `git commit`**. The artifact is GH release-body text, not a
  commit message. The skill stops at promote; the user (or a future
  publish subcommand) drives `gh release create`.
- **Storage divergence from TUI**: the TUI's release mode writes to
  a separate `releases` table; `ai release` writes to `commits`
  (with `type=RELEASE`). The two surfaces don't see each other's
  drafts today. If the user expected to see a TUI-drafted release
  in `ai list`, that's why.
- **Publish is intentionally separate**. The skill never runs
  `gh release create` on its own — that's a public, mostly
  irreversible action. The user authorizes it explicitly.

## What this skill does NOT do

- It does not push.
- It does not execute `git merge` or `git tag` on its own initiative —
  the user must explicitly request the operation for the corresponding
  flow to run.
- It does not reword existing commits — for that, use the TUI's reword
  flow (`commitcraft -w <hash>`).
- It does not publish to GitHub. `ai release` drafts notes but never
  calls `gh`; the publish step (`gh release create` + tag push + asset
  upload) is the user's decision, until a separate
  `ai release publish` subcommand lands.

## Cheat sheet

```sh
# 0. ensure right files are staged
git status --short
git add <paths the assistant changed>

# 0.5. pre-flight context gate (offline, no Groq call)
commitcraft ai context --strict                       # exit 3 if fits=false

# 1. enumerate available tags
commitcraft ai list-tags
commitcraft ai list-addable-tags                      # builtin tags not yet in local config
commitcraft ai add-tag --tag <TAG>                    # register an addable tag locally

# 2. generate
commitcraft ai generate -k "..." -k "..." -t <TAG> -s <scope>

# 2.5. deterministic gate on final_message
commitcraft ai verify --id <id>                       # exit 4 if errors

# 3. review → if needed
#    a) small textual fix / wrong tag-scope / clear changelog: patch directly
commitcraft ai edit --id <id> --title "..."           # or --body, --changelog, --tag, --scope
#    b) make the model think again about a stage:
commitcraft ai regenerate --id <id> --stage <body|title|changelog>
#    c) staged file set changed after step 2:
commitcraft ai regenerate --id <id> --refresh-diff

# 4. promote
commitcraft ai promote --id <id>

# 5. assistant runs
git commit -m "$(cat <<'EOF'
<final_message>
EOF
)"

# 6. if more functionalities remain, loop back to step 0 with the next subset

# 7. recover keypoints after the fact (if needed)
commitcraft ai show --id <draft_id>                   # JSON with keypoints[]
commitcraft ai list                                   # recent drafts/commits

# 8. merge a feature branch (different flow — see "Merge commits" section)
commitcraft ai merge --branch <source> --into main    # generates [MERGE] draft
commitcraft ai verify --id <id>                       # same gate
commitcraft ai edit --id <id> --title "..."           # if title is too long
commitcraft ai promote --id <id>
git checkout main
git merge --no-ff <source> -m "$(commitcraft ai show --id <id> | jq -r .final_message)"

# 9. draft release notes for a version (see "Release notes" section)
commitcraft ai release --version v1.2.3               # defaults: --from=last-tag --to=HEAD
commitcraft ai verify --id <id>
commitcraft ai edit --id <id> --body -                # trim verbose body via stdin
commitcraft ai promote --id <id>
commitcraft ai show --id <id> | jq -r .body           # hand off body text to user
```
