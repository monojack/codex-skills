# Codex Skills

Shared Codex skills that can be installed from GitHub.

## Skills

- `gh-issue-pr-loop`: run an end-to-end GitHub issue implementation workflow that selects an actionable issue, fixes it, opens a pull request, requests Copilot review, and monitors review feedback.
- `codebase-review`: perform a thorough whole-codebase review, consult relevant official framework documentation first, write findings under `reviews/`, and open GitHub issues for each actionable finding.

## Install

Install a skill on another machine with Codex's built-in skill installer.

```sh
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo YOUR_ORG_OR_USER/codex-skills \
  --path skills/gh-issue-pr-loop
```

Or install from a GitHub URL:

```sh
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --url https://github.com/YOUR_ORG_OR_USER/codex-skills/tree/main/skills/gh-issue-pr-loop
```

Replace the path with another skill directory, such as `skills/codebase-review`, to install a different skill from this repository.

Restart Codex after installing.

For a private repository, authenticate first with `gh auth login` or set `GH_TOKEN` / `GITHUB_TOKEN` on the target machine.
