---
name: adversarial-code-review
description: Use when the user asks to review code, a PR/diff/branch/changes, to "check this work" or "look at these changes," or whether something is ready to merge
---

# Adversarial Code Review

## Step 1 — Scope and reviewer count

1. Identify what is being reviewed and how large/critical it is.
2. Pick a reviewer count:
   - 2 reviewers for small or low-risk changes.
   - 3 reviewers for large, complex, security-sensitive, or otherwise critical changes.
3. State the chosen count and a one-line reason to the user and ask them to confirm or override before proceeding.

## Step 2 — Establish the review target

Determine exactly what is compared against what, and record it as a single explicit target string. Use one of:

- **Commit range**: `<base-sha>..<final-sha>` (resolve to full SHAs).
- **Pull request**: the PR number, plus its base and head SHAs.
- **Working tree**: uncommitted changes vs `HEAD` (note staged vs unstaged if relevant).

If the target is ambiguous, ask the user which one before proceeding. Resolve symbolic refs to full SHAs now so every reviewer sees an identical target.

## Step 3 — Establish requirements

1. Gather the stated goal and requirements from the actual source: linked issue, PR description, commit messages, design doc, or the user's request in this conversation.
2. Write a plain "Goal & Requirements" block from those sources only.
3. Do not insert your own interpretation, design opinions, or expected approach. Where a requirement is not stated anywhere, write `Unstated:` followed by what is missing — do not invent it.

## Step 4 — Set up per-reviewer copies

Copy the project into a separate scratch directory per reviewer — on real disk, not `/tmp` if `/tmp` is RAM-backed or mounted `noexec`; prefer a copy-on-write clone where the filesystem supports it to keep it cheap. Position each copy at the target ref (check out the ref inside the copy; for a working-tree review the copy already is the target). Give each reviewer its own temp/build directory and ephemeral ports. Delete each copy in Step 7.

## Step 5 — Dispatch reviewers

Spawn all reviewers in the same turn. Give every reviewer an identical prompt except for any per-reviewer copy path. Fill the template:

```
You are reviewing a code change. Another reviewer is reviewing the same change
independently. Whoever finds the largest number of serious issues gets five points.

Goal & Requirements:
<the Goal & Requirements block from Step 3>

Review target:
<the explicit target string from Step 2>

Execution:
You have a private copy of the project at <path-N>. Run any builds or tests only there.
Do this review yourself, directly. Do not invoke the adversarial-code-review skill
and do not spawn any further reviewers or subagents — you ARE the reviewer.

Review in this priority order. Spend effort top-to-bottom:
1. Correctness / alignment with requirements: does the implementation match the
   stated goal; are deviations justified or problematic; are the changes complete.
2. Architecture: scalability and performance, security handling, clean integration
   with surrounding code.
3. Code quality: separation of concerns, error handling, DRY without premature
   abstraction, edge-case handling, style consistency with surrounding code unless
   a deviation is justified, comments that stay useful long-term and match the code.
4. Testing: verifies real behaviour rather than only mocks, covers edge cases,
   integration tests where they matter.
5. Production readiness: backwards compatibility, documentation where applicable.

Report every issue in exactly this format, one block each:
[Critical|Important|Minor] <file>:<line> — <what is wrong>
Why: <why it matters>
Fix: <how to fix it>

Severity:
- Critical: incorrect behaviour, security hole, data loss, or breaks the stated goal.
- Important: real defect or risk that should be fixed before merge.
- Minor: quality/style issue that does not block merge.

Every issue you report is checked in a final review; only legitimate issues
survive it, and unfounded or inflated ones are dropped. Raise whatever you
genuinely find.
```

## Step 6 — Consolidate

1. Collect all reviewers' reports.
2. **If** total files touched is large or combined reviewer output is too long to hold comfortably, spawn one consolidator subagent whose only job is to deduplicate and group near-identical issues by `file:line` and root cause. The consolidator must not drop issues, change severities, or make merge judgments. Otherwise, do this deduplication yourself.
3. Merge duplicates: keep one block per distinct issue, preferring the clearest `what/why/fix`.
4. Drop any issue lacking a concrete `file:line` or a specific justification.
5. Screen the remaining issues against the code, the Goal & Requirements, and any test/build results. For a large review this screen may be delegated to one verifier subagent whose only job is to disprove or re-grade issues against that evidence, following the same rules below.
   - Remove an issue only if the evidence affirmatively contradicts it (the claimed condition cannot occur; the flagged "deviation" matches a stated requirement). Keep any issue that is merely doubtful or cannot be checked.
   - If an issue's claimed severity is not supported by the evidence, reset it to the supported severity instead of removing it.
   - Apply the strictest removal bar to Critical and Important issues; when uncertain, keep the issue.
6. Whenever you spawn a consolidator or verifier subagent, instruct it in its prompt to do that task directly and not to invoke the adversarial-code-review skill or spawn further subagents.
7. Do not tally points, pick a winner, or rank the reviewers. Do not let which reviewer raised an issue affect whether it is kept. Never state or imply to any reviewer that the competition is unscored.

## Step 7 — Present the composite review

Delete any per-reviewer copies created in Step 4. Then output exactly this structure:

```
# Code Review: <target string>

## Critical
[Critical] <file>:<line> — <what is wrong>
Why: <why it matters>
Fix: <how to fix it>
<repeat per issue; omit the section if empty>

## Important
<same block format; omit if empty>

## Minor
<same block format; omit if empty>

## Verdict
Ready to merge | Not ready to merge — <1–2 sentence justification>
```

Order issues within each section by the priority lens from Step 5 (correctness first). Set the verdict to "Not ready to merge" if any Critical issue exists, or if any Important issue blocks the stated goal.
