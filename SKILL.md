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

### 2. Pick the tag

Run `commitcraft ai list-tags` and parse the JSON. The output is an array
of `{tag, description, source}`. Pick the tag whose `description` best
matches the nature of the staged diff:

- New functionality → `ADD`.
- Bug fix → `FIX`.
- Internal refactor with no user-visible change → `REF` or `IMP`.
- Pure docs → `DOC`.
- Etc.

Bias toward `local`-source tags when they exist — those reflect the
project's own taxonomy.

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

The keypoints are the most relevant facts about the change, **concise**
(short noun phrases or single sentences, not paragraphs), in **Spanish**.
Each keypoint should be something a reviewer wouldn't immediately see
from the diff alone — *what* was done at the conceptual level, not the
file-by-file inventory.

Good: `"Subcomando ai promote"`, `"Bump v0.35.1 -> v0.36.0"`.

Bad: `"Se agregó un nuevo archivo internal/cli/ai/promote.go que
implementa el subcomando promote para que el usuario pueda marcar un
draft como completado en la base de datos"`.

Aim for 3–6 keypoints. Use the assistant's own memory of what it just did
in the session — that's what they're for.

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

Read `final_message` and check for:

- AI residue: phrases like "I made the following changes", "Here is the
  commit message", "PARAGRAPH 1", literal stage labels, code-fence wrappers.
- Truncation: title that's clearly cut off mid-word, or a body that
  refers to bullets the body doesn't have.
- Hallucinated paths/symbols: file or function names that aren't in the
  staged diff.
- Wrong language: title in Spanish when the project's body is English
  (or vice versa, depending on the project's convention).
- Mention-line duplication: the same `Updated CHANGELOG.md` line
  appearing twice.

Decide which stage produced the issue:

- Title problem (wrong scope, truncated, doesn't match body) → re-run
  `--stage title`.
- Body problem (residue, hallucinations, wrong tone) → re-run
  `--stage body` (this also re-runs title and changelog).
- Changelog mention/entry issue only → `--stage changelog`.
- Multiple stages broken or summary itself looks bad → full
  `regenerate` (no `--stage`).

Run:

```sh
commitcraft ai regenerate --id <ID> [--stage <body|title|changelog>]
```

If you need to re-include files that were staged **after** the original
`generate` (or remove ones that were unstaged), pass `--refresh-diff`
instead of `--stage`. That re-reads `git diff --cached` from the
commit's workspace and persists the new snapshot before the pipeline
runs. Use it whenever you hit hallucinated version numbers, missing
files, or stale references — those are usually symptoms of a stale
diff snapshot rather than a bad stage:

```sh
commitcraft ai regenerate --id <ID> --refresh-diff
```

`--refresh-diff` and `--stage` are mutually exclusive — only the
change analyzer consumes the diff, so refreshing it always implies a
full pipeline run.

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

Report briefly to the user:

- The CommitCraft draft id and the resulting commit hash (`git rev-parse
  --short HEAD`).
- A one-line summary of what was committed.

If there are remaining functionalities pending from the original plan
(unstaged changes still in the working tree that belong to the next
commit boundary), continue with the next cycle from step 1. When all
planned commits are done, stop. **Never** push — that's the user's
call.

## What this skill does NOT do

- It does not push.
- It does not reword existing commits — for that, use the TUI's reword
  flow (`commitcraft -w <hash>`).
- It does not handle release commits — those have a separate flow in
  CommitCraft's TUI (release mode).

## Cheat sheet

```sh
# 0. ensure right files are staged
git status --short
git add <paths the assistant changed>

# 1. enumerate available tags
commitcraft ai list-tags

# 2. generate
commitcraft ai generate -k "..." -k "..." -t <TAG> -s <scope>

# 3. review → if needed
commitcraft ai regenerate --id <id> --stage <body|title|changelog>
# or, if more files were staged after step 2:
commitcraft ai regenerate --id <id> --refresh-diff

# 4. promote
commitcraft ai promote --id <id>

# 5. assistant runs
git commit -m "$(cat <<'EOF'
<final_message>
EOF
)"

# 6. if more functionalities remain, loop back to step 0 with the next subset
```
