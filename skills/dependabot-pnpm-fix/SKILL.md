---
name: dependabot-pnpm-fix
description: 'Fix Dependabot vulnerability alerts in pnpm-based Node.js projects. Use when the user asks to fix dependabot findings, resolve npm vulnerabilities, update vulnerable transitive dependencies, or apply pnpm overrides for security patches.'
argument-hint: 'Optionally specify which function/package to focus on'
---

# Fix Dependabot Alerts (pnpm)

## When to Use

- Dependabot has opened vulnerability alerts on the repository
- User wants to batch-fix all open npm security findings
- Transitive dependency vulnerabilities need pnpm overrides
- User mentions "dependabot", "vulnerability", "security alerts", or "CVE"

## Procedure

### 1. Fetch Open Alerts

Use the GitHub CLI API endpoint with `--jq` for clean filtering. Always pipe through `| cat` or set `GH_PAGER=""` to prevent the pager from blocking non-interactive execution:

```bash
GH_PAGER="" gh api repos/{owner}/{repo}/dependabot/alerts --jq '
  .[] | select(.state == "open") | {
    number,
    package: .security_vulnerability.package.name,
    severity: .security_advisory.severity,
    fixed_in: .security_vulnerability.first_patched_version.identifier,
    manifest: .dependency.manifest_path,
    summary: .security_advisory.summary
  }'
```

> **Note:** There is no `gh dependabot` subcommand. Always use `gh api` with the REST endpoint. The `gh api` command uses a pager by default for large output — always disable it with `GH_PAGER=""` or by piping through `| cat`.

### 2. Deduplicate and Summarize

Group alerts by package name and manifest. Identify:
- Which packages are affected
- The minimum patched version required
- Which functions/workspaces are impacted

### 3. Verify Patched Versions Exist on npm

**Always ground version numbers** — never assume a version exists. Check each package:

```bash
npm view <package-name> versions --json | jq '[.[] | select(startswith("<major>."))][-5:]'
```

Or simply check the latest in the relevant major line:

```bash
npm view <package-name>@">=<patched-version>" version
```

### 4. Determine Fix Strategy

Check if the vulnerable package is a direct or transitive dependency:

```bash
pnpm why <package-name>
```

| Situation | Action |
|-----------|--------|
| Direct dependency with update available | Bump version in `dependencies`/`devDependencies` |
| Transitive dependency, parent not yet updated | Add `pnpm.overrides` in `package.json` |
| Override already exists but outdated | Update the existing override version |

### 5. Apply pnpm Overrides

Add or update the `pnpm.overrides` field in the relevant `package.json`:

```json
{
  "pnpm": {
    "overrides": {
      "vulnerable-package": "<patched-version>"
    }
  }
}
```

**Conventions:**
- Keep overrides alphabetically sorted
- Use exact versions (not ranges) for security overrides
- Apply the minimum patched version unless there's reason to use a newer one

### 6. Refresh Lockfiles

```bash
pnpm install
```

### 7. Verify Resolved Versions

Confirm the override took effect:

```bash
pnpm list <package-name> --depth=Infinity
```

### 8. Validate the Project

Run the validation suite for each affected workspace. Only run scripts that exist in the project's `package.json`:

```bash
pnpm run lint   # if "lint" script exists
pnpm run build  # if "build" script exists
pnpm run test   # if "test" script exists
```

All existing scripts must pass before considering the fix complete.

### 9. Output Summary Table

After all fixes are applied and validated, present a summary table:

```
| Package | Function/Workspace | Previous Version | Fixed Version | Fix Method |
|---|---|---|---|---|
| protobufjs | gcp-asset-collector | 7.5.5 | 7.5.8 | pnpm override |
| fast-uri | gcp-cdc-exporter | 3.1.0 | 3.1.2 | pnpm override |
```

Include: package name, affected workspace, old resolved version, new resolved version, and fix method (direct bump vs pnpm override).

## Monorepo Variant (Single Root Lockfile)

If the project uses a **pnpm workspace** with a single root `pnpm-lock.yaml`:

- Place `pnpm.overrides` in the **root** `package.json` (overrides are always root-level in pnpm workspaces)
- Use `pnpm why <package> --recursive` to trace across all workspace packages
- Use `pnpm list <package> --recursive --depth=Infinity` to verify resolution
- Run `pnpm install` once at the root
- Run validation per workspace: `pnpm --filter <workspace> run test`

## Decision Points

- **Multiple functions share the same vulnerability:** Apply the override to each function's `package.json` independently (they have separate lockfiles).
- **Override might break a parent package:** Use `pnpm why` to check the dependency chain. If the parent explicitly requires an older version range, check if the patched version still satisfies the range (it usually does for patch/minor bumps).
- **No patched version available:** Document the finding and skip — do not suppress or ignore the alert silently.

## Completion Criteria

- [ ] All open Dependabot alerts have corresponding fixes
- [ ] npm registry confirms all override versions are real/published
- [ ] `pnpm list` shows patched versions resolved
- [ ] `pnpm install` succeeds without errors
- [ ] All available validation scripts (lint/build/test) pass
- [ ] Summary table of changes is presented to the user
