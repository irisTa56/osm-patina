# CLAUDE.md

OSM Patina shows a mapper where OpenStreetMap features have gone longest without an edit.
What it covers, what it leaves out, and the order of its phases are in [dev/ROADMAP.md](dev/ROADMAP.md).

## Development documents

Development is steered through the documents the [`dev-docs`](https://github.com/irisTa56/dotfiles/blob/main/.claude/skills/dev-docs/SKILL.md) skill describes, in that skill's default layout under `dev/`.
Follow that skill when starting, working in, or closing a phase, and when recording a decision.

## Keeping a developer's location private

This repository is public, and [a commit pushed to GitHub stays reachable](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) after its branch is rewritten or deleted.
So nothing that reveals where a developer works is committed on any branch or published, whether the point itself or what narrows it down, for example:

- the point a developer works around, as coordinates, a bounding box, or a default value
- the region of the data a developer works from; write "a regional extract" rather than naming it
- a place name near the point
- an image of the map around the point, such as a screenshot kept as evidence that a phase is done
- data cut from the area, such as a test fixture or a sample output

Where a fixture, a screenshot, or an example needs a real area, use one picked for that purpose, unrelated to where any developer works.

Research notes under `dev/research/` may hold such details, which is why [.gitignore](.gitignore) keeps them out of every commit.
Before every push, check that no commit it would publish carries anything of this kind, in its tree or its message, and rewrite those commits first if one does.

## Commands

[mise.toml](mise.toml) is the task list and carries its own reasons.
`mise install` installs the tools and the git hooks, and `mise run pre-commit` is the gate the pre-commit hook runs.

## Git workflow

- Never push to `main` directly; branch first, then open a pull request, which is squash-merged.

## Delegation

- Pass `run_in_background: true` explicitly on every background Agent call, although it is the default.
  - Entire records a subagent's transcript only when that argument is passed ([entireio/cli#2556](https://github.com/entireio/cli/issues/2556)).
  - The rule applies only where the Agent tool has that parameter, which [fork mode](https://code.claude.com/docs/en/sub-agents#turn-fork-mode-on-or-off) removes; in such a session, tell the user that subagents are not recorded.

## Writing conventions

- Everything committed is written in English: code, comments, documents, and commit messages.
- Prose follows [`document-writing.md`](https://github.com/irisTa56/dotfiles/blob/main/.claude/rules/document-writing.md).
