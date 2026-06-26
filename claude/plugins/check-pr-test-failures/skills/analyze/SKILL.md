---
name: analyze
description: "Check a GitHub PR for test failures in the checks section, then drill into GitHub Action logs and artifacts for failure details"
whenToUse: "When the user wants to investigate test failures on a GitHub pull request, check PR CI status, find why tests failed, or analyze GitHub Action run logs and artifacts"
---

# Check PR Test Failures

Investigate test failures on a GitHub pull request. Output must be short and concise.

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

Collect all checks where STATUS is `fail`.

## Step 3: Investigate each failed check

For each failed check that links to a GitHub Actions run:

1. Extract the run ID from the URL (format: `https://github.com/OWNER/REPO/actions/runs/RUN_ID/...`).
2. Fetch failed step logs:
   ```bash
   gh run view RUN_ID --repo OWNER/REPO --log-failed 2>&1 | head -500
   ```
3. Search logs for the root cause. Focus on lines containing: `FAIL`, `ERROR`, `FAILED`, `AssertionError`, `Exception`, `error:`, `BUILD FAILURE`, `exit code`.
4. If logs are insufficient, check for test artifacts:
   ```bash
   gh api repos/OWNER/REPO/actions/runs/RUN_ID/artifacts --jq '.artifacts[] | {name, size_in_bytes, expired}'
   ```
   Download and inspect if relevant artifacts exist:
   ```bash
   gh run download RUN_ID --repo OWNER/REPO --dir /tmp/pr-artifacts-RUN_ID
   ```
   Clean up after: `rm -rf /tmp/pr-artifacts-*`

## Step 4: Output the report

Output MUST be short and concise. Use exactly this format:

```
**PR #PR_NUMBER — OWNER/REPO** — PR_TITLE

| Check | Status |
|-------|--------|
| check-name-1 | passed |
| check-name-2 | passed |
| check-name-3 | FAILED |
| ... | ... |

### Failed: CHECK_NAME
- **Why:** One-sentence root cause extracted from logs
- **Log:** [link to the failed run URL]
- **Rerun:** `gh run rerun RUN_ID --repo OWNER/REPO --failed`

(repeat for each failed check)
```

Rules:
- List ALL checks in the table (passed and failed).
- For passed checks, just show "passed" — no extra details.
- For failed checks, add a section below the table with exactly three lines: Why, Log, Rerun.
- The "Why" line must be one concise sentence explaining the failure, not raw log output.
- If no checks failed, just show the table and say "All checks passed."
- Do NOT add analysis, suggestions, or commentary beyond the format above.

## Important notes

- Always use `gh` CLI — it handles authentication automatically.
- For large log output, use `head` or `tail` to avoid overwhelming the context.
- If a check is from a non-GitHub-Actions CI system (e.g. Jenkins, Prow), note that and show whatever info is available from the check URL.
