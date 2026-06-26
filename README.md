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
| `enforce` | `false` | Fail CI on any zizmor finding. Set `true` once the repo's baseline is clean; new findings then block the PR. Accept a specific finding with an inline `# zizmor: ignore[rule]` comment instead of disabling enforcement. |

### Behaviour notes

- **Fork PRs are scanned in annotation mode.** A `pull_request` from a fork
  gets a read-only `GITHUB_TOKEN`, so the SARIF upload to the Security tab is
  not possible. Rather than skip the scan — which would leave external
  contributors' workflow changes, the ones most worth checking, unscanned until
  after merge — the workflow downgrades fork PRs to **annotation mode**
  (`advanced-security: false`): findings show up as inline PR annotations
  instead of in the Security tab. The report-vs-block decision matches what the
  same change would get on a same-repo PR — advisory under `enforce: false`,
  blocking under `enforce: true` (and always blocking for a caller that sets
  `advanced-security: false` itself). zizmor is static analysis and never
  executes the scanned workflows, so running it on fork content is safe.
- **Findings are advisory during calibration** (`enforce: false`, the
  default). They land in the Security tab and the job stays green. Once a
  repo's baseline is clean, set `enforce: true` in its caller so new findings
  fail CI; accept a specific finding with an inline `# zizmor: ignore[rule]`
  comment rather than turning enforcement off.
