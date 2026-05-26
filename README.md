# Railo

**Auto-fix security vulnerabilities in your pull requests.**

Railo scans every PR for known vulnerability patterns and opens a companion Fix PR with verified, deterministic patches. No AI, no LLM-generated code.

## Quick Start

1. **Get your free license key** at [railo.dev](https://railo.dev)
2. **Add it as a repository secret** named `RAILO_LICENSE_KEY`
3. **Create** `.github/workflows/railo.yml`:

```yaml
name: Railo
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write
  pull-requests: write
  statuses: write

jobs:
  railo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
          fetch-depth: 0
      - name: Railo
        uses: IWEBai/railo-action@v2
        with:
          license_key: ${{ secrets.RAILO_LICENSE_KEY }}
          mode: warn
          base_branch: ${{ github.base_ref }}
```

Open a PR and Railo scans automatically.

## Modes

| Mode | Behavior |
|------|----------|
| `warn` | Posts proposed fixes as PR comments. Your branch is unchanged. |
| `enforce` | Applies fixes and commits them to your branch automatically. |

Start with `warn` to see what Railo finds before enabling auto-fix.

## What it fixes

**17 vulnerability types across 6 languages:**

- **SQL Injection** (CWE-89) — f-strings, concatenation, .format(), ORM raw queries
- **Cross-Site Scripting** (CWE-79) — template filters, innerHTML, dangerouslySetInnerHTML
- **Hardcoded Secrets** (CWE-798) — passwords, API keys, tokens, database URIs
- **Command Injection** (CWE-78) — os.system, subprocess shell=True
- **Path Traversal** (CWE-22) — unsanitized file path joins
- **SSRF** (CWE-918) — unvalidated URL requests
- **Weak Cryptography** (CWE-327) — MD5/SHA1 for passwords
- **And more:** SSTI, XXE, open redirect, CSRF, unsafe deserialization, cookie security, clickjacking

**Supported languages:** Python, JavaScript, TypeScript, Go, Java, Ruby, C#

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `license_key` | Yes | — | Your Railo license key. Get a free key at [railo.dev](https://railo.dev). |
| `mode` | No | `warn` | `warn` or `enforce` |
| `base_branch` | No | `main` | Branch to diff against |
| `max_diff_lines` | No | `500` | Safety limit on fix size |
| `test_before_commit` | No | `false` | Run tests before committing (enforce mode) |
| `test_command` | No | `pytest` | Test command when `test_before_commit` is true |

## Configuration

Add a `.railo.yml` to your repo root for advanced configuration:

```yaml
preset: balanced  # starter, balanced, or strict
exclude:
  - "vendor/"
  - "*.generated.go"
max_diff_lines: 300
test_before_commit: true
test_command: "npm test"
```

## How it works

1. Developer opens a pull request
2. Railo scans the diff for vulnerability patterns (powered by Semgrep)
3. Generates deterministic fixes — same vulnerability, same fix, every time
4. Each fix is verified using formal methods (Z3 SMT solver)
5. Posts results as PR comments (warn) or commits fixes (enforce)

All processing runs on GitHub-hosted runners. Your code never leaves your environment.

## Links

- [Documentation](https://railo.dev/docs)
- [Privacy Policy](https://railo.dev/privacy)
- [Terms of Service](https://railo.dev/terms)
- [Support](mailto:support@railo.dev)

## License

This action is provided by IWEB. See [Terms of Service](https://railo.dev/terms) for usage terms.
