---
name: gh-pr-loop
description: "Use when Codex should run only the existing GitHub pull request review loop for a prompt-provided pull request number and optional issue number: verify the PR context, inspect existing reviews, review threads, and comments before the first heartbeat, address actionable feedback, recheck for new feedback before every push, push only when clean, request re-review, recreate the monitoring heartbeat, and repeat until clean or blocked. This skill does not search for, select, assign, implement from scratch, or open issues or pull requests."
---

# GitHub PR Loop

## Overview

Use this skill for an existing pull request. The user prompt must indicate the PR number; an issue number is optional. Favor the GitHub connector for repository, pull request, review, comment, thread, reviewer-request, and optional issue operations; use narrow shell commands for local git, validation, and fallback reviewer requests when connector operations are unavailable or cannot be verified.

This skill starts at the PR review loop. Do not search for issues, choose an issue, assign an issue, create an implementation branch, implement the issue from scratch, or open a new PR.

## Start Conditions

Resolve the target repository from the user request or local git remote. If the repository or PR number is missing or ambiguous, ask one concise clarification. Do not ask for an issue number solely because it is absent.

Fetch the PR metadata before scheduling work. If the user provided an issue number, also fetch the issue metadata and confirm the PR is related to it through closing keywords, linked issues, body text, timeline cross-references, branch naming, or other clear repository context. If the user did not provide an issue number, use any linked issue context discoverable from the PR but proceed with PR-only monitoring when none is available. If the PR is merged, closed, unrelated to the provided issue, or cannot be accessed, stop and report the blocker.

Before creating the first heartbeat, fetch the current PR review state: reviews, open review threads, unresolved or unaddressed review comments, PR timeline comments, requested reviewers, and Copilot review status. Classify the existing review items with the Monitor Check Logic. If actionable feedback already exists, do not create a heartbeat yet; preserve the relevant thread or comment IDs and go directly to Addressing Review Feedback.

Once the repository and PR are clear, follow the workflow end to end without asking for extra "go-ahead" confirmations between normal monitoring, feedback-fix, commit, push, re-review, and heartbeat steps.

## Triage Before Automation

Only create the first Codex heartbeat automation after the initial PR review-state triage finds no actionable feedback already pending, or after initially pending feedback has been addressed, pushed, replied to, resolved as appropriate, and re-review has been requested. Stop/delete any existing matching heartbeat for the same PR before creating the new heartbeat. Use a heartbeat attached to the current thread, not a detached cron job. Do not update an existing heartbeat in place for a new cycle; each monitoring cycle must have a newly created heartbeat with a fresh 3-check budget.

When the workflow reaches its first clean monitoring phase, create the heartbeat as monitoring cycle 1. If the first PR triage found actionable feedback, the first heartbeat still starts at cycle 1 after that feedback is handled; do not count the pre-heartbeat triage as a monitor check.

The heartbeat must run every 5 minutes and be capped at 3 monitor checks for the current monitoring cycle. The first cycle starts at 1, and the 3-check cap applies only to that cycle.

The heartbeat prompt must include:

- repository full name
- issue number and URL when provided or discovered; otherwise state that no issue context is available
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

## Copilot Review

Request GitHub Copilot review through the GitHub connector first when the connector exposes reviewer-request support for the current environment. Verify the connector request before treating it as successful. Use the GitHub CLI as the fallback path because GitHub documents Copilot PR review with `gh` reviewer syntax.

- For an existing PR, request Copilot review through the connector first when available, then verify the request.
- After every Copilot review request or re-review request, verify the PR's reviewer state with a readback, such as `gh pr view <PR-NUMBER> --json reviewRequests,reviews` or the GitHub connector's pull request fields.
- Treat `requested_reviewers: null`, empty review request fields, or missing Copilot reviewer state as unverified, not successful. In that case, fall back to `gh pr edit <PR-NUMBER> --add-reviewer @copilot`, then read the PR state again.
- If the request still cannot be verified, report it as an attempted Copilot review request and explain the observed readback. Do not claim Copilot review or re-review was requested unless the request is verified or the `gh` command succeeds and GitHub reports no additional reviewer state because Copilot has already reviewed the current head.
- If the connector is unavailable, blocked, or does not support Copilot reviewer requests in the current environment, use `gh pr edit <PR-NUMBER> --add-reviewer @copilot` as the fallback and verify the request succeeded before reporting it as requested.

Do not use `@github-copilot` or a PR comment mention as a substitute for Copilot code review. A comment mention may trigger a different Copilot agent behavior instead of the PR review flow.

## Monitor Check Logic

On each monitor check:

1. Fetch pull request reviews, review threads, and PR comments with the GitHub connector.
2. Classify each new item as actionable, non-actionable, duplicate, already addressed, or blocked.
3. Treat comments as actionable only when they request a concrete code, test, behavior, security, performance, accessibility, documentation, or release-note change.
4. Treat praise, status updates, vague preferences, duplicate Copilot repeats, already-addressed suggestions, and questions answered by the PR body as non-actionable.
5. If Copilot says the PR is fine, leaves no comments, or only leaves non-actionable comments, count the check as clean.
6. Count only checks from the current monitoring cycle toward the 3-check cap; never count checks from prior cycles.
7. If 3 checks complete in the current cycle with no actionable comments, stop/delete the current heartbeat and report monitoring complete for that cycle.
8. If actionable comments exist, stop/delete the current heartbeat before editing code and address them in the active workspace.
9. Preserve the review thread or comment identifiers for every actionable item so follow-up replies and thread resolution target the correct discussion.

## Pre-Push Review Gate

Before every push that contains review-feedback fixes:

1. After local fixes are validated and the commit or commits are prepared, but before `git push`, fetch the latest PR reviews, review threads, PR comments, timeline comments, and current head SHA.
2. Classify any new or still-unaddressed items with the Monitor Check Logic. Do not count this pre-push gate toward any heartbeat cycle's 3-check cap.
3. If new actionable feedback exists, do not push. Preserve the thread or comment IDs, implement the additional feedback locally, rerun relevant validation, update the prepared commit or create an additional focused commit according to repository convention, and repeat this gate.
4. Push to the same PR branch only after the recheck finds no new actionable feedback.

## Addressing Review Feedback

When actionable feedback exists:

1. Summarize the actionable findings and the files likely affected.
2. Fetch and check out the existing PR branch. Do not create a new PR branch unless the original branch is inaccessible and the user approves a recovery path.
3. Inspect the relevant code before editing and preserve unrelated user changes.
4. Implement reasonable changes directly. If a comment conflicts with project requirements or would degrade the solution, explain why it was not applied and reply on the PR when appropriate.
5. Run validation relevant to the changes.
6. Check repository instructions for commit message and PR title conventions, such as CONTRIBUTING files, PR templates, or project docs. Commit follow-up changes with the project's required message convention; if no project instructions exist, use Conventional Commit format.
7. If the existing PR title clearly violates the repository's squash-merge title convention and updating it is safe, update the title to match the same convention.
8. Use the Pre-Push Review Gate before pushing. Do not push while newly fetched actionable feedback remains unaddressed.
9. Reply to each addressed actionable review comment or thread with a concise note describing the fix, validation, or reason the change was intentionally not applied.
10. Mark each addressed review thread as resolved after the fix is pushed. Prefer the GitHub connector when it exposes thread resolution; otherwise use the narrowest available GitHub GraphQL mutation, such as `resolveReviewThread`, against the captured review thread ID.
11. Do not resolve blocked, disputed, duplicate, or intentionally-unapplied comments unless the reviewer explicitly accepts the explanation or the thread is otherwise clearly resolved.
12. Request re-review from the initial actionable reviewers. Request Copilot re-review through the connector first when Copilot provided actionable comments or when Copilot was part of the initial review loop; fall back to `gh pr edit <PR-NUMBER> --add-reviewer @copilot` only if the connector path is unavailable, blocked, or cannot be verified. Verify the re-review request using the Copilot Review verification rules before restarting monitoring or reporting success.
13. Stop/delete the current heartbeat if it still exists, then create a new 5-minute heartbeat for the next monitoring cycle with its 3-check budget reset to zero. If no heartbeat has been created yet because the workflow started with pre-existing actionable feedback, create monitoring cycle 1 after the feedback is handled; otherwise increment the prior cycle number. Do not reuse or update the previous cycle's heartbeat.

Repeat the monitor/address/re-review cycle until Copilot indicates the PR is fine, no actionable comments remain after the capped checks, the PR is merged/closed, or a blocker requires user input.

## Completion Report

When the workflow completes, report:

- PR monitored, and issue monitored when provided or discovered
- branch name
- commit or commits created, if any
- reviewers requested, separating verified reviewer requests from attempts that could not be verified
- validation commands and outcomes, if code changed
- monitoring result, including whether it ended cleanly, addressed review feedback, replied to addressed comments, resolved review threads, recreated the heartbeat for a new cycle, or stopped on a blocker
