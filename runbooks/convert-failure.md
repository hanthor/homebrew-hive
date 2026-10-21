# Runbook: `Convert Hive contributor image` workflow fails or the formula goes stale

Use this when `convert.yml` (scheduled every 6 hours, plus manual and
`repository_dispatch` runs) fails, or when `Formula/contribute.rb` on `main`
lags the image that `ghcr.io/hivecommons/hive-contributor:stable` points to.

## Impact

No user-visible breakage: installed copies and the formula already on `main`
keep working. The impact is that new promotions of `stable` (or `candidate`)
do not reach `brew install` / `brew upgrade` until the pipeline recovers and
the generated PR is merged.

## Detect

- A failed run of `Convert Hive contributor image` in the Actions tab.
- An open `generated/contribute` (or `generated/contribute-candidate`) PR that
  has not been merged after a promotion.
- `hive-contribute --wrapper-version` reports an `image=` digest that differs
  from the current `stable` digest:
  `docker buildx imagetools inspect ghcr.io/hivecommons/hive-contributor:stable`.

## Triage by failing job

| Job / step | Typical cause | Action |
|---|---|---|
| `resolve` / Resolve channel digest | Image or channel tag missing in GHCR, registry outage, unsupported `channel` input | Confirm the tag exists upstream; if upstream is broken, raise it in `hivecommons/hive`. Otherwise re-run. |
| `build` / Install Apptainer | PPA or apt outage | Re-run the failed job. |
| `build` / Build SIF | Image pull failure or one arch missing from the manifest | Check the manifest lists both `linux/amd64` and `linux/arm64`; the `fail-fast: false` matrix means one arch can fail alone, and `publish` will not run. |
| `build` / Build SIF (size check) | SIF reached GitHub's 2 GB release asset limit | The image has grown. Raise it upstream; do not publish a truncated asset. |
| `publish` / Publish release assets | `Release <tag> already exists` | A partial earlier run created the release. Re-run with `force=true` to replace its assets, or delete the release and re-run. |
| `publish` / Commit generated formula | Push or PR creation rejected (branch protection, token permissions) | Check that the `generated/*` branch is not protected against force-push and that `contents: write` and `pull-requests: write` are still granted to the job. |

## Recover

1. Fix or wait out the cause from the table.
2. Re-run the failed jobs, or start a fresh run:
   `gh workflow run convert.yml --repo hivecommons/homebrew-hive -f channel=stable`
   Add `-f force=true` only when a release with the same tag exists and its
   assets are wrong or incomplete.
3. Review and merge the updated `generated/contribute` PR. Each run
   force-pushes the newest formula to that branch and edits the existing PR, so
   there is only one PR per channel to merge.

## Verify

```sh
brew update
brew upgrade hivecommons/hive/contribute
hive-contribute --wrapper-version   # image= digest matches the current stable digest
```

Confirm the next scheduled run is green and `Tap CI` passes on `main`.

## Follow-up

- If the failure was caused by a bad published formula rather than the
  pipeline, use [`rollback.md`](rollback.md) and fill in
  [`postmortem-template.md`](postmortem-template.md).
- If the same step fails repeatedly, open an issue describing the cause and
  fix the workflow rather than relying on re-runs.
