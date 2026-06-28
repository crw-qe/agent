---
name: action
description: "Analyze GitHub Actions check failures on a PR. Lists all checks, classifies them, fetches GHA logs, extracts errors, and produces a failure report. For Prow/OpenShift CI failures, use /che-pr-check:openshift-ci instead."
whenToUse: "When the user wants to investigate GitHub Actions test failures on a pull request, check PR CI status, find why GHA tests failed, or analyze GitHub Action run logs and artifacts in Eclipse Che repos"
---

# Analyze GitHub Actions Failures

Investigate GitHub Actions check failures on a GitHub pull request. Produce a concise, actionable report.

For Prow/OpenShift CI failures, tell the user to use `/che-pr-check:openshift-ci` instead.

**This skill is strictly READ-ONLY.** See the constraints section at the end.

## Input

The user provides one of:
- A GitHub PR URL (e.g. `https://github.com/eclipse-che/che-operator/pull/123`)
- A PR number + repository (e.g. `PR #123 in eclipse-che/che-operator`)
- Just a PR number if already inside a cloned repo

## Step 1: Parse the PR reference

Extract `OWNER`, `REPO`, and `PR_NUMBER` from the user input.

- If given a URL like `https://github.com/OWNER/REPO/pull/PR_NUMBER`, parse directly.
- If given just a number, detect the repo from the current git remote:
  ```bash
  gh repo view --json nameWithOwner -q '.nameWithOwner'
  ```

Also fetch the PR title:
```bash
gh pr view PR_NUMBER --repo OWNER/REPO --json title -q '.title'
```

## Step 2: List all checks

```bash
gh pr checks PR_NUMBER --repo OWNER/REPO
```

Parse the output. Each line has: `NAME\tSTATUS\tDURATION\tURL`

## Step 3: Classify each check

Assign a category by matching check name (case-insensitive, first match wins):

| Category | Patterns |
|----------|----------|
| E2E Tests | `*-happy-path*`, `*-e2e-*`, `*-smoke-test*`, `smoke-test` |
| Integration Tests | `*-on-minikube*`, `chectl-deploy-test*`, `chectl-update-test*`, `release-test*` |
| Unit Tests | `unit-test*`, `build-and-test*`, `pr-check`, `code-coverage*`, `coverage-report*` |
| Static Analysis | `source-code-validation*`, `Analyze*`, `CodeQL*` |
| Build | `docker-build*`, `image-build*`, `multiplatform-image-build*`, `build`, `assemble*`, `check-artifacts*`, `build-che-*` |
| Validation | `*-validation`, `dash-licenses*`, `check-*-licenses*`, `*-check` |
| External | `eclipsefdn/eca`, `WIP`, `DCO`, `CodeRabbit*`, `Snyk*`, `Scorecard*`, `workspaces.openshift.com*`, `add-link*`, `add-web-ide-link*`, `web-ide-add-link*`, `surge*`, `time-check*`, `pull-request-info*` |

Unmatched checks → **Other**.

## Step 4: Detect CI system per check

- URL contains `github.com/.../actions/runs/` → **GHA**
- URL contains `prow.ci.openshift.org` or name starts with `ci/prow/` → **Prow**
- Everything else → **External**

## Step 5: Investigate GitHub Actions failures

For each failed check where CI = **GHA**:

1. Extract the run ID from the URL (format: `https://github.com/OWNER/REPO/actions/runs/RUN_ID/...`).

2. Fetch failed step logs:
   ```bash
   gh run view RUN_ID --repo OWNER/REPO --log-failed 2>&1 | head -1000
   ```

3. Analyze logs thoroughly. Look for:
   - **Go tests:** `--- FAIL: TestName` — extract test name and error message.
   - **Java / Maven:** `Tests run: N, Failures: N, Errors: N` and `[ERROR]` lines with class names.
   - **Node.js:** `FAIL` lines with test file paths, assertion errors.
   - **Generic:** `FAIL`, `FAILED`, `ERROR`, `AssertionError`, `Exception`, `error:`, `BUILD FAILURE`, `Bad formatted`, `mismatch`, `not well formatted`.
   - **Exit code:** Last meaningful error lines before `Process completed with exit code N`.

   Extract: (1) specific failing test/check names, (2) error messages, (3) file or module involved.

4. Check for test artifacts (read-only):
   ```bash
   gh api repos/OWNER/REPO/actions/runs/RUN_ID/artifacts --jq '.artifacts[] | {name, size_in_bytes, expired}'
   ```

   If test-related artifacts exist (names containing `junit`, `test-report`, `test-results`, `surefire`), download, parse JUnit XML, then clean up:
   ```bash
   gh run download RUN_ID --repo OWNER/REPO --dir /tmp/pr-artifacts-RUN_ID
   # ... parse XML ...
   rm -rf /tmp/pr-artifacts-*
   ```

5. Produce a 2-3 sentence explanation covering:
   - **WHAT** failed — the specific test name, check step, or build phase.
   - **WHY** it failed — the error message or root cause.
   - **WHERE** — the file, module, or test class involved.

## Step 6: Output the report

Use exactly this format:

```
**PR #PR_NUMBER -- OWNER/REPO** -- PR_TITLE

| Check | Category | Status | Duration | CI |
|-------|----------|--------|----------|----|
| unit-tests | Unit Tests | PASSED | 2m11s | GHA |
| source-code-validation | Static Analysis | FAILED | 1m54s | GHA |
| ci/prow/v19-happy-path | E2E Tests | FAILED | 15m42s | Prow |
| eclipsefdn/eca | External | PASSED | — | — |
| ... | ... | ... | ... | ... |

---

### FAILED check-name

EXPLANATION -- 2-3 sentences.

Log: [check-name](URL_TO_FAILED_CHECK)
Rerun: [Re-run failed jobs](https://github.com/OWNER/REPO/actions/runs/RUN_ID)

---

(repeat for each failed GHA check)
```

### Output rules

- The table lists ALL checks (passed, failed, pending) with category, duration, and CI system.
- Below the table, add one section per failed **GHA** check with: explanation, log link, and rerun link.
- If there are failed **Prow** checks, add a note at the end: "N Prow check(s) also failed. Use `/che-pr-check:openshift-ci PR_URL` to investigate."
- If no GHA checks failed, show only the table. If there are Prow failures, mention the prow skill. Otherwise say "All checks passed."
- Skip External checks in the investigation (show in table only).

## READ-ONLY constraints

**This skill MUST NEVER perform any write or state-modifying operation.**

- NEVER run `gh run rerun`, `gh api` with POST/PUT/PATCH/DELETE, `gh pr comment`, `git commit`, `git push`, `gh pr merge`, `gh pr close`
- Only use: `gh pr checks`, `gh pr view`, `gh run view`, `gh api GET`, `gh run download` (+ cleanup)
