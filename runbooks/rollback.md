# Runbook: roll back a bad `contribute` formula

Use this when a merged formula (`Formula/contribute.rb` or
`Formula/contribute-candidate.rb`) installs a broken `hive-contribute`, or the
image it pins is bad. `Formula/` is generated output; roll back by restoring a
previous generated version, never by hand-editing.

## Symptoms

- `brew install hivecommons/hive/contribute` or `hive-contribute --help` fails.
- `brew test hivecommons/hive/contribute` fails on a machine that passed before.
- The pinned image digest in the formula caveats is reported as defective by
  the image owner (`hivecommons/hive`).

## 1. Confirm the blast radius

1. Find the merge commit that introduced the bad formula:
   `git log --oneline -- Formula/contribute.rb`.
2. Note the pinned image digest and release tag (`contribute-<digest>`) in that
   commit and in the previous one. Users who already installed keep the old SIF;
   only new installs and `brew upgrade` pick up the bad version.
3. Check that the SIF release assets for the previous formula still exist under
   the repository releases. The previous formula is only restorable while its
   release assets are present. Do not delete old `contribute-*` releases.

## 2. Roll back the formula

1. `git revert <merge-commit>` on a branch from `main` (sign off: `git revert -s`).
2. Open a PR to `main`. CI (`Tap CI`) must pass: `brew audit --strict`,
   install from source, `hive-contribute --help`, `brew test`.
3. Merge. Verify on a clean machine:
   `brew update && brew reinstall hivecommons/hive/contribute && hive-contribute --help`.

## 3. Stop the automation re-introducing the bad digest

`convert.yml` runs every six hours and regenerates the formula from the current
`stable` (or `candidate`) image channel. It skips a digest only when a release
for that digest already exists, so while the channel still points at the bad
digest the next run will not re-publish it, but any new digest will open a new
generated PR.

- If the bad digest is still the channel tip, ask the image owner in
  `hivecommons/hive` to move the channel back to a good digest before the next
  scheduled run, and do not merge generated PRs for the bad digest.
- Close any open `generated/contribute-*` PR for the bad digest.
- If a generated PR for the reverted digest is open, close it rather than merge.

## 4. Follow up

- Open an issue describing the failure, the digest, and the time to detect.
- If the failure was not caught by `Tap CI`, add a check to
  `.github/workflows/ci.yml` or `scripts/generate-formula.py` so it is.
