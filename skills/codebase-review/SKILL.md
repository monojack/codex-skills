---
name: codebase-review
description: "Use when Codex should perform a very thorough whole-codebase review: inspect architecture and implementation deeply, read relevant official framework documentation first, identify bugs plus nitpicky maintainability issues such as non-idiomatic, hacky, outdated, inconsistent, duplicated, dead, or unused code, write the review under reviews/, and open GitHub issues for every actionable finding."
---

# Codebase Review

## Overview

Use this skill for exhaustive repository reviews where the deliverables are a written review file and GitHub issues for each actionable finding. Treat the work as a code review, not a refactor: inspect and report; do not edit product code unless the user explicitly asks for fixes.

Prioritize correctness, security, data integrity, performance, accessibility, test gaps, maintainability, idiomatic framework usage, outdated patterns, duplicated logic, dead code, dead variables, inconsistent implementations, and suspicious abstractions.

## Start Conditions

Resolve the target repository from the current workspace or the user request. If the repository, GitHub remote, or issue target is ambiguous, ask one concise clarification before opening issues.

Check the working tree before reviewing. Do not revert or modify unrelated local changes. If uncommitted changes are present, mention in the review whether findings are against the current working tree, the last commit, or a user-specified ref.

## Required Documentation Pass

Before judging framework idioms, identify the relevant frameworks, runtimes, and major libraries from manifests, lockfiles, config files, imports, and README docs.

Read official or primary documentation first for each review-relevant framework or library. Prefer local project docs when the repository vendors or links them; otherwise browse official documentation or primary source docs. For OpenAI, Vercel, Next.js, React, GitHub Actions, database clients, ORMs, test frameworks, and language tooling, use official docs or primary sources only.

In the review file, include a short "Documentation Consulted" section listing the sources or local docs used. If documentation could not be accessed, say so and separate confirmed findings from framework-idiom concerns that need verification.

## Structured Review Cadence

Use a progressive capture workflow so review evidence survives long sessions and context compaction:

1. First analyze the codebase at a high level: identify the stack, architecture, key flows, risky surfaces, test strategy, and review plan.
2. Create the review file immediately after the initial analysis, with placeholders for scope, documentation consulted, commands, findings, issue links, and residual risks.
3. As soon as a finding is credible, append a concise draft finding to the review file with file paths, evidence, severity, and open questions. Do not wait until the entire codebase review is complete.
4. Continue reviewing in passes. Expand, merge, downgrade, or remove draft findings as more evidence appears.
5. Before opening GitHub issues, do one deduplication and evidence-quality pass over the accumulated findings.

Prefer an imperfect but written draft over keeping findings only in conversation context. Mark uncertain findings as "Needs verification" until confirmed.

## Review Method

1. Map the repository: manifests, package/workspace layout, application entry points, API routes, background jobs, database schema/migrations, shared libraries, tests, CI, deployment config, generated files, and documentation.
2. Build a focused file inventory. Exclude dependency folders, build artifacts, caches, vendored generated output, and large binaries unless they are directly relevant.
3. Inspect similar implementations together. Compare route handlers, components, hooks, services, jobs, schemas, migrations, tests, and config files for inconsistent behavior or drift.
4. Search for repeated code, dead exports, dead variables, TODO/FIXME/HACK notes, unreachable branches, stale feature flags, outdated APIs, skipped tests, broad catches, ignored promises, unsafe casts, and configuration mismatches.
5. Run static analysis and tests when practical. Use the repository's existing commands first, then narrow targeted commands as needed. Never invent successful validation; record commands that could not run and why.
6. Classify every potential issue. Prefer actionable findings with concrete risk over style preferences. Include nitpicks when they point to real maintenance cost, inconsistency, outdated practice, or likely future bugs.

Use subagents only when explicitly allowed by the user or by the active system instructions. If used, split the review by independent areas and reconcile duplicate findings yourself before writing issues.

## Finding Standards

Each finding must include:

- title
- severity: Critical, High, Medium, Low, or Nitpick
- category
- affected file and line when possible
- concrete evidence from the code
- why it matters
- recommended fix
- documentation basis when the finding depends on framework or library idioms

Do not open issues for vague preferences, speculative rewrites, or findings that cannot be tied to code evidence. Combine duplicate instances into one issue when they share the same root cause and fix; create separate issues when they need different owners, fixes, or risk treatment.

## Review File

Create the `reviews/` folder if needed during the initial analysis phase. Write one Markdown review file named with the current date and a short slug, for example `reviews/2026-04-25-codebase-review.md`.

The review file should contain:

- scope and reviewed ref
- documentation consulted
- commands run and results
- executive summary
- findings grouped by severity
- issue links or issue creation status for each finding
- non-actionable observations, if useful
- residual risks and areas not fully reviewed

Keep the review self-contained enough that a maintainer can understand the risks without rereading the entire conversation.

## GitHub Issues

Open one GitHub issue for every actionable finding after deduplicating. Prefer the GitHub connector for issue creation; use `gh issue create` only when connector support is unavailable or insufficient.

Before creating issues, check for repository issue templates in `.github/ISSUE_TEMPLATE/`, `.github/ISSUE_TEMPLATE.md`, organization-level templates exposed by GitHub, or issue forms available through the GitHub UI/API. Use the most appropriate template for each finding, such as bug, security, performance, documentation, or tech-debt templates. Fill all required template fields faithfully, preserving the template structure and prompts. If no suitable template exists, use the fallback issue body structure below.

Issue bodies should include:

- severity and category
- affected files and lines
- evidence
- expected or preferred behavior
- suggested fix
- validation idea
- link or reference to the review file path

Use labels when the repository supports obvious labels such as `bug`, `security`, `performance`, `accessibility`, `tech debt`, `documentation`, `tests`, or `good first issue`. Do not create new labels unless the user asks.

After issue creation, update the review file with the issue URLs. If an issue cannot be opened, record the exact blocker and keep the finding in the review.

## Completion Report

Report:

- review file path
- number of findings by severity
- GitHub issues opened and any failures
- validation commands run
- notable areas not fully covered
