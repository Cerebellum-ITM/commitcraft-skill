---
name: commitcraft
description: Generate a structured git commit message via the CommitCraft CLI and create the commit. Use this skill whenever the user asks to commit work the assistant just produced — it plans atomic commits per functionality, ensures CHANGELOG.md is updated by the assistant before generation, stages the relevant files, picks the right tag/scope, obtains the message (writing it itself in delegate mode, or through the Groq pipeline), verifies it, promotes the draft to completed, and runs `git commit` itself. It never pushes.
---

# CommitCraft skill

This skill drives the `commitcraft` CLI in headless mode (`commitcraft ai …`)
and creates the actual git commit, without involving the user beyond the
initial intent.

There are two ways the message gets written, and the CLI tells you which one
is active by the shape of the `ai generate` response:

- **Delegate mode** (the default on this machine: `[agent] mode = "delegate"`
  in `~/.config/CommitCraft/config.toml`). The CLI does not call Groq. It
  persists a pending draft, hands you a prompt bundle, and **you write the
  title and body** following that prompt, then return them with
  `commitcraft ai submit`. Response shape: `"mode": "delegate"`.
- **Groq mode** (`mode = "groq"`, or `--no-agent`). The CLI runs the 3-stage
  Groq pipeline and returns a finished draft. Response shape: a draft with
  `id` + `status` + `final_message`. This path is documented in
  **Appendix A** and only differs between generate and review.

Everything else (staging, tag, scope, keypoints, verify, promote, `git
commit`, link) is identical.

## How I work — read this first

### Atomic commits per functionality, planned up front

Commits are **atomic per functionality**, and that decision is made during
planning, not at the end. When laying out work that will end in a commit,
identify the distinct functionalities / phases up front; each one is its own
commit boundary and its own commitcraft cycle, in order.

Unrelated edits (a stray typo fix, an unrelated config tweak) **must** be
split into their own commit, never piggybacked. Stage only the files that
belong to the current functionality, even when that leaves other modified
files in the working tree for the next cycle. If the task is truly one piece
of work, one commit is correct.

### CHANGELOG.md is the assistant's responsibility

If the project has a `CHANGELOG.md`, **update it as part of the code changes
for each functionality, before invoking commitcraft**, under the appropriate
section (Unreleased / current version), following the file's existing style.
Stage it with the rest of the functionality. If the project has no
`CHANGELOG.md`, skip this; commitcraft will not invent one.

### The commit message has one author: the prompt

In delegate mode the quality and consistency of the message depend entirely
on how faithfully you follow the bundle's `unified.system` prompt
(`agent_commit.prompt`). That prompt is the style contract. Do not improvise
a different structure because a change "feels" like it deserves bullets, or
a longer title, or a note about how it was tested. Step 6 summarizes the
rules that are violated most often; the prompt is the authority.

### The why travels in the keypoints, or it is lost

A commit message records **why** the change was made; the diff already
shows what changed. The writer of the message (usually a sub-agent) sees
only the diff and the keypoints. It was not in the conversation, so it
cannot know the problem, the business rule or the trade-off unless the
keypoints say so. When they don't, it fills the gap with a walk through the
implementation, or with a fake reason ("X had no way to do Y").

So the agent that did the work owes the writer two things (step 4): the
reason, stated so that someone outside the conversation understands it, and
the decisions with their reasons. The conversation's private vocabulary
(decision ids, plan units, nicknames for things) must be translated before
it leaves, because the writer will copy it literally.

### Preferred invocation: delegate to a sub-agent

When the parent agent is the one that just produced the code (the normal
case), run this skill in a **sub-agent** so the staged diff and the bundle
do not bloat the parent's context. The sub-agent receives the keypoints,
executes the whole pipeline, and returns only the commit hash + title.

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",  # alias for the latest Sonnet
  description: "Commit current changes",
  prompt: """
    Run the commitcraft skill end-to-end for the staged tree.

    Keypoints (pass to `ai generate` verbatim, do not paraphrase):
    - Por qué: <problem / need / rule, understandable outside this chat>
    - Decisión: <choice> porque <reason>
    - <optional fact a reader must know>

    Functionality covered by this commit: <one line>
    Files in scope (already staged): <list>

    If `ai generate` returns a delegate bundle, YOU write the English
    title and body following the bundle's `unified.system` prompt
    exactly, then `ai submit` with the bundle's `id`. The body leads with
    the `Por qué`; it does not walk through the implementation. Read the
    `verify` block; fix warnings that the prompt forbids (line > 72,
    title > 50, title restating the tag verb) with `ai edit` or a
    re-submit. Before promoting, re-read the message as someone who never
    saw this conversation: if a term, id or reference would not make
    sense to them, fix it. Cap 2 retries.

    If the keypoints carry no `Por qué`, do not invent one: stop before
    `ai generate` and report back `MISSING_WHY`.

    Report back ONLY: <commit_hash> <title line of final_message>.
  """
)
```

If the sub-agent answers `MISSING_WHY`, write the `Por qué` keypoint from
your session memory and delegate again. Never let it guess the reason.

**Model choice matters in delegate mode.** The sub-agent is the author of
the commit message, not a script runner. Use **the latest Sonnet** (the
`"sonnet"` alias, never a pinned version) as the floor; Haiku
produces noticeably different bodies (mechanism dumps, bullet-only bodies)
from the same prompt, and mixing authors is the main source of
inconsistency across a repo's history. Only in Groq mode, where the model
writes nothing, is Haiku adequate.

**When NOT to delegate**: when the user is driving the skill directly
(invoked `/commitcraft`, asked "commitea esto" in a fresh session, or
otherwise has no parent context to protect), running inline is fine.

## Prerequisites

`commitcraft` must be on `$PATH`; if `command -v commitcraft` fails, stop and
tell the user to install it (see `README.md`). Delegate mode needs no API
key. Groq mode needs `GROQ_API_KEY` in `~/.config/CommitCraft/.env`; see
Appendix A for key slots and rate limits.

## When to invoke

- The assistant just made code changes in this session and the user says
  "commit this", "commitea", "haz el commit", "create the commit message",
  or anything that means *produce a commit for what we just did*.
- The user asks for a commit message review of staged changes.

Do **not** invoke this skill when the user wants to write the message
themselves, when there are no changes to commit, or for unrelated git
operations (push, rebase, etc).

## Workflow

Follow these steps in order. Stop and report on any failure; don't paper
over errors.

### 1. Confirm the commit boundary and stage the right files

Decide what *this* commit covers. If the session spans multiple
functionalities, this skill runs **once per functionality**: pick the first,
stage only its files, and leave the rest for later cycles. Make sure the
`CHANGELOG.md` entry for this functionality is written and part of the
staged set.

Check `git status --short`:

- Files already in the staged column (`M `, `A `, `D `) and nothing
  unrelated staged: proceed.
- Files only in the working-tree column (` M`, `??`): `git add` them
  explicitly by path. **Never** `git add -A` or `git add .`.
- `git diff --cached --quiet` succeeds (nothing staged): stop and tell the
  user there's nothing to commit.

### 2. Pick the tag

Run `commitcraft ai list-tags` and parse the JSON array of
`{tag, description, source}`. Pick the tag whose description best matches
the staged diff: new functionality → `ADD`, bug fix → `FIX`, internal
refactor → `REF` or `IMP`, pure docs → `DOC`, and so on. Bias toward
`local`-source tags; they reflect the project's own taxonomy.

If nothing fits, `commitcraft ai list-addable-tags` lists builtin tags not
yet registered locally; register one with `commitcraft ai add-tag --tag
<TAG>` (idempotent, creates the local config if needed). **Never** pass a
tag to `--tag` that isn't in `list-tags` after any `add-tag` you ran.

### 3. Pick the scope

Inspect `git diff --cached --name-only`. The scope is **a single name**, the
folder or file most representative of where the change lives, in the spirit
of Odoo module names. Never a path with slashes.

- All files under a common subtree like `internal/cli/ai/*` → the deepest
  folder with semantic meaning, `ai`. Skip generic wrappers (`src`, `pkg`,
  `lib`, `internal`).
- A single file at any depth → its basename without extension
  (`README.md` → `README`, `database.go` → `database`).
- Multiple unrelated trees → the leaf folder of the dominant subtree; mention
  the rest in keypoints. Don't combine names with separators.

| Staged paths                                          | Good      | Bad               |
| ----------------------------------------------------- | --------- | ----------------- |
| `internal/cli/ai/{generate,promote,list_tags}.go`     | `ai`      | `internal/cli/ai` |
| `internal/storage/{queries,types}.go`                 | `storage` | `internal/storage`|
| `cmd/cli/main.go`                                     | `main`    | `cmd/cli`         |
| `README.md`                                           | `README`  | `docs`            |
| `internal/tui/{ai_pipeline,commands}.go` (mostly tui) | `tui`     | `tui,commands`    |

### 4. Build the keypoints

The keypoints carry the **reason** behind the diff, the one thing the diff
cannot show. They are the only bridge between your session memory and the
writer of the message, who was not in the conversation.

Structure, in this order:

1. **`Por qué: …` (required, exactly one).** The problem, need or rule that
   made the change necessary, as a teammate outside this conversation
   would understand it: the symptom and who suffers it (fix), the need and
   who has it (feature), the constraint (business, legal, compatibility),
   the cost removed (refactor). Not "no existía X" or "faltaba X": say who
   needed X and what they had to do without it.
2. **`Decisión: … porque …` (0–2).** Non-obvious choices and their reason:
   X instead of Y, what deliberately does not change, what a caller or a
   database must know. Only the ones a future reader would wonder about.
3. **Plain facts (0–2).** Something the reader must know and the diff does
   not make obvious (compatibility, migration, a version bump with its
   kind).

Rules:

- **Spanish, one idea per keypoint, ≤ ~120 characters.** At most 5 in
  total. The keypoints are Spanish; the commit is always English. Never
  "fix" an English commit back to Spanish because the keypoints were
  Spanish.
- **No implementation list.** Do not enumerate files, functions, fields,
  components or buttons; the diff carries them and the writer reads it.
  Name a symbol only when a decision is about it.
- **Outsider test: translate the conversation's private vocabulary.** The
  writer copies terms literally. Never pass decision ids (`R31`), plan unit
  numbers ("unidad 8"), sections of planning docs (`spec.md §3.1`), names
  coined in the session for things the code or UI calls otherwise, the
  test machine, or "el usuario" meaning the person in this chat. Write
  what each one means instead.
- **No session narrative.** No "suite de 41 tests en verde", "verificado
  contra habitta_dev", "medido en la Mac de pruebas". If a verification
  detail matters to the reader of `git log`, it is a design fact ("el cron
  es noupdate, por eso el .po no lo sobreescribe"), not a test report.

Good:

- `"Por qué: el contrato quedaba firmado por el cliente con una firma interna, porque se distinguía a los firmantes por correo y el correo cambió"`
- `"Decisión: distinguir por la etapa de la invitación, que no cambia"`
- `"Bump v0.35.1 -> v0.36.0 (minor: subcomando visible)"`

Bad:

- `"Comandos Tauri regimes y set_regimes en profile.rs"` (what, not why;
  the diff already says it)
- `"Por qué: no había forma de exportar"` (absence, not a reason: who
  needed to export, and why)
- `"R57: seguimiento de cancelaciones (unidad 31)"` (private ids)
- `"Se agregó un nuevo archivo internal/cli/ai/promote.go que implementa…"`
  (paragraph, and mechanism)
- `"7 tests nuevos, suite de 41 en verde"` (session narrative)

If you cannot state the `Por qué` in one sentence, you are probably
committing two things at once; split the commit.

#### Context-store and planning changes

Changes to a context store or planning docs (`STATE.md`, specs, plans,
decision logs) are not features. When they describe the functionality being
committed, stage them **with** that commit instead of making a separate
`[DOC] context` commit that retells it. Changes that only record the
session's progress go in one commit at session close (`/ctx close`), with a
short message: title plus at most two lines.

### 5. Generate

```sh
commitcraft ai generate -k "<keypoint 1>" -k "<keypoint 2>" … -t <TAG> -s <scope>
```

Read the JSON on stdout.

- `"mode": "delegate"` → a **bundle**. Continue with step 6. The bundle
  carries `id` (a pending draft already persisted), `inputs` (tag, scope,
  keypoints, `changelog_active`), `unified.system` + `unified.user` (or
  `stages[]` under the `staged` strategy), `instructions` and
  `submit_example`.
- A draft with `id` + `status` + `final_message` → Groq mode. Skip to
  **Appendix A** for generation errors and review, then rejoin at step 9.

Non-zero exit: parse the stderr JSON `{error, code}`:

- `no_staged_diff` → step 1 was misjudged; re-check status.
- `invalid_input` with "unknown tag" → step 2 was wrong; re-pick.
- `rate_limited` / `api_error` → Groq mode only; see Appendix A.

### 6. Write the message (delegate mode)

Treat `unified.system` as your system instructions and `unified.user` as
the input (TAG / MODULE / DEVELOPER_POINTS / GIT_CHANGES, plus
CHANGELOG_CONTEXT when `inputs.changelog_active` is true). Under the
`staged` strategy, work through `stages[]` in order (summary → body →
title → optional changelog), feeding each output into the next, and emit
one result.

Follow the prompt in full. These are the rules that real drafts break most
often; they are restated here because the verifier now measures them:

- **English**, title and body, always.
- **The body is the why.** First paragraph: the problem, need or rule from
  the `Por qué` keypoint, as behavior someone notices. Then the change in
  one or two sentences, then decisions with their reasons. Never a walk
  through files, functions or UI elements; name a symbol only when a
  decision needs it.
- **Short.** Self-explanatory change: one or two sentences. Typical: 3–8
  lines. Never more than 15. Plain words, sentences of about 25 words at
  most.
- **Outsider test.** No decision ids, plan units, planning-doc sections,
  session nicknames, test machine, or "the user" meaning the requester.
- **Title text ≤ 50 characters**, imperative with a concrete verb (never a
  noun phrase), literal enough to understand from `git log --oneline`,
  lowercase, no period, and it **does not open with the tag's verb**:
  `[ADD] x: add …`, `[FIX] x: fix …`, `[REM] x: remove …`, `[DOC] x:
  document …` are all wrong.
- **Body hard-wrapped at 72 columns.** Count.
- **No session narrative**: no test counts, no "verified against
  <environment>", no tools run, no "the assistant". A version bump is one
  short final line.
- **No actor, no meta**: no "we", "the developer", "this commit", "no other
  changes".
- **Only identifiers that exist in the diff.** Cross-check against
  `git diff --cached --name-only`.

When `changelog_active` is true, also produce `changelog_entry` (imitating
`FORMAT_SAMPLE`) and a one-line `changelog_mention` containing the token
`CHANGELOG.md`.

> **Never `git commit` straight from the bundle.** The bundle is prompt
> material, not a persisted, promoted draft. The order is submit → verify →
> promote → `git commit` → link, every time.

### 7. Submit

Build the payload with `jq` so multiline bodies survive, copy `tag`,
`scope`, `keypoints` from `inputs` verbatim, and **copy the bundle's
top-level `id`** so the pending draft is filled in place instead of
duplicated:

```sh
jq -n \
  --argjson id <BUNDLE_ID> \
  --arg tag "<inputs.tag>" \
  --arg scope "<inputs.scope>" \
  --arg title "<title>" \
  --arg body "$BODY" \
  --argjson keypoints '<inputs.keypoints as JSON array>' \
  '{kind:"commit", id:$id, tag:$tag, scope:[$scope], keypoints:$keypoints,
    title:$title, body:$body}' \
| commitcraft ai submit
```

For a long body, write the JSON to a file in the scratchpad and use
`commitcraft ai submit --input-file <path>`. Never hand-concatenate JSON
with embedded newlines.

`ai submit` composes `final_message`, runs the verifier, and returns the
standard envelope (`id`, `final_message`, …) with an embedded **`verify`**
block. It exits 0 on a successful persist **even when `verify` found
errors**; the quality signal is the `verify` block, not the exit code.

### 8. Review

#### 8a. Deterministic: the `verify` block

Rules and what to do:

| Rule                          | Severity | Action                                                |
| ----------------------------- | -------- | ----------------------------------------------------- |
| `ai_residue_phrase`, `template_placeholder`, `code_fence_wrapper`, `title_format_missing_tag`, `empty_title`, `title_equals_body`, `title_too_long_hard` | error | Fix with `ai edit`; the right text is knowable from diff + keypoints. |
| `title_text_too_long` (> 50)  | warning  | Shorten unless the extra characters carry real meaning. |
| `title_restates_tag_verb`     | warning  | Rewrite the title; this one is never justified.       |
| `body_line_too_long` (> 72)   | warning  | Re-wrap the body and re-submit or `ai edit --body -`. |
| `generic_title`               | warning  | Add specifics.                                        |
| `title_too_long_soft` (> 72 whole line), `title_format_missing_scope`, `empty_body`, `duplicate_line_in_body` | warning | Judgement call. |

Fix with a re-submit (same `id`, corrected `title`/`body`) or with
`commitcraft ai edit --id <id> --title … / --body - / --tag / --scope /
--changelog CLEAR`. After any fix, re-read the returned `verify` block or run
`commitcraft ai verify --id <id>`.

#### 8b. Semantic: what the verifier cannot judge

Read `final_message` once more for:

- **Hallucinated paths or symbols** not in the staged diff.
- **Wrong language**: any Spanish in title or body → fix to English. Never
  the reverse.
- **Misframed intent**: the title is valid but describes a different change
  than the keypoints asked for.
- **Mechanism dump**: a body that lists what each function does instead of
  what changed for the user.
- **Missing or fake why**: cover the first paragraph and ask "why was this
  needed?". If the answer is only "because X did not exist", or the reason
  shows up last after the implementation, rewrite it.
- **Conversation leaks**: read it as a teammate who never saw the session.
  Any id, unit number, nickname or reference that only makes sense with the
  conversation open gets replaced by what it means, or removed.

Cap at **2** correction rounds. If it's still wrong, stop and surface the
latest `final_message` to the user with a one-line note.

### 9. Promote

```sh
commitcraft ai promote --id <ID>
```

Flips the draft to `completed`. It does **not** run `git commit`.

### 10. Create the git commit

Run `git commit` yourself with the final message, via heredoc so title and
body are preserved verbatim:

```sh
git commit -m "$(cat <<'EOF'
<final_message verbatim>
EOF
)"
```

If the commit fails (pre-commit hook, etc.), surface the error and stop. Do
**not** retry with `--no-verify` and do **not** amend; fix the cause,
re-stage, and create a new commit.

Do **not** add `Co-Authored-By` trailers or any automated signature.

### 11. Link the draft to the commit

```sh
commitcraft ai link-commit --id <draft_id> --hash "$(git rev-parse HEAD)"
```

Best-effort: on failure, warn in one line and do not roll back the commit.
In a sub-agent this is the sub-agent's job, before it reports back.

### 12. Hand back and continue

Report using **exactly** this two-line format, nothing else:

```
- Commit: <short_hash> <title line of final_message>
- Resumen: <one-line description in Spanish of what was committed>
```

Example:

```
- Commit: f9e2d73 [DOC] contract: explain how a blog is composed and evolves
- Resumen: Documenta cómo se compone un blog y su ciclo de vida en el contrato.
```

If functionalities remain from the plan, continue with the next cycle from
step 1. When all planned commits are done, stop. **Never** push.

### Regenerating in delegate mode

`commitcraft ai regenerate --id <id>` (with delegate config, or `--agent`)
returns a bundle with `"action": "regenerate"` and the draft's `id`. Write
the new message and submit with that same `id`; it updates the draft in
place. Under Groq config, see Appendix A for `--stage` and
`--refresh-diff`.

### Recovering the keypoints after the commit

```sh
commitcraft ai show --commit <hash>        # short or full hash; ≥ 4 chars
commitcraft ai show --id <draft_id>        # fallback if unlinked
commitcraft ai list -status completed      # search by title_snippet
```

Old drafts can be linked retroactively with `ai link-commit --id <old>
--hash <git>`.

## Merge commits

When the user asks to **merge a feature branch into main** (or another
target), the schema is fixed by project convention:

```
[MERGE] <source-branch>: <title>

<body summarizing the branch>
```

Invoke this for "merge X into main", "haz el merge", "cierra esta rama".
Do **not** invoke it for fast-forward-only situations or when the user
wants a rebase.

```sh
# 1. Generate. Delegate config returns a bundle with kind:"release" and a
#    pending release id; Groq config returns a finished draft.
commitcraft ai merge --branch <source> [--into main]

# 2. (delegate) Write the ENGLISH title + body from the bundle's prompt,
#    then submit copying type/branch/commit_list from inputs and the id.
jq -n --argjson id <ID> --arg title "..." --arg body "$BODY" \
  --arg cl "<inputs.commit_list>" \
  '{kind:"release", id:$id, type:"MERGE", branch:"<source>",
    title:$title, body:$body, commit_list:$cl}' \
| commitcraft ai submit

# 3. Verify (the submit response embeds it), patch, promote. ALWAYS --kind release.
commitcraft ai verify  --id <id> --kind release
commitcraft ai edit    --id <id> --kind release --title "..."
commitcraft ai promote --id <id> --kind release

# 4. Execute the git merge with the composed message, then link.
git checkout <target>
git merge --no-ff <source> -m "$(commitcraft ai show --id <id> --kind release | jq -r .final_message)"
commitcraft ai link-commit --id <id> --kind release --hash "$(git rev-parse HEAD)"
```

Notes:

- No `--keypoint` flag: the branch's commits already carry the intent. Steer
  with `ai edit` if needed.
- `ai regenerate` does **not** support merge drafts; re-run `ai merge`.
- No staging step; the working tree does not matter.
- Long titles are common and sometimes right for a substantive branch. Use
  judgement on `title_too_long_soft`.
- **Pass `--kind release` to every follow-up call.** Merge and release
  drafts live in the `releases` table and their ids can collide with
  `commits` ids.

## Release notes

When the user asks to **draft release notes** for a version ("release
v1.2.3", "haz el release", "saca las notas de la versión"), use the release
flow. It summarizes a commit range; it does not touch the staged tree.

```sh
commitcraft ai release --version v1.2.3       # --from defaults to last tag, --to to HEAD
# (delegate) write title + body, submit with kind:"release", type:"RELEASE",
#            version, commit_list and the bundle id
commitcraft ai verify  --id <id> --kind release
commitcraft ai edit    --id <id> --kind release --body -
commitcraft ai promote --id <id> --kind release
commitcraft ai show    --id <id> --kind release | jq -r .body
```

`--version` is required; push back if the user hasn't picked one. The skill
stops at promote: the user (or the `release-flow` skill, with explicit
authorization) runs `gh release create`.

## Appendix A: Groq mode

Applies when `ai generate` returns a finished draft instead of a bundle
(`[agent] mode = "groq"`, or `--no-agent`).

### A.1 Pre-flight context check (before step 5)

```sh
commitcraft ai context --strict
```

Offline. Rebuilds the exact change-analyzer payload and compares the
estimate to the model's cached `context_window`.

- Exit 0, `fits: true`, `diff_truncated: false` → proceed.
- Exit 0, `fits: true`, `diff_truncated: true` → the diff was capped;
  split the commit further (preferred) or strengthen the keypoints.
- Exit 3, `fits: false` → **stop**; propose splitting into smaller
  commits. Do not run `generate` on an overflowing payload.
- Exit 0, `fits: null`, `context_window: 0` → model not in the local
  cache; advisory only, proceed and mention it.
- Exit 1 with `no_staged_diff` → re-check staging.

### A.2 Rate limits: swap the key slot, don't hammer

CommitCraft keeps two Groq key slots (`user`, `ai`). On
`{"code": "rate_limited"}` from any model-running call (`generate`,
`regenerate`, `merge`, `release`):

1. `commitcraft ai key swap`, then retry the same command **once**.
   `{"code": "empty_slot"}` means only one key exists; go to 3.
2. If the retry is also `rate_limited`, stop swapping (cap: one swap per
   stuck call).
3. Report which slot(s) are limited and stop. Never switch the stage's
   model to dodge the limit; key-swap is the only sanctioned escape.
   Inspect slots with `commitcraft ai key show`.

### A.3 Review and repair

Run `commitcraft ai verify --id <ID>` (exit 4 on errors) and the semantic
review from step 8b. Then:

- **Patch with `ai edit`** when the text is almost right (typos, residue,
  wrong tag/scope, changelog to clear). No Groq call.
- **`ai regenerate --id <ID> --stage <body|title|changelog>`** to make the
  model think again about one stage. Body re-runs also re-run title and
  changelog.
- **Full `regenerate`** when multiple stages are broken or the analyzer
  summary looks wrong.
- **`regenerate --refresh-diff`** when the staged set changed after
  `generate` or you see stale references. Mutually exclusive with
  `--stage`.

Cap at 2 retries, then rejoin the main flow at step 9.

## What this skill does NOT do

- It does not push.
- It does not run `git merge` or `git tag` on its own initiative.
- It does not reword existing commits (use the TUI: `commitcraft -w <hash>`).
- It does not publish to GitHub.

## Cheat sheet

```sh
# 0. stage only what belongs to this functionality
git status --short
git add <paths>

# 1. tags and scope
commitcraft ai list-tags
commitcraft ai list-addable-tags
commitcraft ai add-tag --tag <TAG>
git diff --cached --name-only

# 2. generate (delegate config → bundle with id; Groq config → finished draft)
commitcraft ai generate -k "Por qué: ..." -k "Decisión: ... porque ..." -t <TAG> -s <scope>

# 3. (delegate) write ENGLISH title+body per bundle.unified.system, then submit with the id
jq -n --argjson id <ID> --arg title "..." --arg body "$BODY" \
  '{kind:"commit", id:$id, tag:"<TAG>", scope:["<scope>"], keypoints:["..."],
    title:$title, body:$body}' | commitcraft ai submit
#    → read .verify; fix via re-submit (same id) or ai edit

# 4. verify / patch
commitcraft ai verify --id <id>
commitcraft ai edit --id <id> --title "..."            # or --body -, --tag, --scope, --changelog CLEAR
commitcraft ai regenerate --id <id>                    # delegate: new bundle, submit with same id

# 5. promote, commit, link
commitcraft ai promote --id <id>
git commit -m "$(cat <<'EOF'
<final_message>
EOF
)"
commitcraft ai link-commit --id <id> --hash "$(git rev-parse HEAD)"

# 6. recover later
commitcraft ai show --commit <hash>

# 7. merge / release (kind=release everywhere)
commitcraft ai merge --branch <source> --into main
commitcraft ai release --version v1.2.3
commitcraft ai verify  --id <id> --kind release
commitcraft ai promote --id <id> --kind release
git merge --no-ff <source> -m "$(commitcraft ai show --id <id> --kind release | jq -r .final_message)"
commitcraft ai link-commit --id <id> --kind release --hash "$(git rev-parse HEAD)"

# Groq mode only
commitcraft ai context --strict                        # exit 3 if fits=false
commitcraft ai key show && commitcraft ai key swap     # on rate_limited, retry once
commitcraft ai regenerate --id <id> --stage <body|title|changelog>
commitcraft ai regenerate --id <id> --refresh-diff
```
