---
name: check-pr-test-failures
description: "Check a GitHub PR for test failures in the checks section, then drill into GitHub Action logs and artifacts for failure details"
whenToUse: "When the user wants to investigate test failures on a GitHub pull request, check PR CI status, find why tests failed, or analyze GitHub Action run logs and artifacts"
---

# Check PR Test Failures

Investigate test failures on a GitHub pull request by inspecting the PR checks, drilling into failed GitHub Action runs, and extracting failure details from logs and artifacts.

## Input

The user provides one of:
- A GitHub PR URL (e.g. `https://github.com/owner/repo/pull/123`)
- A PR number + repository (e.g. `PR #123 in owner/repo`)
- Just a PR number if already inside a cloned repo

## Step 1: Parse the PR reference

Extract `OWNER`, `REPO`, and `PR_NUMBER` from the user input.

- If given a URL like `https://github.com/OWNER/REPO/pull/PR_NUMBER`, parse directly.
- If given just a number, detect the repo from the current git remote:
  ```bash
  gh repo view --json nameWithOwner -q '.nameWithOwner'
  ```

## Step 2: List PR checks and identify failures

Run:
```bash
gh pr checks PR_NUMBER --repo OWNER/REPO
```

Parse the output. Each line has: `NAME\tSTATUS\tDURATION\tURL`

- Collect all checks where STATUS is `fail`.
- If no checks failed, report that all checks passed and stop.
- Show the user a summary table of failed checks.

## Step 3: Get failed GitHub Action run details

For each failed check that links to a GitHub Actions run, extract the run ID from the URL.

The URL typically looks like: `https://github.com/OWNER/REPO/actions/runs/RUN_ID/...`

For each failed run, get details:
```bash
gh run view RUN_ID --repo OWNER/REPO
```

This shows the jobs and their statuses. Identify which jobs failed.

## Step 4: Get failed job logs

For each failed run, fetch the logs of only the failed steps:
```bash
gh run view RUN_ID --repo OWNER/REPO --log-failed 2>&1 | head -500
```

Analyze the log output to find:
- Test failure messages and stack traces
- Assertion errors
- Build errors
- Timeout messages
- Any error patterns

If the log is very long, focus on lines containing: `FAIL`, `ERROR`, `FAILED`, `AssertionError`, `Exception`, `error:`, `BUILD FAILURE`, `exit code`.

## Step 5: Check for test artifacts

List artifacts from the failed run:
```bash
gh api repos/OWNER/REPO/actions/runs/RUN_ID/artifacts --jq '.artifacts[] | {name, size_in_bytes, expired}'
```

If test-related artifacts exist (e.g. test reports, surefire reports, JUnit XML, screenshots, logs), download them to a temp directory and inspect:
```bash
gh run download RUN_ID --repo OWNER/REPO --dir /tmp/pr-artifacts-RUN_ID
```

Look for:
- `**/TEST-*.xml` or `**/surefire-reports/*.xml` — JUnit XML test reports
- `**/*.log` — log files
- `**/screenshots/` — failure screenshots
- Any file with `test`, `report`, or `result` in the name

Parse JUnit XML files to extract failed test names, failure messages, and stack traces.

## Step 6: Report findings

Present a structured report to the user:

### Summary
- PR: #PR_NUMBER in OWNER/REPO
- Total checks: N, Failed: M, Passed: P

### Failed Checks
For each failed check:
- **Check name**: NAME
- **GitHub Action Run**: link to the run
- **Failed jobs**: list of failed job names
- **Root cause**: extracted from logs
- **Error details**: relevant log excerpts (keep concise — show the most relevant 20-30 lines)
- **Artifacts**: list any downloaded artifacts with key findings

### Analysis
Provide a brief analysis:
- Are the failures related to the PR changes or flaky tests?
- Are there patterns across failures?
- Suggest next steps (re-run, fix specific tests, etc.)

## Important notes

- Always use `gh` CLI — it handles authentication automatically.
- For large log output, use `head` or `tail` to avoid overwhelming the context.
- If a check is from a non-GitHub-Actions CI system (e.g. Jenkins, Prow), note that and show whatever info is available from the check URL.
- Clean up downloaded artifacts after analysis: `rm -rf /tmp/pr-artifacts-*`
