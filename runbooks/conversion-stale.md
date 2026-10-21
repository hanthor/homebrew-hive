# Runbook: formula is stale or the conversion workflow is failing

Use this when `Convert Hive contributor image` (`convert.yml`) fails, or when
`Formula/contribute.rb` no longer tracks the upstream `stable` image.

## Impact

New installs and upgrades keep getting the last merged formula. Nothing breaks
for existing users, but a promoted upstream fix does not reach them until the
conversion succeeds and its generated PR is merged.

## Detect

`convert.yml` runs every six hours and on `repository_dispatch`. Failures only
show in the Actions tab; nothing pages. Check:

```sh
gh run list --repo hivecommons/homebrew-hive --workflow convert.yml --limit 5
docker buildx imagetools inspect ghcr.io/hivecommons/hive-contributor:stable
hive-contribute --wrapper-version   # image= digest installed from the tap
```

The tap is stale when the `stable` digest differs from the digest in
`Formula/contribute.rb` on `main` and no open `generated/contribute` PR carries
it. An open `generated/contribute` PR that has not merged is a review backlog,
not a conversion failure.

## Triage by failing job

| Job / step | Likely cause | Action |
| --- | --- | --- |
| `resolve` / Resolve channel digest | Channel tag missing or GHCR unreachable | Confirm the tag exists upstream with the `imagetools inspect` command above; re-run once GHCR recovers. |
| `build` / Build SIF | Apptainer PPA or runner outage, or image pull failure | Re-run the failed job. |
| `build` / size check | A SIF reached the 2 GB release asset limit | Shrink the upstream `hive-contributor` image; the tap cannot publish it as-is. |
| `publish` / Publish release assets | Release `contribute-<digest12>` already exists | Do not clobber it. If the release is good, run with `force=false` and let the formula step regenerate; if it is bad, follow [rollback.md](rollback.md). |
| `publish` / Commit generated formula | Push or PR creation rejected | Check repository Actions permissions; the workflow needs `contents: write` and `pull-requests: write`. |

Re-run by hand when the cause is transient:

```sh
gh workflow run convert.yml --repo hivecommons/homebrew-hive -f channel=stable
```

## Verify

- The latest `convert.yml` run is green.
- `gh pr view generated/contribute --repo hivecommons/homebrew-hive` shows the
  current `stable` digest; a maintainer must merge it.
- After merge, `Tap CI` on `main` is green and
  `brew update && brew upgrade hivecommons/hive/contribute` followed by
  `hive-contribute --wrapper-version` prints the expected `image=` digest.

## Follow-up

If the stale window exceeded one day or a promoted fix was delayed, record it
with [postmortem-template.md](postmortem-template.md).
