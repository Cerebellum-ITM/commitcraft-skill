---
name: commitcraft
description: Generate a structured git commit message via the CommitCraft CLI (Groq-powered). Use this skill whenever the user asks to commit work the assistant just produced — it stages the relevant files, picks the right tag/scope, runs the multi-stage AI pipeline, reviews the output for AI residue and re-runs the broken stage if needed, then promotes the draft to completed. The user still runs `git commit` themselves with the printed final_message.
---

# CommitCraft skill

This skill drives the `commitcraft` CLI in headless mode (`commitcraft ai …`).
The CLI runs a 3-stage AI pipeline (change-analyzer → commit body → commit
title, plus an optional changelog refiner) against the staged diff and
persists the result as a `draft` row in CommitCraft's local SQLite DB. The
skill's job is to drive that CLI end-to-end without involving the user
beyond the initial intent.

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

### 1. Ensure the right files are staged

The skill is responsible for `git add`-ing the files the assistant just
modified in this session. Check `git status --short`:

- If the files the assistant changed are already in the staged column
  (`M `, `A `, `D ` in column 1), proceed.
- If they're only in the working-tree column (` M`, `??`), `git add` them
  explicitly by path. **Never** run `git add -A` or `git add .` — only
  add what's relevant to the commit being requested. If you're unsure
  which files belong, ask the user.
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

### 8. Hand back

Print to the user:

- The CommitCraft draft id.
- The final commit message (verbatim, in a fenced block so they can copy
  it).
- The exact `git commit` invocation they should run, e.g.:

  ```sh
  git commit -m "<title>" -m "<body>"
  ```

  Or, for a multi-line message, suggest a heredoc. Don't run `git commit`
  yourself.

## What this skill does NOT do

- It does not run `git commit`. The user is the one who actually creates
  the commit.
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

# 5. user runs
git commit -m "..."
```
