# EUDCH org-shared GitHub Actions

This is the EUDCH `.github` repository. It holds two unrelated things:

- **`profile/README.md`** is the public **organization profile** shown at
  <https://github.com/EUDCH>. It is not about CI.
- **`.github/workflows/`** holds **reusable workflows** that other EUDCH repos
  call to share CI logic. This README documents those.

## Reusable workflow: `zizmor`

[`zizmor`](https://github.com/zizmorcore/zizmor) is a static analyzer for
GitHub Actions. The reusable workflow at
`.github/workflows/zizmor.yml` scans a calling repo's workflows for
supply-chain and injection risks (unpinned actions, `pull_request_target`
misuse, template injection, over-broad permissions, credential persistence,
and similar).

By default it runs in **report-only** mode: findings are uploaded to the
calling repo's **Security tab** (code scanning / SARIF) and CI stays green.
Once a repo's findings are triaged, the call can be flipped to blocking.

### How a repo opts in

Add `.github/workflows/zizmor.yml` to the repo:

```yaml
name: zizmor
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
permissions:
  contents: read
  security-events: write
jobs:
  zizmor:
    uses: EUDCH/.github/.github/workflows/zizmor.yml@<commit-sha>
```

Pin `@<commit-sha>` to a commit of this repo (not a branch), the same way you
SHA-pin any third-party action, so an update here cannot silently change a
caller's CI until it re-pins.

### Inputs

| input | default | meaning |
|-------|---------|---------|
| `persona` | `regular` | Audit strictness: `regular`, `pedantic`, or `auditor`. |
| `advanced-security` | `true` | Upload SARIF to the Security tab (needs code scanning). Set `false` on a private repo without GitHub Advanced Security. |

### Behaviour notes

- **Fork PRs are skipped automatically.** A `pull_request` from a fork gets a
  read-only `GITHUB_TOKEN`, so the SARIF upload would 403 and fail. The
  reusable workflow guards against this centrally; a fork's workflow changes
  are scanned on push to the default branch after merge.
- **Findings are advisory during calibration.** They land in the Security
  tab; the job stays green. Flip to blocking per repo once the baseline is
  clean.
