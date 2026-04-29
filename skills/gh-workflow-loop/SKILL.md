---
name: gh-workflow-loop
description: "Use when Codex should run a GitHub workflow from either starting point: an issue-to-PR implementation loop or an existing-PR review loop. In issue mode, select or verify an actionable issue, assign it, implement the complete fix/change/feature, validate, commit using repository conventions with Conventional Commit fallback, push, open a ready PR, request review, and monitor feedback. In PR mode, inspect existing reviews and unaddressed comments before pushing, address actionable feedback, recheck for new feedback before every push, then push, request re-review, and start heartbeat monitoring."
---

# GitHub Workflow Loop

## Overview

Use this skill for GitHub work that should continue through implementation, review requests, and capped PR monitoring. It has two starting points:

- **Issue-to-PR mode**: the user asks to pick up a GitHub issue, fix an issue number, or implement an issue into a pull request.
- **Existing-PR mode**: the user provides a pull request number and asks to handle reviews, comments, re-review, or monitoring.

Favor the GitHub connector for repository, issue, pull request, review, comment, thread, reviewer-request, and optional issue operations. Use narrow shell commands for local git, validation, and fallback reviewer requests when connector operations are unavailable or cannot be verified.

Once the target repository and mode are clear, follow the selected workflow end to end without asking for extra "go-ahead" confirmations between normal selection, assignment, implementation, validation, commit, push, PR, review-request, feedback-fix, and heartbeat steps. Ask only when a required decision cannot be inferred safely, access is missing, the target is ambiguous, the work cannot be fully completed, or proceeding would risk destructive or unrelated changes.

## Mode Selection

Use **Existing-PR mode** when the user prompt includes a PR number, a PR URL, or asks to address review comments on an existing PR. Do not search for issues, choose an issue, create a new implementation branch, or open a new PR in this mode.

Use **Issue-to-PR mode** when the user prompt includes an issue number, asks to pick an actionable issue, or asks to run an end-to-end issue implementation loop. If both an issue and PR are provided, use Existing-PR mode only when the PR is clearly the target; otherwise ask one concise clarification.

## Shared Start Conditions

Resolve the target repository from the user request or the local git remote. If multiple repositories or issue queues are plausible and choosing incorrectly would create work in the wrong place, ask one concise clarification.

Check the working tree before editing. Do not revert or modify unrelated local changes. If unrelated local changes are present, work around them or stop only when they make the requested workflow unsafe.

For commits and PR titles, check repository instructions such as CONTRIBUTING files, PR templates, commitlint config, release docs, or project docs. Always follow the repository's convention when one exists; if none exists, use Conventional Commit format with a scope when it is clear.

## Issue-To-PR Mode

1. Resolve or select the issue. If the user provided an issue number, fetch it and verify it is open, actionable, and belongs to the target repository. If no issue number was provided, use the GitHub connector `search_issues` tool first to find open, unassigned issues that are not pull requests and appear actionable.
2. Prefer issues with labels such as `bug`, `enhancement`, `hardening`, `refactor`, `documentation`, `change-request`, `help wanted`, or repository-specific implementation labels.
3. When selecting from search results, do not select issues that already have any assignee. For a prompt-provided issue, proceed if it is unassigned or already assigned to the authenticated GitHub user; stop or ask before taking over an issue assigned to someone else. Do not select or proceed with issues that require unclear product decisions, depend on missing credentials, require broad redesigns, need external stakeholder input, or involve unsafe production changes.
4. Do not select an issue with an open pull request that references, links, or claims to close/fix/resolve it. Verify candidate issue metadata, linked pull requests, and timeline or cross-reference context when search results are ambiguous.
5. Select only an issue that can be fully completed in the current repository. Every concrete requirement, acceptance criterion, bug, regression, and documented subtask in the selected issue must be resolved before opening the PR.
6. Assign the issue to the authenticated GitHub user before branch creation, code inspection, or implementation. If it is already assigned to the authenticated user, treat it as claimed. Prefer the GitHub connector for assignment; use `gh` only when connector support is missing.
7. If assignment fails, the issue becomes assigned to someone else, or an open PR appears for the issue before implementation starts, stop and select a different eligible issue or ask the user how to proceed.
8. Sync the base branch according to repository instructions. If the repository requires starting from `main`, switch to `main`, run a fast-forward-only pull, and stop on any pull failure instead of creating a branch from stale state.
9. Create a branch using the repository's branch convention; default to `codex/issue-<number>-<slug>`.
10. Inspect the relevant code before editing. For framework-specific work, read relevant local or official docs when repository instructions or the change risk call for it.
11. Implement the root-cause fix/change/feature. Do not add workarounds that hide product, test, cache, or environment problems.
12. Add or update tests when the change affects behavior, fixes a bug, or prevents regression.
13. Run the narrowest sufficient validation command, then broaden validation when shared behavior or user-facing flows are touched. If validation requires dependency install, network access, services, port binding, or other escalation, request escalation and run the command. Do not simulate validation results.
14. Review the diff and ensure unrelated user changes are not reverted.
15. Stage only the intended files and commit using the repository's required convention, falling back to Conventional Commit format when no convention exists.
16. Confirm the branch resolves the entire selected issue. Do not open a PR that knowingly leaves issue requirements unimplemented.
17. Push the branch.
18. Open a ready-for-review PR, not a draft. Set the PR title according to the same project convention so a squash-and-merge commit subject is already correct. Include the selected issue reference, summary of changes, and validation performed.
19. Request human reviewers when the issue, repository metadata, CODEOWNERS, or previous reviewer context clearly indicates them. Request and verify Copilot review unless the user or repository instructions opt out.
20. Create monitoring cycle 1 using the Heartbeat Rules, then perform an immediate monitor check unless the user explicitly asked only to schedule monitoring.

## Existing-PR Mode

1. Fetch the PR metadata before scheduling work. If the user provided an issue number, also fetch the issue metadata and confirm the PR is related through closing keywords, linked issues, body text, timeline cross-references, branch naming, or other clear repository context. If the user did not provide an issue number, use any linked issue context discoverable from the PR but proceed with PR-only monitoring when none is available.
2. If the PR is merged, closed, unrelated to the provided issue, or cannot be accessed, stop and report the blocker.
3. Fetch the current PR review state before creating any heartbeat: reviews, open review threads, unresolved or unaddressed review comments, PR timeline comments, requested reviewers, current head SHA, and Copilot review status.
4. Classify existing review items with the Monitor Check Logic. Preserve thread or comment IDs for every actionable item.
5. Fetch and check out the existing PR branch. Do not create a new PR branch unless the original branch is inaccessible and the user approves a recovery path.
6. If actionable feedback exists, address it with the Addressing Review Feedback rules. Before pushing, run the Pre-Push Review Gate; if new actionable feedback appears, do not push yet.
7. If no actionable feedback exists and no local fixes are needed, request and verify Copilot review if it is missing and no user or repository instruction opts out, then create monitoring cycle 1 using the Heartbeat Rules and perform an immediate monitor check unless the user explicitly asked only to schedule monitoring.

## Pre-Push Review Gate

Use this gate before every push that contains review-feedback fixes in Existing-PR mode or in later review cycles for any PR:

1. After local fixes are validated and the commit or commits are prepared, but before `git push`, fetch the latest PR reviews, review threads, PR comments, timeline comments, and current head SHA.
2. Classify any new or still-unaddressed items with the Monitor Check Logic. Do not count this pre-push gate toward any heartbeat cycle's 3-check cap.
3. If new actionable feedback exists, do not push. Preserve the thread or comment IDs, implement the additional feedback locally, rerun relevant validation, update the prepared commit or create an additional focused commit according to repository convention, and repeat this gate.
4. If the recheck is clean, push the prepared batch to the same PR branch.
5. After pushing, request re-review from the initial actionable reviewers. Request Copilot re-review when Copilot provided actionable comments or was part of the review loop, and verify the request with the Copilot Review rules.
6. Create or recreate the heartbeat for the next monitoring cycle only after the push and verified re-review request.

## Copilot Review

Request GitHub Copilot review through the GitHub connector first when the connector exposes reviewer-request support for the current environment. Verify the connector request before treating it as successful. Use the GitHub CLI as the fallback path because GitHub documents Copilot PR review with `gh` reviewer syntax.

- After every Copilot review request or re-review request, verify the PR's reviewer state with a readback, such as `gh pr view <PR-NUMBER> --json reviewRequests,reviews` or the GitHub connector's pull request fields.
- Treat `requested_reviewers: null`, empty review request fields, or missing Copilot reviewer state as unverified, not successful. In that case, fall back to `gh pr edit <PR-NUMBER> --add-reviewer @copilot`, then read the PR state again.
- If the request still cannot be verified, report it as an attempted Copilot review request and explain the observed readback. Do not claim Copilot review or re-review was requested unless the request is verified or the `gh` command succeeds and GitHub reports no additional reviewer state because Copilot has already reviewed the current head.
- If the connector is unavailable, blocked, or does not support Copilot reviewer requests in the current environment, use `gh pr edit <PR-NUMBER> --add-reviewer @copilot` as the fallback and verify the request succeeded before reporting it as requested.

Do not use `@github-copilot` or a PR comment mention as a substitute for Copilot code review. A comment mention may trigger a different Copilot agent behavior instead of the PR review flow.

## Heartbeat Rules

Use a Codex heartbeat automation attached to the current thread, not a detached cron job. Before creating a heartbeat for a cycle, stop/delete any existing matching heartbeat for the same PR. Do not update an existing heartbeat in place for a new cycle; each monitoring cycle must have a newly created heartbeat with a fresh 3-check budget.

The heartbeat must run every 5 minutes and be capped at 3 monitor checks for the current monitoring cycle. Count only checks from the current monitoring cycle toward the cap; never count checks from prior cycles.

The heartbeat prompt must include:

- repository full name
- issue number and URL when provided, selected, or discovered; otherwise state that no issue context is available
- PR number and URL
- PR branch name and current head SHA
- initial requested reviewers and review status, including whether Copilot review is verified, already complete, missing, or attempted-but-unverified
- current monitoring cycle number
- instruction that the 3-check cap applies only to this cycle and resets when a later cycle creates a new heartbeat
- instruction to inspect reviews, review threads, and PR timeline comments via the GitHub connector
- instruction to request and verify Copilot review if it is missing and no user or repository instruction opts out
- instruction to track review thread IDs for actionable comments so addressed threads can be replied to and resolved
- instruction to count completed checks for the current cycle from thread history
- instruction to stop/delete the heartbeat when monitoring is complete or before making code changes

After the heartbeat is established, perform an immediate monitor check in the active thread unless the user explicitly asked only to schedule monitoring. Count this immediate post-heartbeat check as check 1 for the current cycle.

## Monitor Check Logic

On each monitor check or pre-push review gate:

1. Fetch pull request reviews, review threads, PR comments, and relevant timeline comments with the GitHub connector.
2. Classify each new or still-open item as actionable, non-actionable, duplicate, already addressed, or blocked.
3. Treat comments as actionable only when they request a concrete code, test, behavior, security, performance, accessibility, documentation, or release-note change.
4. Treat praise, status updates, vague preferences, duplicate Copilot repeats, already-addressed suggestions, and questions answered by the PR body as non-actionable.
5. If Copilot says the PR is fine, leaves no comments, or only leaves non-actionable comments, count the check as clean.
6. If 3 checks complete in the current monitoring cycle with no actionable comments, stop/delete the current heartbeat and report monitoring complete for that cycle.
7. If actionable comments exist during monitoring, stop/delete the current heartbeat before editing code and address them in the active workspace.
8. Preserve the review thread or comment identifiers for every actionable item so follow-up replies and thread resolution target the correct discussion.

## Addressing Review Feedback

When actionable feedback exists:

1. Summarize the actionable findings and the files likely affected.
2. Ensure the existing PR branch is checked out and up to date enough to apply the fixes without overwriting unrelated user changes.
3. Inspect the relevant code before editing.
4. Implement reasonable changes directly. If a comment conflicts with project requirements or would degrade the solution, explain why it was not applied and reply on the PR when appropriate.
5. Run validation relevant to the changes.
6. Review the diff, stage only the intended files, and commit using the repository's required convention, falling back to Conventional Commit format when no convention exists.
7. Use the Pre-Push Review Gate before pushing. If the gate finds new actionable feedback, keep the work local, update or add focused commits after fixing it, and repeat the gate.
8. Reply to each addressed actionable review comment or thread with a concise note describing the fix, validation, or reason the change was intentionally not applied.
9. Mark each addressed review thread as resolved after the fix is pushed. Prefer the GitHub connector when it exposes thread resolution; otherwise use the narrowest available GitHub GraphQL mutation, such as `resolveReviewThread`, against the captured review thread ID.
10. Do not resolve blocked, disputed, duplicate, or intentionally-unapplied comments unless the reviewer explicitly accepts the explanation or the thread is otherwise clearly resolved.
11. Recreate monitoring with the Heartbeat Rules after the push and re-review request. If no heartbeat has been created yet because Existing-PR mode started with pre-existing actionable feedback, create monitoring cycle 1 after that feedback is pushed and re-review is requested; otherwise increment the prior cycle number.

Repeat the monitor/address/re-review cycle until Copilot indicates the PR is fine, no actionable comments remain after the capped checks, the PR is merged/closed, or a blocker requires user input.

## Completion Report

When the workflow completes, report:

- mode used and target issue or PR
- branch name
- commit or commits created
- PR URL, when one exists
- reviewers requested, separating verified reviewer requests from attempts that could not be verified
- validation commands and outcomes
- monitoring result, including whether it ended cleanly, addressed review feedback, replied to addressed comments, resolved review threads, recreated the heartbeat for a new cycle, or stopped on a blocker
