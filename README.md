# commitcraft-skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that drives [CommitCraft](https://github.com/) in
headless mode to produce structured `[TAG] scope: title` commit messages
from the changes the assistant just made in a session.

The skill stages the relevant files, picks a tag and scope, writes the
message (or has the Groq pipeline write it), verifies it, promotes the
draft to `completed`, and creates the git commit. It never pushes.

## Installation

### 1. Install the `commitcraft` binary

The skill assumes `commitcraft` is on your `$PATH`. From a clone of the
CommitCraft repo:

```sh
cd path/to/CommitCraft_v2
go install ./cmd/cli
```

This puts `commitcraft` in `$GOBIN` (defaults to `$GOPATH/bin`, which
should already be in your `PATH` if you use Go).

Verify:

```sh
command -v commitcraft
commitcraft ai --help
```

### 2. Configure your Groq API key

CommitCraft reads `GROQ_API_KEY` from `~/.config/CommitCraft/.env`. Create
the file if it doesn't exist:

```sh
mkdir -p ~/.config/CommitCraft
printf 'GROQ_API_KEY=gsk_xxx\n' > ~/.config/CommitCraft/.env
```

The first run of `commitcraft` (TUI or any `ai` subcommand) writes a
default `~/.config/CommitCraft/config.toml` you can later customize.

### 3. Install the skill

Either symlink this repo into your Claude Code skills directory:

```sh
ln -s "$PWD" ~/.claude/skills/commitcraft
```

…or copy the `SKILL.md` (and this whole directory) into
`~/.claude/skills/commitcraft/`.

After that, Claude Code will pick the skill up automatically. Trigger
it by asking for a commit:

```
commitea estos cambios
```

```
generate the commit message for this work
```

## What the skill does

See [`SKILL.md`](./SKILL.md) for the full workflow. Short version, in
the default **delegate mode** (`[agent] mode = "delegate"` in
`~/.config/CommitCraft/config.toml`):

1. Stages the files the assistant just modified (`git add` by path,
   never `-A`).
2. Reads `commitcraft ai list-tags`, picks a tag; deduces the scope from
   the staged paths.
3. Composes 3–6 concise keypoints in Spanish from the work done in the
   session.
4. Runs `commitcraft ai generate`, which returns a prompt bundle instead
   of calling Groq.
5. Writes the English title and body itself, following the bundle's
   prompt, and persists them with `commitcraft ai submit`.
6. Reads the embedded `verify` report and patches with `ai edit` if
   needed.
7. Promotes the draft, runs `git commit`, and links the draft to the
   new hash with `ai link-commit`.

With `mode = "groq"` the same flow runs the Groq pipeline instead of
step 5; that path is documented as an appendix in `SKILL.md`.

## What it does NOT do

- It never pushes.
- It never merges, tags, or publishes a GitHub release on its own
  initiative.
- It doesn't reword existing commits (use the CommitCraft TUI's
  `-w <hash>` flow for that).

## Required CommitCraft version

`v0.70.0` or newer: delegate bundles carry the pending draft `id`, and
`ai verify` reports the style rules the skill relies on
(`title_text_too_long`, `title_restates_tag_verb`, `body_line_too_long`).

## License

MIT — see [`LICENSE`](./LICENSE).
