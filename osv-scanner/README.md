# OSV-Scanner

This action runs [OSV-Scanner](https://google.github.io/osv-scanner/) against the checked-out repository. It replaces the per-repository OSV-Scanner workflows with a single implementation that can be shared across repositories.

The scan runs once and can either:

- Produce markdown results, write them to the job summary and optionally post/update a sticky PR comment.
- Produce SARIF results and upload them to GitHub code scanning.

A single recursive scan covers every supported ecosystem in the repository (JavaScript, Python, Go, etc.), so multi-language repositories do not need one run per language.

## Requirements

The action expects the repository to be checked out already. Add `actions/checkout` before this action.

A composite action cannot grant permissions. The calling workflow must grant the permissions needed for the features it uses:

- `contents: read` is always needed to read the checked-out repository.
- `pull-requests: write` is needed when `comment: true`.
- `actions: read` and `security-events: write` are needed when `upload-sarif: true`.

## Inputs

- `scan-args`: Arguments passed to `osv-scanner scan source`, including the path to scan. Defaults to `--recursive --no-resolve ./`. The `--format` and `--verbosity` flags are set by the action.
- `osv-scanner-version`: OSV-Scanner release version to download. Defaults to `v2.5.0`.
- `comment`: Post or update a sticky PR comment with the markdown results. Defaults to `false`. Cannot be used together with `upload-sarif`.
- `comment-header`: Hidden identifier used to find and update the existing sticky comment. Defaults to `osv-scanner`.
- `upload-sarif`: Produce SARIF and upload it to GitHub code scanning. Defaults to `false`. Cannot be used together with `comment`.
- `fail-on-vuln`: Fail the step when vulnerabilities or scan errors are found. Defaults to `true`.

## Outputs

- `format`: Output format used by the scan, either `markdown` or `sarif`.
- `results-file`: Absolute path to the generated markdown or SARIF results file.

## Usage

### Pull request scan

```yaml
name: OSV-Scanner PR Scan

on:
  pull_request:
    branches: [main]
  merge_group:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ensuro/github-actions/osv-scanner@main
        with:
          comment: true
```

This preserves the previous behavior: a full scan on every PR, a sticky comment with the results, and a failing check when vulnerabilities are found.

### Scheduled or push scan

```yaml
name: OSV-Scanner Scheduled Scan

on:
  schedule:
    - cron: "30 12 * * 1"
  push:
    branches: [main]

permissions:
  contents: read
  actions: read

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ensuro/github-actions/osv-scanner@main
```

By default this scans and fails when vulnerabilities are found, without uploading anything. To publish results to the Security > Code scanning tab, set `upload-sarif: true` and grant the extra permission:

```yaml
permissions:
  contents: read
  actions: read
  security-events: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ensuro/github-actions/osv-scanner@main
        with:
          upload-sarif: true
```

### Plain scan

With the default inputs the action performs the scan and fails when vulnerabilities are found. Markdown results are also written to the job summary:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ensuro/github-actions/osv-scanner@main
```

## Notes

- `comment` and `upload-sarif` are mutually exclusive because a single scan emits a single format. If a repository ever needs both in the same job, run the action in two steps or extend the action to emit both formats from one scan.
- The sticky comment is only posted on `pull_request` events. On `merge_group` the scan still runs and the results appear in the job summary.
- When replacing an existing OSV-Scanner workflow that used the sticky comment action without a header, set `comment-header: ''` so this action updates the existing comment instead of creating a new one.
- Pull requests from forks have a read-only `GITHUB_TOKEN`, so the sticky comment cannot be posted there.
- The default `--no-resolve` assumes dependency files are fully pinned (for example, compiled `requirements.txt` files). Override `scan-args` if a repository needs transitive resolution.
- Pin the action to a tag or commit instead of `@main` when possible.
