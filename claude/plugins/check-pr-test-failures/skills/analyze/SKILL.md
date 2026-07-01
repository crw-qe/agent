---
name: analyze
description: "Analyze all CI check failures on a PR — GitHub Actions and Prow/OpenShift CI. Lists checks, fetches logs, extracts errors, and produces a unified failure report."
whenToUse: "When the user wants to investigate test failures on a pull request, check PR CI status, find why CI tests failed, analyze GitHub Actions or Prow/OpenShift CI run logs and artifacts in Eclipse Che repos"
---

# Analyze PR Check Failures

Investigate all CI check failures on a GitHub pull request — both GitHub Actions and Prow/OpenShift CI. Produce a concise, actionable report.

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

## Step 5: Investigate failed checks

### 5a. GitHub Actions failures

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

### 5b. Prow / OpenShift CI failures

For each failed check where CI = **Prow**:

1. Get the check URL from the `gh pr checks` output. It typically points to:
   `https://prow.ci.openshift.org/view/gs/test-platform-results/pr-logs/pull/...`

2. **Extract the GCS path** from the Prow URL. The URL format is:
   ```
   https://prow.ci.openshift.org/view/gs/{GCS_PATH}
   ```
   Strip the `https://prow.ci.openshift.org/view/gs/` prefix to get the GCS_PATH.
   Example: for URL `https://prow.ci.openshift.org/view/gs/test-platform-results/pr-logs/pull/eclipse-che_che-dashboard/1492/pull-ci-eclipse-che-che-dashboard-main-v19-happy-path/123456`,
   the GCS_PATH is `test-platform-results/pr-logs/pull/eclipse-che_che-dashboard/1492/pull-ci-eclipse-che-che-dashboard-main-v19-happy-path/123456`.

3. **Fetch the raw build log** directly from Google Cloud Storage:
   ```bash
   curl -sL "https://storage.googleapis.com/${GCS_PATH}/build-log.txt" 2>&1 | tail -300
   ```
   This bypasses the Prow JavaScript SPA and returns the actual log content.

4. **Fetch the JUnit XML** for structured test results (may not exist for all jobs):
   ```bash
   curl -sL "https://storage.googleapis.com/${GCS_PATH}/artifacts/junit_operator.xml" 2>&1 | head -500
   ```
   Parse the XML to find `<testcase>` elements with `<failure>` children. Extract:
   - Step/test name from the `name` attribute
   - Error details from the `<failure>` element content

5. **Fetch job result metadata** for overall status:
   ```bash
   curl -sL "https://storage.googleapis.com/${GCS_PATH}/finished.json" 2>&1
   ```
   The JSON contains `"result"` ("SUCCESS" or "FAILURE") and `"timestamp"`.

6. If all three fetches fail (e.g. GCS bucket is not public, network error):
   - Note: "Prow logs could not be fetched automatically."
   - Provide the direct Prow URL for the user to inspect manually.

7. Analyze the fetched content. Look for Prow-specific failure patterns in the build log:
   - **Go test failures:** `--- FAIL: TestName` followed by assertion/error messages
   - **Pod crashes:** `OOMKilled`, `CrashLoopBackOff`, `Error` in pod status
   - **Image issues:** `ImagePullBackOff`, `ErrImagePull`, failed to pull image
   - **Timeouts:** `context deadline exceeded`, `timed out waiting for`, `deadline exceeded`
   - **Infrastructure:** `error: pods "..." not found`, `unable to connect to the server`
   - **Step failures:** `step ... failed`, `exit code N`
   - **Install failures:** `level=fatal`, `Installer exit with code`
   - **Scheduling:** `Pod scheduling timeout`, `could not be scheduled`
   - **Namespace issues:** `namespace "..." not found`, `forbidden`
   - **Resource leaks:** `leaked resources`, `leaked ARNs`, `leaked DNS`

   Cross-reference with JUnit XML: test cases with `<failure>` elements give precise step names and error messages.

   Extract: (1) specific failing test or step names, (2) error messages, (3) pod/container/namespace involved.

8. Produce a 2-3 sentence explanation covering:
   - **WHAT** failed — the specific test name, step, or pod.
   - **WHY** it failed — the error message, timeout, OOM, or root cause.
   - **WHERE** — the namespace, pod, container, or test file involved.

   Distinguish between:
   - **Test failures** — actual test assertions failed (likely PR-related)
   - **Infrastructure failures** — cluster issues, image pull errors, timeouts (likely flaky/infra)

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

### FAILED check-name (GHA)

EXPLANATION -- 2-3 sentences.

Log: [check-name](URL_TO_FAILED_CHECK)
Rerun: [Re-run failed jobs](https://github.com/OWNER/REPO/actions/runs/RUN_ID)

---

### FAILED ci/prow/v19-job-name (Prow)

EXPLANATION -- 2-3 sentences describing what failed, why, and whether it looks like a test failure or infrastructure issue.

Log: [ci/prow/v19-job-name](PROW_URL)
Rerun: Post `/test v19-job-name` as a PR comment
  Or rerun all failed Prow jobs: `/retest`

---

(repeat for each failed check)
```

### Output rules

- The table lists ALL checks (passed, failed, pending) with category, duration, and CI system.
- Below the table, add one section per failed **GHA** and **Prow** check with: explanation, log link, and rerun instructions.
- For GHA failures, the rerun link points to the Actions run page.
- For Prow failures, the rerun instruction shows the `/test JOB_NAME` command (job name = check name without `ci/prow/` prefix) and the `/retest` shortcut.
- Skip External checks in the investigation (show in table only).
- If no checks failed, show only the table and say "All checks passed."

## READ-ONLY constraints

**This skill MUST NEVER perform any write or state-modifying operation.**

- NEVER run `gh run rerun`, `gh api` with POST/PUT/PATCH/DELETE, `gh pr comment`, `git commit`, `git push`, `gh pr merge`, `gh pr close`
- Only use: `gh pr checks`, `gh pr view`, `gh run view`, `gh api GET`, `gh run download` (+ cleanup), `WebFetch`, `curl -s` (GET only)
