# commitcraft-skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that drives [CommitCraft](https://github.com/) in
headless mode to produce structured `[TAG] scope: title` commit messages
from the changes the assistant just made in a session.

The skill stages the relevant files, picks a tag and scope, generates a
message via the multi-stage AI pipeline, reviews the output for AI
residue, regenerates the broken stage if necessary, and finally promotes
the draft to `completed`. The user runs `git commit` themselves with the
printed message.

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

See [`SKILL.md`](./SKILL.md) for the full workflow. Short version:

1. Stages the files the assistant just modified (`git add` by path,
   never `-A`).
2. Reads `commitcraft ai list-tags`, picks a tag.
3. Deduces the scope from the staged paths.
4. Composes 3–6 concise keypoints in Spanish from the work done in the
   session.
5. Runs `commitcraft ai generate`.
6. Reviews the result; if the title/body/changelog has AI residue,
   re-runs that stage with `commitcraft ai regenerate --stage …` (max 2
   retries).
7. Promotes the draft (`commitcraft ai promote --id …`).
8. Prints the `final_message` and a `git commit` invocation for the
   user to run.

## What it does NOT do

- It never runs `git commit` — that's intentional. The user owns the
  final commit step.
- It never pushes.
- It doesn't reword existing commits (use the CommitCraft TUI's
  `-w <hash>` flow for that).

## Required CommitCraft version

`v0.36.0` or newer (introduces `ai list-tags` and `ai regenerate
--stage`). Earlier versions only support the full-pipeline regenerate.

## License

MIT — see [`LICENSE`](./LICENSE).
