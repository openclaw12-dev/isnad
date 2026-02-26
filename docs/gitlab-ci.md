# GitLab CI/CD Integration

Run ISNAD Scanner automatically on every merge request and push to your default branch. Findings appear in GitLab's Security Dashboard.

## Quick Start

Copy the template into your repo:

```bash
cp templates/gitlab-ci.yml .gitlab-ci.yml
git add .gitlab-ci.yml
git commit -m "ci: add isnad-scanner security scan"
git push
```

That's it. The scanner runs on every pipeline and uploads results to GitLab's Security Dashboard.

---

## How It Works

```
push / MR ──▶ install @isnad/scanner ──▶ batch scan ──▶ JSON results
                                                            │
                                                  ┌─────────┴──────────┐
                                                  ▼                    ▼
                                          GitLab report        Pipeline pass/fail
                                       (gl-sast-report.json)
                                                  │
                                                  ▼
                                       GitLab Security Dashboard
```

1. **Install** — pulls `@isnad/scanner` from npm (cached between runs)
2. **Scan** — runs `npx @isnad/scanner batch` against your target glob pattern
3. **Convert** — transforms JSON findings to GitLab Security Report format
4. **Report** — uploads the report artifact so findings appear in the Security Dashboard
5. **Gate** — optionally fails the pipeline on critical/high findings

---

## CI/CD Variables

Configure these in **Settings → CI/CD → Variables** or directly in `.gitlab-ci.yml`.

| Variable | Default | Description |
|---|---|---|
| `ISNAD_TARGET` | `**/*.{js,ts,py,mjs,cjs}` | Glob pattern for files to scan |
| `ISNAD_OUTPUT_FORMAT` | `gitlab` | Output format: `gitlab` (Security Dashboard), `sarif`, or `json` |
| `ISNAD_FAIL_ON_RISK` | `true` | Fail pipeline on critical/high findings |
| `ISNAD_FAIL_FAST` | `false` | Stop after first critical/high finding |
| `ISNAD_VERSION` | `latest` | Scanner version to install (`latest` or semver) |
| `ISNAD_NODE_IMAGE` | `node:20-alpine` | Docker image for the job |

---

## Example Configurations

### Minimal

Scan everything with defaults:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/counterspec/isnad/main/templates/gitlab-ci.yml'
```

Or copy the template as-is — zero configuration needed.

### Custom Target

Scan only your `skills/` directory:

```yaml
variables:
  ISNAD_TARGET: "skills/**/*.{js,ts,py}"
```

### Non-Blocking (Audit Mode)

Run the scan but never fail the pipeline:

```yaml
variables:
  ISNAD_FAIL_ON_RISK: "false"
```

Findings still appear in the Security Dashboard — you just don't gate on them.

### Fail Fast

Stop immediately when a critical/high issue is found (useful for large repos):

```yaml
variables:
  ISNAD_FAIL_FAST: "true"
  ISNAD_FAIL_ON_RISK: "true"
```

### SARIF Output

Generate SARIF 2.1.0 output for external tools:

```yaml
variables:
  ISNAD_OUTPUT_FORMAT: "sarif"
```

### JSON Output Only

Skip conversion, produce raw scanner JSON:

```yaml
variables:
  ISNAD_OUTPUT_FORMAT: "json"
```

### Pinned Version

Lock the scanner version for reproducible builds:

```yaml
variables:
  ISNAD_VERSION: "0.1.0"
```

### Monorepo

Scan multiple directories with separate jobs:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/counterspec/isnad/main/templates/gitlab-ci.yml'

# Override for the backend
isnad-scan-backend:
  extends: isnad-scan
  variables:
    ISNAD_TARGET: "backend/skills/**/*.ts"

# Override for the frontend
isnad-scan-frontend:
  extends: isnad-scan
  variables:
    ISNAD_TARGET: "frontend/plugins/**/*.{js,ts}"

# Override for Python agents
isnad-scan-agents:
  extends: isnad-scan
  variables:
    ISNAD_TARGET: "agents/**/*.py"
```

### Full Configuration

Every knob turned:

```yaml
variables:
  ISNAD_TARGET: "src/**/*.{js,ts,mjs}"
  ISNAD_OUTPUT_FORMAT: "gitlab"
  ISNAD_FAIL_ON_RISK: "true"
  ISNAD_FAIL_FAST: "false"
  ISNAD_VERSION: "0.1.0"
  ISNAD_NODE_IMAGE: "node:20-alpine"

include:
  - remote: 'https://raw.githubusercontent.com/counterspec/isnad/main/templates/gitlab-ci.yml'
```

---

## Security Dashboard

When `ISNAD_OUTPUT_FORMAT` is set to `gitlab` (the default), findings are uploaded as a [SAST artifact](https://docs.gitlab.com/ee/ci/yaml/artifacts_reports.html#artifactsreportssast). This makes them available in:

- **Merge Request → Security tab** — see new findings introduced by the MR
- **Security → Vulnerability Report** — all findings across the project
- **Pipeline → Security tab** — findings for that specific run

### Severity Mapping

| ISNAD Severity | GitLab Severity |
|---|---|
| Critical | Critical |
| High | High |
| Medium | Medium |
| Low | Low |
| Clean | Info |

---

## Using as a Local Dependency

If `@isnad/scanner` is already in your `package.json`, the CI template detects it automatically and runs `npm ci` instead of a global install. This gives you version locking through your lockfile:

```bash
npm install --save-dev @isnad/scanner
```

---

## Caching

The template caches:

- **npm packages** — `$CI_PROJECT_DIR/.npm-cache/`
- **node_modules** — reused between runs on the same branch

Cache key is derived from `package-lock.json` so it invalidates when dependencies change.

---

## Troubleshooting

### "Cannot find module '@isnad/scanner'"

The install might have failed. Check:
- Network connectivity to npm registry
- `ISNAD_VERSION` is a valid published version
- Try `ISNAD_NODE_IMAGE: "node:20"` (non-alpine) if native deps are needed

### No findings in Security Dashboard

- Ensure `ISNAD_OUTPUT_FORMAT` is `gitlab` (the default)
- Check that `gl-sast-report.json` appears in the job artifacts
- GitLab Security Dashboard requires **Ultimate** tier for full features (merge request widget works on all tiers)
- Verify the report artifact is not empty

### Pipeline fails unexpectedly

If the scanner exit code is non-zero but you want an audit-only mode:
```yaml
variables:
  ISNAD_FAIL_ON_RISK: "false"
```

### Scanning takes too long

Narrow the target glob or enable fail-fast:
```yaml
variables:
  ISNAD_TARGET: "src/skills/**/*.ts"
  ISNAD_FAIL_FAST: "true"
```
