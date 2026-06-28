---
name: openshift-ci
description: "Analyze Prow/OpenShift CI check failures on a PR. Fetches Prow build logs, extracts test failures, and produces a failure report. For GitHub Actions failures, use /che-pr-check:action instead."
whenToUse: "When the user wants to investigate Prow or OpenShift CI test failures on a pull request, analyze ci/prow check results, or understand E2E test failures from OpenShift CI in Eclipse Che repos"
---

# Analyze Prow / OpenShift CI Failures

Investigate Prow (OpenShift CI) check failures on a GitHub pull request. Produce a concise, actionable report.

For GitHub Actions failures, tell the user to use `/che-pr-check:action` instead.

**This skill is strictly READ-ONLY.** See the constraints section at the end.

## Repos that use Prow

These repos have Prow/OpenShift CI checks:
- `eclipse-che/che-operator` — Prow jobs: `v19-*` prefix
- `eclipse-che/che-server` — Prow jobs: `v19-*` prefix (28+ jobs, git provider matrix)
- `eclipse-che/che-dashboard` — Prow jobs: `v19-*` prefix
- `eclipse-che/che-plugin-registry` — Prow jobs: `v19-*` prefix
- `devfile/devworkspace-operator` — Prow jobs (various prefixes)

If the PR is in a repo not listed above and has no Prow checks, report that and stop.

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

## Step 2: List checks and find Prow failures

```bash
gh pr checks PR_NUMBER --repo OWNER/REPO
```

Parse the output. Filter to only Prow checks:
- Check name starts with `ci/prow/`
- OR check URL contains `prow.ci.openshift.org`

If no Prow checks exist on this PR, report: "No Prow/OpenShift CI checks found on this PR." and stop.

Identify which Prow checks have STATUS `fail`.

## Step 3: Investigate each failed Prow check

For each failed Prow check:

### 3a. Extract Prow URL

Get the check URL from the `gh pr checks` output. It typically points to:
`https://prow.ci.openshift.org/view/gs/test-platform-results/pr-logs/pull/...`

### 3b. Fetch Prow build logs

Attempt to fetch the Prow build page using WebFetch:

```
WebFetch on the Prow URL with prompt:
"Extract all test failure details from this Prow build log page. Look for:
- Lines containing FAIL, ERROR, FAILED
- Go test failures: '--- FAIL: TestName'
- Pod/container failures: OOMKilled, CrashLoopBackOff, ImagePullBackOff
- Timeout errors: 'context deadline exceeded', 'timed out'
- Step failures: 'STEP FAILED'
- Infrastructure errors: 'error: pods not found', 'unable to connect'
Return the specific test names, error messages, and any file/module references."
```

### 3c. Handle fetch failures

If WebFetch fails (auth required, redirect, timeout, empty response):
- Note: "Prow logs could not be fetched automatically."
- Provide the direct Prow URL for the user to inspect manually.
- Still produce the output report with the URL and rerun instruction.

### 3d. Analyze fetched logs

If WebFetch succeeded, look for these Prow-specific failure patterns:

- **Go test failures:** `--- FAIL: TestName` followed by assertion/error messages
- **Pod crashes:** `OOMKilled`, `CrashLoopBackOff`, `Error` in pod status
- **Image issues:** `ImagePullBackOff`, `ErrImagePull`, failed to pull image
- **Timeouts:** `context deadline exceeded`, `timed out waiting for`, `deadline exceeded`
- **Infrastructure:** `error: pods "..." not found`, `unable to connect to the server`
- **Step failures:** `STEP FAILED:`, `exit code N`
- **Namespace issues:** `namespace "..." not found`, `forbidden`

Extract: (1) specific failing test or step names, (2) error messages, (3) pod/container/namespace involved.

### 3e. Produce the explanation

Combine findings into a 2-3 sentence explanation covering:
1. **WHAT** failed — the specific test name, step, or pod.
2. **WHY** it failed — the error message, timeout, OOM, or root cause.
3. **WHERE** — the namespace, pod, container, or test file involved.

Distinguish between:
- **Test failures** — actual test assertions failed (likely PR-related)
- **Infrastructure failures** — cluster issues, image pull errors, timeouts (likely flaky/infra)

## Step 4: Output the report

Use exactly this format:

```
**PR #PR_NUMBER -- OWNER/REPO** -- PR_TITLE

### Prow / OpenShift CI Checks

| Check | Status | Duration |
|-------|--------|----------|
| ci/prow/v19-devworkspace-happy-path | FAILED | 15m42s |
| ci/prow/v19-upgrade-stable-to-next | PASSED | 20m13s |
| ci/prow/v19-images | PASSED | 5m30s |
| ... | ... | ... |

---

### FAILED ci/prow/v19-job-name

EXPLANATION -- 2-3 sentences describing what failed, why, and whether it looks like a test failure or infrastructure issue.

Log: [ci/prow/v19-job-name](PROW_URL)
Rerun: Post `/test v19-job-name` as a PR comment
  Or rerun all failed Prow jobs: `/retest`

---

(repeat for each failed Prow check)
```

### Output rules

- The table lists only Prow checks (not GHA or External checks).
- Below the table, add one section per failed Prow check.
- The explanation MUST be 2-3 sentences and should note whether the failure looks test-related or infrastructure-related.
- If no Prow checks failed, show the table and say "All Prow checks passed."
- If there are also failed GHA checks, add a note: "N GitHub Actions check(s) also failed. Use `/che-pr-check:action PR_URL` to investigate."
- The rerun instruction shows the `/test JOB_NAME` command (job name = check name without `ci/prow/` prefix) and the `/retest` shortcut.

## READ-ONLY constraints

**This skill MUST NEVER perform any write or state-modifying operation.**

- NEVER run `gh run rerun`, `gh api` with POST/PUT/PATCH/DELETE, `gh pr comment`, `git commit`, `git push`, `gh pr merge`, `gh pr close`
- Only use: `gh pr checks`, `gh pr view`, `WebFetch`, `curl -s` (GET only)
