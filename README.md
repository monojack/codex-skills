# Codex Skills

Shared Codex skills that can be installed from GitHub.

## Skills

- `gh-workflow-loop`: run a unified GitHub workflow from either starting point: implement an issue through a ready PR, or handle an existing PR's review feedback with pre-push rechecks and heartbeat monitoring.
- `gh-issue-pr-loop`: run an end-to-end GitHub issue implementation workflow that selects an actionable issue, fixes it, opens a pull request, requests Copilot review, and monitors review feedback.
- `gh-pr-loop`: run the review-monitoring and feedback-addressing loop for an existing pull request when the PR number is known; an issue number may be provided but is not required.
- `codebase-review`: perform a thorough whole-codebase review, consult relevant official framework documentation first, write findings under `reviews/`, and open GitHub issues for each actionable finding.

## Install

Install a skill on another machine with Codex's built-in skill installer.

```sh
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo YOUR_ORG_OR_USER/codex-skills \
  --path skills/gh-workflow-loop
```

Or install from a GitHub URL:

```sh
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --url https://github.com/YOUR_ORG_OR_USER/codex-skills/tree/main/skills/gh-workflow-loop
```

Replace the path with another skill directory, such as `skills/gh-pr-loop`, `skills/gh-issue-pr-loop`, or `skills/codebase-review`, to install a different skill from this repository.

## Symlink For Local Development

To test skills from a local checkout without reinstalling after each edit, symlink a skill directory into Codex's user skills directory:

```sh
mkdir -p ~/.codex/skills
ln -s "$(pwd)/skills/gh-workflow-loop" ~/.codex/skills/gh-workflow-loop
```

If the target path already exists, move or remove the old installed copy first so the symlink points directly at this checkout.

To symlink every skill in this repository without overwriting existing installs:

```sh
mkdir -p ~/.codex/skills
for skill in skills/*; do
  name="$(basename "$skill")"
  target="$HOME/.codex/skills/$name"

  if [ -e "$target" ] || [ -L "$target" ]; then
    echo "Skipping $name: $target already exists"
  else
    ln -s "$(pwd)/$skill" "$target"
  fi
done
```

Restart Codex after installing or changing skill symlinks.

For a private repository, authenticate first with `gh auth login` or set `GH_TOKEN` / `GITHUB_TOKEN` on the target machine.
