# Railo

**Closes the security findings in your pull requests with a small, reviewed fix PR.**

Railo scans each pull request, rewrites the vulnerable code with a deterministic, known-safe pattern, and opens a separate fix PR against the PR's branch for you to review. No LLM writes the code, and everything runs in your own GitHub Actions runner.

## Quick start

Create `.github/workflows/railo.yml`:

```yaml
name: Railo
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write        # enforce mode pushes the fix PR's branch; use `read` for warn mode
  pull-requests: write   # PR comments and the fix PR
  checks: write          # the Railo check run on your PR
  security-events: write # optional: upload findings to GitHub code scanning

jobs:
  railo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
          fetch-depth: 0
      - uses: IWEBai/railo-action@v2
        with:
          mode: warn
```

Open a pull request and Railo runs. No license key is needed on the Free tier (3 repositories, 200 scans a month). Paid plans add theirs as the `RAILO_LICENSE_KEY` secret and pass `license_key: ${{ secrets.RAILO_LICENSE_KEY }}`.

## Modes

| Mode | What happens |
|------|--------------|
| `warn` | Railo posts each proposed fix as a one-click suggestion on the lines it changes (**Commit suggestion** applies it). Nothing is pushed. |
| `enforce` | Railo opens a separate fix PR against your PR's branch. Your branch is never changed until you merge it. |

Start with `warn` to see what Railo would change.

A fix PR opened with the workflow's own token does not start your other workflows (a GitHub rule). Re-run your checks on the fix PR, or push to it, before merging.

## What Railo fixes

Fix quality is measured on every release against a benchmark of reviewed vulnerable-to-fixed pairs; a fix class ships only at 90% or better.

| Python fix class | Example rewrite |
|------------------|-----------------|
| SQL injection (CWE-89) | `cur.execute(f"... {x}")` → `cur.execute("... %s", (x,))` |
| Command injection (CWE-78) | `os.system("ping -c 1 " + host)` → `subprocess.run(["ping", "-c", "1", host])` |
| Hardcoded secrets (CWE-798) | `password="..."` → `password=os.environ.get("PASSWORD")` |
| Cross-site scripting (CWE-79) | `mark_safe("<b>" + name + "</b>")` → `format_html("<b>{}</b>", name)` |
| Path traversal (CWE-22) | adds a check that the resolved path stays under the base directory, raising `PermissionError` otherwise |
| XXE (CWE-611) | `xml.etree.ElementTree` → `defusedxml.ElementTree` (the fix PR says when your project must add `defusedxml`) |
| Missing timeout (CWE-400) | `httpx.post(url, json=data)` → `httpx.post(url, json=data, timeout=10.0)` |
| Insecure randomness (CWE-330) | `random.randint(a, b)` → `secrets.SystemRandom().randint(a, b)` |
| Cookie flags (CWE-614/1004) | `set_cookie("session", v)` → `set_cookie("session", v, httponly=True, secure=True, samesite="Lax")` |

Python also has fixes for SQL injection in Django and SQLAlchemy raw queries, server-side template injection, open redirects, disabled CSRF protection, permissive CORS with credentials, disabled TLS certificate checks, unsafe YAML loading, `eval()` on data, XML entity expansion, stack traces sent to users, and debug mode left on.

Other languages, each fix class on the same benchmark:

| Language | Fix classes |
|----------|-------------|
| JavaScript and TypeScript | SQL injection, command injection, cross-site scripting, path traversal, open redirects, hardcoded secrets, missing timeouts, insecure randomness, insecure cookies |
| Java | SQL injection, cross-site scripting, path traversal, SSRF, hardcoded secrets |
| Go | SQL injection, command injection, hardcoded secrets |
| PHP | SQL injection, command injection, cross-site scripting, path traversal, unsafe `unserialize`, hardcoded secrets, insecure randomness, insecure cookies |

Findings in other languages (Ruby and C#, for example) are still detected and reported, but not rewritten.

When Railo cannot fix something safely (for example a command whose program comes from user input), it says so in the PR rather than guessing.

## How a fix is checked

Each fix PR states, per finding, what was checked:

- **Rescan.** The rules run again on the patched file: the finding must be gone and no new finding may appear.
- **Safe pattern present.** The rewritten code must contain the known-safe construct (a bound query parameter, an argument list, an escaping call).
- **Tests.** When Railo finds your test suite (pytest or a `package.json` test script), it runs it before and after each fix in your runner and drops any fix that adds a failure.

Every fix PR needs your review before merging.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `license_key` | none | Paid plans only. |
| `mode` | `warn` | `warn` or `enforce` |
| `base_branch` | the PR's base branch | Branch to diff against |
| `max_diff_lines` | `500` | Largest total diff Railo will deliver in one run |
| `test_before_commit` | `false` | Only for direct pushes (`RAILO_DELIVERY=push`): also run `test_command` first. Fix PRs always run your tests as described above. |
| `test_command` | `pytest` | Command used by `test_before_commit` |
| `sarif` | none | SARIF files from scanners you already run, as globs (e.g. `results/*.sarif`) |

## Fixing findings from your own scanners

If a step earlier in the job writes SARIF (CodeQL, Semgrep, Bandit, Snyk and most other scanners can), pass it in and Railo fixes those findings too:

```yaml
      - run: semgrep scan --sarif --output semgrep.sarif
      - uses: IWEBai/railo-action@v2
        with:
          mode: enforce
          sarif: semgrep.sarif
```

Railo checks these fixes by rerunning its own rules, not your scanner's, so the fix PR says the original rule was not rerun. Rescan with your scanner before merging.

## Configuration

Add a `.railo.yml` to your repository root for more control:

```yaml
max_diff_lines: 300
test_timeout: 300
```

To skip paths, list them in a `.railoignore` file (same syntax as `.gitignore`).

## Privacy

Scanning, fixing and testing run in your GitHub Actions runner; your source code is not sent to Railo. The Action sends usage data (organization and repository name, plan, mode, number of findings and fixes, scan duration) to railo.dev, which also enforces plan limits; set the environment variable `RAILO_TELEMETRY=0` to turn this off.

## Links

- [Documentation](https://railo.dev/docs)
- [Privacy policy](https://railo.dev/privacy)
- [Terms of service](https://railo.dev/terms)
- [Support](mailto:support@railo.dev)

## License

This action is provided by IWEB. See the [terms of service](https://railo.dev/terms).
