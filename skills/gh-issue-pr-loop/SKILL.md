---
name: gh-issue-pr-loop
description: "Use when Codex should run an end-to-end GitHub issue implementation loop: search GitHub issues, select an actionable issue, implement the fix, validate it, commit with a Conventional Commit, push a branch, open a ready-for-review pull request, request GitHub Copilot review, and monitor/address PR review comments with capped 5-minute heartbeat checks."
---

# GitHub Issue PR Loop

## Overview

Use this skill to turn a short request into a complete GitHub issue-to-PR workflow with review monitoring. Favor the GitHub connector for repository, issue, pull request, review, comment, and reviewer-request operations; use narrow shell commands for local git, validation, and fallback reviewer requests when connector operations are unavailable or cannot be verified.

## Start Conditions

Resolve the target repository from the user request or the local git remote. If multiple repositories or issue queues are plausible and choosing incorrectly would create work in the wrong place, ask one concise clarification.

Before implementation, sync the base branch according to the repository instructions. If the repository requires starting from `main`, switch to `main`, run a fast-forward-only pull, and stop on any pull failure instead of creating a branch from stale state.

## Workflow Autonomy

Once this skill is invoked and the target repository is clear, follow the workflow end to end without asking for extra "go-ahead" confirmations between normal steps. Do not pause to ask whether to select an issue, create a branch, implement the fix, add tests, commit, push, open the PR, request review, or start monitoring when those actions are part of this skill's stated workflow.

Ask the user only when a required decision cannot be inferred safely, the repository or issue target is ambiguous, credentials or external access are missing, the selected issue cannot be fully resolved, or proceeding would risk destructive or unrelated changes.

## Issue Selection

1. Use the GitHub connector `search_issues` tool first.
2. Search for open, unassigned issues that are not pull requests and appear actionable. Prefer issues with labels such as `bug`, `enhancement`, `hardening`, `refactor`, `documentation`, `change-request`, `help wanted`, or repository-specific implementation labels.
3. Do not select issues that already have any assignee.
4. Do not select issues with an open pull request that references, links, or claims to close/fix/resolve the issue. Verify candidate issue metadata, linked pull requests, and timeline or cross-reference context when search results are ambiguous.
5. Avoid issues that require unclear product decisions, missing credentials, broad redesigns, external stakeholder input, or unsafe production changes.
6. Pick one issue that can be fully completed in the current repository with focused changes and clear validation.
7. State the selected issue briefly before starting work.

Do not select an issue if only part of it can reasonably be addressed. Every concrete requirement, acceptance criterion, bug, regression, and documented subtask in the selected issue must be resolved before opening the PR; never intentionally leave an issue partially fixed.

## Issue Claiming

After selecting an issue and before any branch creation, code inspection, or implementation work, assign the issue to the authenticated GitHub user running the workflow. Prefer the GitHub connector for assignment; use `gh` only when connector support is missing.

If assignment fails, the issue becomes assigned to someone else, or an open PR appears for the issue before implementation starts, stop and select a different eligible issue or ask the user how to proceed.

## Implementation Workflow

1. Proceed only after Issue Claiming succeeds, then create a branch using the repository's branch convention; default to `codex/issue-<number>-<slug>`.
2. Inspect the codebase before editing. For Next.js work, read the relevant local Next.js docs before coding when the repository requires it.
3. Implement the root-cause fix. Do not add workarounds that hide product, test, cache, or environment problems.
4. Address the whole issue, not just the easiest or first visible symptom. If the issue contains multiple requested changes, complete all of them or stop and explain the blocker before creating a PR.
5. Add or update tests when the change affects behavior, fixes a bug, or prevents regression.
6. Run the narrowest sufficient validation command, then broaden validation when shared behavior or user-facing flows are touched.
7. If validation requires dependency install, network access, services, port binding, or other escalation, request escalation and run the command. Do not simulate validation results.

## Commit And PR

1. Review the diff and ensure unrelated user changes are not reverted.
2. Check repository instructions for commit message and PR title conventions, such as CONTRIBUTING files, PR templates, or project docs.
3. Stage only the intended files.
4. Commit with the project's required message convention. If no project instructions exist, use Conventional Commit messages, using a scope when it is clear. Use a single commit for small cohesive fixes, or split the work into multiple focused commits when that makes the issue easier to review.
5. Before opening the PR, confirm the branch resolves the entire selected issue. Do not open a PR that knowingly leaves issue requirements unimplemented.
6. Push the branch.
7. Open a ready-for-review pull request, not a draft. Set the PR title according to the same project commit-message convention so a squash-and-merge commit subject is already correct; if no project instructions exist, use Conventional Commit format. Include a concise body with:
   - selected issue reference
   - summary of changes
   - validation performed
8. Request human reviewers when the issue, repository metadata, CODEOWNERS, or previous reviewer context clearly indicates them.

## Copilot Review

Request GitHub Copilot review through the GitHub connector first when the connector exposes reviewer-request support for the current environment. Verify the connector request before treating it as successful. Use the GitHub CLI as the fallback path because GitHub documents Copilot PR review with `gh` reviewer syntax.

- When creating a PR through the connector, request Copilot review through the connector after the PR exists, then verify the request.
- For an existing PR, request Copilot review through the connector first when available, then verify the request.
- After every Copilot review request or re-review request, verify the PR's reviewer state with a readback, such as `gh pr view <PR-NUMBER> --json reviewRequests,reviews` or the GitHub connector's pull request fields.
- Treat `requested_reviewers: null`, empty review request fields, or missing Copilot reviewer state as unverified, not successful. In that case, fall back to `gh pr edit <PR-NUMBER> --add-reviewer @copilot`, then read the PR state again.
- If the request still cannot be verified, report it as an attempted Copilot review request and explain the observed readback. Do not claim Copilot review or re-review was requested unless the request is verified or the `gh` command succeeds and GitHub reports no additional reviewer state because Copilot has already reviewed the current head.
- If the connector is unavailable, blocked, or does not support Copilot reviewer requests in the current environment, use `gh pr edit <PR-NUMBER> --add-reviewer @copilot` as the fallback and verify the request succeeded before reporting it as requested.

Do not use `@github-copilot` or a PR comment mention as a substitute for Copilot code review. A comment mention may trigger a different Copilot agent behavior instead of the PR review flow.

## Heartbeat Monitoring

After the PR is open and Copilot review is requested, create a new Codex heartbeat automation attached to the current thread for monitoring cycle 1. The heartbeat must run every 5 minutes and be capped at 3 monitor checks for that cycle.

The heartbeat prompt must include:

- repository full name
- PR number and URL
- branch name
- initial requested reviewers, including Copilot when requested
- current monitoring cycle number, starting at 1
- instruction that the 3-check cap applies only to this cycle and resets when a later cycle creates a new heartbeat
- instruction to inspect reviews, review threads, and PR timeline comments via the GitHub connector
- instruction to track review thread IDs for actionable comments so addressed threads can be replied to and resolved
- instruction to count completed checks for the current cycle from thread history
- instruction to stop/delete the heartbeat when monitoring is complete or before making code changes

Use a heartbeat, not a detached cron job, when continuing this thread. Before creating a heartbeat for a cycle, stop/delete any existing matching heartbeat for the same PR. Do not update an existing heartbeat in place for a new cycle; each monitoring cycle must have a newly created heartbeat with a fresh 3-check budget.

## Monitor Check Logic

On each heartbeat run:

1. Fetch pull request reviews, review threads, and PR comments with the GitHub connector.
2. Classify each new item as actionable, non-actionable, duplicate, already addressed, or blocked.
3. Treat comments as actionable only when they request a concrete code, test, behavior, security, performance, accessibility, documentation, or release-note change.
4. Treat praise, status updates, vague preferences, duplicate Copilot repeats, already-addressed suggestions, and questions answered by the PR body as non-actionable.
5. If Copilot says the PR is fine, leaves no comments, or only leaves non-actionable comments, count the check as clean.
6. Count only checks from the current monitoring cycle toward the 3-check cap; never count checks from prior cycles.
7. If 3 checks complete in the current cycle with no actionable comments, stop/delete the current heartbeat and report monitoring complete for that cycle.
8. If actionable comments exist, stop/delete the current heartbeat before editing code and address them in the active workspace.
9. Preserve the review thread or comment identifiers for every actionable item so follow-up replies and thread resolution target the correct discussion.

## Addressing Review Feedback

When actionable feedback exists:

1. Summarize the actionable findings and the files likely affected.
2. Implement reasonable changes directly. If a comment conflicts with project requirements or would degrade the solution, explain why it was not applied and reply on the PR when appropriate.
3. Run validation relevant to the changes.
4. Commit with a Conventional Commit message.
5. Push to the same PR branch.
6. Reply to each addressed actionable review comment or thread with a concise note describing the fix, validation, or reason the change was intentionally not applied.
7. Mark each addressed review thread as resolved after the fix is pushed. Prefer the GitHub connector when it exposes thread resolution; otherwise use the narrowest available GitHub GraphQL mutation, such as `resolveReviewThread`, against the captured review thread ID.
8. Do not resolve blocked, disputed, duplicate, or intentionally-unapplied comments unless the reviewer explicitly accepts the explanation or the thread is otherwise clearly resolved.
9. Request re-review from the initial actionable reviewers. Request Copilot re-review through the connector first when Copilot provided actionable comments or when Copilot was part of the initial review loop; fall back to `gh pr edit <PR-NUMBER> --add-reviewer @copilot` only if the connector path is unavailable, blocked, or cannot be verified. Verify the re-review request using the Copilot Review verification rules before restarting monitoring or reporting success.
10. Stop/delete the current heartbeat if it still exists, then create a new 5-minute heartbeat for the next monitoring cycle with the cycle number incremented and its 3-check budget reset to zero. Do not reuse or update the previous cycle's heartbeat.

Repeat the monitor/address/re-review cycle until Copilot indicates the PR is fine, no actionable comments remain after the capped checks, the PR is merged/closed, or a blocker requires user input.

## Completion Report

When the workflow completes, report:

- issue selected
- branch name
- commit or commits created
- PR URL
- reviewers requested, separating verified reviewer requests from attempts that could not be verified
- validation commands and outcomes
- monitoring result, including whether it ended cleanly, addressed review feedback, replied to addressed comments, resolved review threads, recreated the heartbeat for a new cycle, or stopped on a blocker
