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
| `warn` | Railo comments the proposed fixes on your PR. Nothing is pushed. |
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

Railo also has fixers for path traversal, missing timeouts, insecure randomness, cookie flags and XXE, and detects TypeScript findings. Those fixers are not yet on the benchmark, so review their fix PRs with extra care.

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
