# crw-qe-agent

Plugin marketplace for automation tools used by the Dev Spaces QE team.

## Plugins

### check-pr-test-failures

Analyze PR check failures across Eclipse Che repos. Handles both GitHub Actions and Prow/OpenShift CI in a single pass. Read-only.

**Usage:**

```
/check-pr-test-failures:analyze https://github.com/eclipse-che/che-operator/pull/123
```

The skill auto-detects the CI system for each check, fetches logs, extracts errors, and produces a unified failure report.

## Installation

1. Add this plugin to your project's `.claude/settings.json`:

   ```json
   {
     "enabledPlugins": {
       "crw-qe-agent": "/path/to/crw-qe/agent"
     }
   }
   ```

2. Add the required permissions to your project's `.claude/settings.json` (or user-level `~/.claude/settings.json`):

   ```json
   {
     "permissions": {
       "allow": [
         "Bash(gh pr checks *)",
         "Bash(gh pr view *)",
         "Bash(gh run view *)",
         "Bash(gh api repos/*/actions/runs/*/artifacts *)",
         "Bash(gh run download *)",
         "Bash(gh repo view *)",
         "Bash(rm -rf /tmp/pr-artifacts-*)",
         "Read(//tmp/**)",
         "WebFetch(domain:prow.ci.openshift.org)",
         "WebFetch(domain:gcsweb-ci.apps.ci.l2s4.p1.openshiftapps.com)"
       ]
     }
   }
   ```

   Without these permissions, the skill will still work but you'll be prompted to approve each command individually.

3. Accept the workspace trust dialog on first run (or set `hasTrustDialogAccepted: true` for the project in `~/.claude.json`).

## Prerequisites

- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
